# Viral Moments Microservice

High-level reference for the Viral Moments service that packages long-form transcripts into short-form, social-ready clips. This document focuses on externally consumable behaviour and configuration.

## Pipeline Overview
- **Input** – Jobs provide a transcript (sentences + optional words), metadata, webhook endpoints, and runtime overrides such as chunk duration or concurrency limits.
- **Chunking** – Transcripts are segmented into overlapping windows (default 450 s duration, 45 s overlap, 180 s minimum) to balance context retention with throughput.
- **AI Pass** – Each chunk is analysed with a structured prompt that enforces JSON schema constraints, timing requirements, hook limits, and per-platform copy.
- **Refinement** – Returned candidates are clamped to sentence/word boundaries, hooks and quotes are rebuilt, platform copy is verified against limits, and invalid moments are discarded.
- **Aggregation** – Moments from every chunk are merged, deduplicated, scored, and combined with usage + pricing telemetry. Signed progress and completion webhooks keep the caller informed.

## Core Capabilities
- **Deterministic Schema** – Responses adhere to a fixed JSON contract (`ServiceResult` + `ViralMoment`) documented in `docs/ViralMomentResponse.md`.
- **Platform Copy Enforcement** – Twitter/X, Instagram Reels, YouTube Shorts, TikTok, and LinkedIn copy are always present and trimmed to platform-specific character limits.
- **Scorecard Guarantees** – Every moment ships with the five-part scorecard (`shareability`, `emotionalImpact`, `surpriseFactor`, `rewatchability`, `hookStrength`) mirrored to `viralScore` for compatibility with existing consumers.
- **Boundary Validation** – Clip durations are constrained to 40–90 s, hooks must land within the first 2 s, and quotes derive from precise word timings when available.
- **Duplicate Pruning** – Overlapping or near-identical clips are collapsed using IoU and scoring heuristics to keep the strongest candidate.
- **Telemetry & Pricing** – Token usage is normalised across chunks, and per-model pricing estimates are included for simple cost attribution.
- **Webhook Instrumentation** – Progress events (`chunks_scheduled`, `chunk_completed`, `chunk_failed`) and completion/failure payloads are signed when a webhook secret is configured.

## Prompt & Output Expectations
- Maximum of three moments per chunk to prevent low-quality spam.
- Hooks limited to 120 characters and rewritten when the AI output exceeds the cap.
- Categories delivered in Title Case, joining multiple facets with ` / `.
- Engagement value summarises why the clip resonates (tension, emotional impact, etc.).
- Returned `platforms` default to the full five-network list unless explicitly filtered later.

## Post-processing Safeguards
- **Duration Enforcement** – Moments outside the 40–90 s window are rejected after boundary adjustments.
- **Hook Placement** – Hooks must start in the first 2 s; otherwise the moment is dropped.
- **Sentence Alignment** – Start/end boundaries snap to nearby sentence or word edges to avoid truncating speech.
- **Quote Reconstruction** – Word-level timings rebuild quotes for accuracy, falling back to sentence text or AI output only when necessary.
- **Score Integrity** – Missing or partial scorecards cause the candidate to be removed before aggregation.

## Configuration Highlights

| Environment Variable | Default | Purpose |
| --- | --- | --- |
| `OPENAI_API_KEY` | — | Primary key when the payload omits `openai_api_key`. |
| `OPENAI_MODEL` | `gpt-4o-mini` | AI model used for the Responses API call (supports `gpt-5`, `o3`, etc.). |
| `OPENAI_TEMPERATURE` | `0.25` | Sampling temperature applied per chunk when supported by the model. |
| `OPENAI_MAX_OUTPUT_TOKENS` | `1800` | Maximum output tokens per chunk. |
| `MAX_CONCURRENCY` | `4` | Upper bound for concurrent chunk processing. |
| `CHUNK_DURATION_MS` | `450000` | Default chunk duration; override per request if needed. |
| `CHUNK_OVERLAP_MS` | `45000` | Overlap to preserve context between windows. |
| `WEBHOOK_SECRET` | — | Enables HMAC-SHA256 signatures for progress/result payloads. |
| `DERIVE_PROGRESS_WEBHOOK` | `true` | Automatically appends `/progress` to the result URL when no explicit progress hook is provided. |

## Webhook Behaviour
- **Progress Payloads** – Emitted for `chunks_scheduled`, `chunk_completed`, and `chunk_failed` with job + chunk metadata.
- **Completion Payload** – Includes `status: completed`, the full `ServiceResult`, token usage, and pricing details.
- **Failure Payload** – Includes `status: failed`, an error string, and (when available) the chunk index that triggered the failure.
- **Signing** – When `WEBHOOK_SECRET` is present, payloads are signed using `hex(hmac_sha256(secret, body))` in `X-Webhook-Signature`.
- **Retries** – Exponential backoff driven by `WEBHOOK_RETRY_ATTEMPTS`, `WEBHOOK_RETRY_BASE_DELAY_SECONDS`, and `WEBHOOK_RETRY_MAX_DELAY_SECONDS`.

## Deployment Snapshot
1. Build or pull the published container image (`ghcr.io/oak-ridge-ventures/viral-moments-service:<tag>`).
2. Configure environment variables (OpenAI, webhook secret, concurrency, timeouts).
3. Deploy to RunPod Serverless with command `python -m viral_moments_service.runpod_worker --serve` and set webhook URLs.
4. Use the synchronous API or webhook-enabled async mode to validate the integration with sample payloads.

## Testing & Validation Tips
- Generate deterministic payloads from transcripts with `scripts/generate_payload.py`.
- Exercise the RunPod endpoint end-to-end using `scripts/test-runpod-endpoint.sh` or the sync examples in `docs/RunPodIntegration.md`.
- Inspect `runpod_test_output.json` or sync responses to compare against the schema in `docs/ViralMomentResponse.md`.

This document intentionally avoids implementation specifics so it can be shared with partners who only interact with the hosted service.
