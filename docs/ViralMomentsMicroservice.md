# Viral Moments Microservice

Comprehensive reference for the RunPod-hosted Viral Moments pipeline that mirrors Laravel's content-generation stack.

## Architecture Overview
- **Input**: `ProcessRequest` (transcript sentences, words, metadata, webhook endpoints, options).
- **Chunking**: `chunking.build_chunks` splits transcript into overlapping windows mirroring `DetectViralMomentsJob` config (450s duration, 45s overlap, 180s minimum) with override support.
- **Per-chunk processing**: `chunk_processor.process_chunk` analyses structure, builds prompts, calls OpenAI, refines/validates via `tooling`, and returns `ChunkResult`.
- **Aggregation**: `processor.execute` dispatches chunks concurrently, streams progress webhooks, deduplicates results, sends completion/failure webhooks, and returns `ServiceResult`.
- **Sync vs RunPod**: `handler.handle_sync` executes inline (tests/local use); `handler.handler` + `runpod_worker.py` integrate with RunPod serverless.

## File Map & Laravel Counterparts
| Python Module / Asset | Responsibility | Laravel Origin / Parity |
|-----------------------|----------------|-------------------------|
| `src/viral_moments_service/config.py` | Pydantic settings (`OPENAI_*`, chunk defaults, concurrency). | Mirrors `DetectViralMomentsJob` ctor defaults + `config/services.php` overrides. |
| `src/viral_moments_service/models.py` | Typed request / response DTOs (`ProcessRequest`, `ChunkTranscript`, `ViralMoment`). | Maps to `Src\Domain\Video\DataTransferObjects\*` and `ProcessViralMomentChunkJob` payload schema. |
| `src/viral_moments_service/transcript_provider.py` | Sentence/word lookup helpers, word-boundary clamp. | Port of `Src\Domain\Video\PreTreatment\ToolCalls\TranscriptDataProvider`. |
| `src/viral_moments_service/tooling.py` | Structure analysis, sentence/word refinement, quote rebuild, validation constants. | Consolidates executor logic from `AnalyzeTranscriptStructureExecutor`, `FindSegmentBoundariesExecutor`, `CalculateWordBoundariesExecutor`, `GeneratePlatformCopyExecutor`, `ValidateViralMomentExecutor`. |
| `src/viral_moments_service/chunking.py` | Chunk window construction + overlap handling. | Matches `DetectViralMomentsJob::buildTimeChunks`. |
| `src/viral_moments_service/chunk_processor.py` | Per-chunk OpenAI call, JSON parsing, refine/validate pipeline. | Equivalent to `ProcessViralMomentChunkJob::handle` tool-call loop. |
| `src/viral_moments_service/dedup.py` | IoU/pruning + score richness tie-breakers. | Aligns with `StoreViralMoments` duplicate rejection and planned cross-chunk pruning. |
| `src/viral_moments_service/processor.py` | Orchestrates chunk futures, aggregates results, emits webhooks. | Covers `DetectViralMomentsJob` batching + `ContentTaskService::storeViralMoments`. |
| `src/viral_moments_service/webhook.py` | Signed webhook dispatcher with retries/backoff. | Mirrors Laravel's `WebhookDispatcher` middleware + `WebhookClient`. |
| `src/viral_moments_service/prompts.py` | System/user prompts enforcing 40–90 s, hook requirements, platform copy instructions. | Synced with Blade prompt fragments in `Src/Domain/Video/Prompts`. |
| `src/viral_moments_service/handler.py` | Sync + RunPod job handlers (progress/result payloads). | Equivalent to `DetectViralMomentsJob::handle` + RunPod callback binding. |
| `src/viral_moments_service/runpod_worker.py` | Serverless bootstrap (`--serve` guard, stub fallback for local tests). | Wraps RunPod entrypoint used by Laravel's job dispatcher. |
| `src/viral_moments_service/logging_config.py` | Structured logging setup. | Matches Laravel Monolog channel defaults for job context. |
| `tests/test_core.py` | End-to-end + unit coverage across config, tooling, dedup, processor, handler, runpod worker. | Regression harness parallel to Laravel Pest tests pending. |
| `tests/test_handler.py`, `tests/test_webhook.py` | Focused webhook/handler contracts & failure modes. | Mirrors HTTP tests in `tests/Feature/ViralMoments`. |
| `Dockerfile`, `scripts/ghcr.sh`, `scripts/run_tests.sh` | Container build, GHCR publish helper, CI entrypoint. | Align with GitHub Actions workflow for RunPod deployments. |

_All modules now carry 100 % line coverage via the Python test suite._

## Tooling Parity Checklist
| Tool (Laravel)                     | Purpose                                                      | Python Port Status |
|-----------------------------------|--------------------------------------------------------------|--------------------|
| `analyze_transcript_structure`    | Speaker/topic analysis & context for downstream tools        | ✅ Ported (Python) |
| `find_segment_boundaries`         | Natural start/end detection within duration limits           | ✅ Ported (Python) |
| `calculate_word_boundaries`       | Word-level clamping to avoid mid-word/mid-sentence starts    | ✅ Ported (Python) |
| `generate_platform_copy`          | Platform-specific copy generation with tone + limits         | ⚠ Using AI output + limit checks |
| `validate_viral_moment`           | Enforces duration/hooks/category constraints                 | ✅ Ported (Python) |
| `postprocess_deduplicate`         | IoU-based pruning across overlapping chunk outputs           | ✅ Greedy IoU      |

## Post-processing Guards
- **Hook landing window** – first spoken word must begin ≤2 000 ms from the refined start (skipped when no words).
- **Sentence alignment** – starts that drift more than 600 ms into a sentence snap back to the sentence boundary while respecting 40–90 s duration constraints.
- **Quote reconstruction** – rebuilds quotes from word-level timing, falling back to sentence text → AI-provided text → placeholder.
- **Default platforms & score parity** – responses always include (Twitter, Instagram Reels, YouTube Shorts, TikTok, LinkedIn) and map `scorecard`/`viralScore` both ways for Laravel compatibility.
- **Duplicate pruning** – IoU ≥0.85 or ±1.5 s proximity retains best candidate by average score, richness, earliest start, then shortest duration.
- **Telemetry & Webhooks** – `ServiceResult.metadata` reports `moments_before_dedup` / `moments_after_dedup`. Completion webhooks emit `{job_id,status,result|error}` payloads signed with HMAC-SHA256 (`X-Webhook-Signature`) and reuse exponential backoff. If only a completion hook is provided the service appends `/progress` automatically.

## Concurrency Model
- `ThreadPoolExecutor` uses `min(MAX_CONCURRENCY, total_chunks)` workers.
- Fail-fast: any chunk failure cancels outstanding work, emits `chunk_failed` + failure webhook, and raises upstream.

## Configuration
Environment variables (see `src/viral_moments_service/config.py`):

| Variable                  | Default        | Description                                   |
|---------------------------|----------------|-----------------------------------------------|
| `OPENAI_API_KEY`          | _required_*    | Default key for OpenAI requests                |
| `OPENAI_MODEL`            | `gpt-4o-mini`  | Chat completion model                          |
| `OPENAI_TEMPERATURE`      | `0.25`         | Sampling temperature                           |
| `OPENAI_MAX_OUTPUT_TOKENS`| `1800`         | Output token cap per chunk                     |
| `MAX_CONCURRENCY`         | `4`            | Worker pool size                               |
| `CHUNK_DURATION_MS`       | `450000`       | Configurable via options / env                 |
| `CHUNK_OVERLAP_MS`        | `45000`        | Overlap between adjacent chunks                |
| `WEBHOOK_SECRET`          | _unset_        | Secret key for HMAC signatures (optional)      |
| `DERIVE_PROGRESS_WEBHOOK` | `true`         | Auto-append `/progress` when only result URL   |

_*Either set this variable or supply `openai_api_key` inside each job payload._

## Webhook Behaviour
- **Progress** (`status="in_progress"`): `event`, `job_id`, progress payload.
- **Completion** (`status="completed"`): `result.total_chunks`, `result.moments`.
- **Failure** (`status="failed"`): `error` string + optional context.
- Retries: attempts = `WEBHOOK_RETRY_ATTEMPTS`, exponential backoff capped by `WEBHOOK_MAX_DELAY`.
- Signatures: `HMAC-SHA256(secret, payload_json)` in `X-Webhook-Signature`.

## Sync Mode (Direct Execution)
- Call `viral_moments_service.handler.handle_sync(payload)` with the same JSON used for RunPod jobs; returns serialized `ServiceResult`.
- Webhooks still fire when configured, allowing tests to validate callbacks.
- RunPod handler adds metadata (`handler_time_seconds`, `runpod_job_id`, `gpu_used`).

## Deployment
1. **Build Image**: use project scripts to build/push `ghcr.io/oak-ridge-ventures/viral-moments-service:<tag>`.
2. **RunPod Template**: command `python -m viral_moments_service.runpod_worker --serve`, provide `OPENAI_API_KEY` & configuration env vars.
3. **Serverless Endpoint**: configure concurrency slots, attach webhook URLs, supply optional secret for signing.
4. **Laravel Integration**: replace local pipeline with RunPod invocation; verify webhook signatures using shared secret; map `scorecard` → `viralScore` before persistence.

## Testing & Coverage
- Run unit suite with `python -m unittest discover ViralMomentsService/tests` or `coverage run --source=viral_moments_service -m unittest discover`.
- Coverage target: 100% across all modules (tooling, dedup, chunking, webhooks, handler, processor).
- Tests rely on helper constructors to bypass alias friction when instantiating Pydantic models.

## Roadmap
1. ✅ Toolchain port parity.
2. ✅ Word-level clamping & quote reconstruction.
3. ✅ IoU-based deduplication before final webhook dispatch.
4. ✅ Signed webhooks + sync mode.
5. ⏳ Regression harness with real transcript fixtures + Laravel contract tests.
6. ⏳ Replace `datetime.utcnow()` with timezone-aware timestamps.
