# RunPod Integration Guide

This externally shareable guide covers everything required to operate the Viral Moments microservice on RunPod Serverless—credentials to gather, how to exercise the endpoint, bundled test scripts, and the webhook lifecycle.

## 0. Quick Checklist

- ✅ **RunPod API key** (`RUNPOD_API_KEY`) authorised for the Viral Moments endpoint.
- ✅ **RunPod endpoint ID** from the RunPod console (`<endpointId>` in API calls).
- ✅ **OpenAI API key** via environment (`OPENAI_API_KEY`) or payload field `openai_api_key`.
- ✅ **Webhook URLs** (result + optional progress) reachable over HTTPS.
- ✅ **Webhook secret** (if signatures are required) mirrored between RunPod worker and receiver.
- ✅ **Local tooling**: `scripts/generate_payload.py` to create payloads, `scripts/test-runpod-endpoint.sh` to fire a full RunPod run.

## 1. Prerequisites

| Requirement | Notes |
| --- | --- |
| OpenAI API key | Provide via `OPENAI_API_KEY` env var or embed `openai_api_key` inside the request payload. |
| Model defaults | `OPENAI_MODEL` defaults to `gpt-4o-mini`; change to `gpt-5` or others via env or request overrides. |
| RunPod credentials | `RUNPOD_API_KEY` + endpoint ID required for API calls in CI/local tooling. |
| Optional webhook secret | Set `WEBHOOK_SECRET` for HMAC-SHA256 signatures (`X-Webhook-Signature`). |
| Timeouts & concurrency | `REQUEST_TIMEOUT_SECONDS` (default `30`), `MAX_CONCURRENCY` (default `4`). |

> Tip: `DERIVE_PROGRESS_WEBHOOK=true` appends `/progress` to the supplied result URL when no explicit progress hook is provided, mirroring the behaviour expected by existing downstream systems.

## 2. Request Schema

```json
{
  "job_id": "runpod-test-123",
  "openai_api_key": "sk-...",
  "transcript": {
    "language": "en",
    "sentences": [
      {"text": "Hello", "start": 0, "end": 5000, "speaker_id": "speaker_1"}
    ],
    "words": [
      {"text": "Hello", "start": 0, "end": 1000, "sentence_index": 0}
    ]
  },
  "metadata": {
    "video_id": "abc123",
    "video_title": "Sample Interview",
    "audience": "Founders"
  },
  "options": {
    "chunk_duration_ms": 450000,
    "chunk_overlap_ms": 45000,
    "min_chunk_duration_ms": 180000,
    "max_concurrency": 10,
    "openai_model": "gpt-5",
    "openai_temperature": 0.25
  },
  "webhook_endpoints": {
    "result_webhook_url": "https://example.com/webhooks/result",
    "progress_webhook_url": "https://example.com/webhooks/progress"
  }
}
```

Key points:

- `openai_api_key` is optional when the worker sets `OPENAI_API_KEY`.
- Metadata is passed through to prompts (`video_title`, `audience`, `tone`) and echoed in the response under `ServiceResult.metadata`.
- Options are merged with environment defaults; unknown keys are ignored.
- `max_concurrency` cannot exceed the configured `MAX_CONCURRENCY` environment value.

## 3. Job Lifecycle

1. **Submit** – POST to `https://api.runpod.ai/v2/<endpointId>/run` with the payload above.
2. **Queued** – RunPod returns `{ "id": "<jobId>" }`. Use `/status/<jobId>` to poll or rely on webhooks.
3. **Processing** – The worker emits progress events (`chunks_scheduled`, `chunk_completed`, `chunk_failed`) if a progress webhook is configured.
4. **Completed / Failed** – The result webhook receives the final payload. Sync mode returns the same structure immediately.

Example submission:

```bash
curl -X POST "https://api.runpod.ai/v2/$RUNPOD_ENDPOINT_ID/run" \
  -H "Authorization: Bearer $RUNPOD_API_KEY" \
  -H "Content-Type: application/json" \
  -d @payload.json
```

## 4. Local Test Scripts

| Script | Usage | Notes |
| --- | --- | --- |
| `scripts/generate_payload.py` | `python scripts/generate_payload.py --sentences sentences.json --words words.json --output payload.json` | Builds a valid `payload.json` from transcript fixtures. Adjust CLI flags to point at your assets. |
| `scripts/test-runpod-endpoint.sh` | `RUNPOD_API_KEY=... RUNPOD_ENDPOINT_ID=... OPENAI_API_KEY=... ./scripts/test-runpod-endpoint.sh payload.json` | Fires a RunPod job, streams status updates, and saves the final response to `runpod_test_output.json`. Requires `jq` for formatting. |

> For local smoke tests outside of RunPod, execute the published container with the sample payload and inspect the printed `ServiceResult`.

## 5. RunPod Sync API Examples

RunPod exposes a synchronous variant that blocks until the job completes. This is useful for tooling or when webhook infrastructure is unavailable.

### 5.1 `curl`

```bash
curl -X POST "https://api.runpod.ai/v2/$RUNPOD_ENDPOINT_ID/run-sync" \
  -H "Authorization: Bearer $RUNPOD_API_KEY" \
  -H "Content-Type: application/json" \
  -d @payload.json \
  | jq
```

The response body matches the asynchronous result payload (`status`, `result`, `moments`, `usage`, `pricing`). RunPod caps synchronous execution time; long transcripts may require asynchronous submission instead.

### 5.2 Python (`requests`)

```python
import json
import os
import requests

endpoint = os.environ["RUNPOD_ENDPOINT_ID"]
api_key = os.environ["RUNPOD_API_KEY"]

with open("payload.json", "r", encoding="utf-8") as fh:
    payload = json.load(fh)

resp = requests.post(
    f"https://api.runpod.ai/v2/{endpoint}/run-sync",
    headers={
        "Authorization": f"Bearer {api_key}",
        "Content-Type": "application/json",
    },
    json=payload,
    timeout=900,
)
resp.raise_for_status()
result = resp.json()
print(json.dumps(result, indent=2))
```

## 6. Webhook Payloads

Webhooks surface three high-level statuses:

- `in_progress` events (progress hook) for chunk lifecycle: `chunks_scheduled`, `chunk_completed`, `chunk_failed`.
- `completed` payload (result hook) containing the final `ServiceResult` envelope with deduplicated moments, usage, and pricing.
- `failed` payload (result hook) when processing aborts early, including the failing chunk index and error message.

### Progress Event

```json
{
  "status": "in_progress",
  "event": "chunk_completed",
  "job_id": "runpod-test-123",
  "data": {
    "chunk_index": 6,
    "total_chunks": 22,
    "started_at": "2024-05-14T16:41:00.512Z",
    "completed_at": "2024-05-14T16:41:09.948Z"
  }
}
```

### Completion Payload

```json
{
  "status": "completed",
  "job_id": "runpod-test-123",
  "result": {
    "total_chunks": 22,
    "successful_chunks": 22,
    "failed_chunks": 0,
    "moments": [ /* ViralMoment[] */ ],
    "metadata": {
      "video_id": "abc123",
      "moments_before_dedup": 27,
      "moments_after_dedup": 9,
      "pricing": { /* see section 7 */ }
    },
    "usage": {
      "totals": {
        "input_tokens": 220312,
        "output_tokens": 135096,
        "input_cached_tokens": 218752,
        "output_reasoning_tokens": 117376,
        "total_tokens": 355408
      },
      "by_chunk": {
        "0": {
          "input_tokens": 10986,
          "output_tokens": 6048,
          "input_cached_tokens": 10920,
          "response_time_ms": 8892
        }
      }
    }
  }
}
```

### Failure Payload

```json
{
  "status": "failed",
  "job_id": "runpod-test-123",
  "error": "OpenAI responses call failed: ...",
  "data": {
    "failed_chunk": 7,
    "total_chunks": 22
  }
}
```

> When `WEBHOOK_SECRET` is set, each webhook body is signed with `hex(hmac_sha256(secret, raw_body))` and surfaced in `X-Webhook-Signature`.

Retries follow exponential backoff: attempts = `WEBHOOK_RETRY_ATTEMPTS` (default `3`), base delay = `WEBHOOK_RETRY_BASE_DELAY_SECONDS` (`1.5`), capped by `WEBHOOK_RETRY_MAX_DELAY_SECONDS` (`12`).

## 7. Result & Pricing Schema

Sync and async executions share the same `ServiceResult` envelope documented in [docs/ViralMomentResponse.md](./ViralMomentResponse.md). Highlights:

- `moments[]` – Normalised `ViralMoment` models (absolute ms boundaries, hooks, category, quote, per-platform copy, scorecard & viralScore mirroring).
- `usage` – Canonical token metrics with alias normalisation for text/audio responses; `response_time_ms` is included when available from OpenAI streaming endpoints.
- `metadata.pricing` – Cost estimate derived from the bundled pricing table (`gpt-5`, `gpt-4o`, `gpt-4o-mini`, `o3`, `o3-mini`). Includes `rates_per_million`, actual `tokens`, and monetised `cost` breakdown plus `total`.

## 8. Configuration Overrides

| Option | Type | Default | Description |
| --- | --- | --- | --- |
| `chunk_duration_ms` | int | 450000 | Target window size per chunk. |
| `chunk_overlap_ms` | int | 45000 | Sliding overlap to preserve context. |
| `min_chunk_duration_ms` | int | 180000 | Ensures tail segments merge to satisfy 40 s minimum. |
| `max_concurrency` | int | `MAX_CONCURRENCY` | Per-request worker limit; clamped to env. |
| `openai_model` | str | `OPENAI_MODEL` | Override the Responses model for one job. |
| `openai_temperature` | float | `OPENAI_TEMPERATURE` | Only applied when the model supports temperature. |
| `openai_max_output_tokens` | int | `OPENAI_MAX_OUTPUT_TOKENS` | Responses API max output tokens. |
| `request_timeout_seconds` | float | `REQUEST_TIMEOUT_SECONDS` | Applied to OpenAI + webhook HTTP calls. |

Unrecognised options are ignored, ensuring forward compatibility.

## 9. Sync Mode Reference

Run the handler directly (useful for contract tests or quick transcript experiments):

```bash
docker run --rm -i --platform=linux/amd64 \
  -e OPENAI_API_KEY=sk-... \
  -e MAX_CONCURRENCY=10 \
  -e REQUEST_TIMEOUT_SECONDS=1800 \
  -v "$(pwd)/payload.json:/tmp/payload.json:ro" \
  ghcr.io/oak-ridge-ventures/viral-moments-service:main-5091c69 \
  python - <<'PY'
import json
from pathlib import Path
from viral_moments_service.handler import handle_sync
payload = json.loads(Path('/tmp/payload.json').read_text())
result = handle_sync(payload.get('input', payload))
print(json.dumps(result, indent=2))
PY
```

The same entrypoint powers RunPod jobs (`handle_sync` behind the scenes), so running locally is representative of serverless behaviour.

## 10. Troubleshooting

- **Empty `moments` array** – Check transcript quality and ensure hooks land in the first 2 s; the AI will drop mediocre clips rather than force results.
- **Validation errors** – Inspect logs for `Discarding moment due to validation error`; boundary clamps can fail when transcripts lack words/sentences in the requested range.
- **Webhook timeouts** – Increase `WEBHOOK_RETRY_ATTEMPTS` / `WEBHOOK_RETRY_MAX_DELAY_SECONDS` or ensure the receiving service responds within RunPod’s timeout window.
- **Token spikes** – Large transcripts can create many chunks. Tune `chunk_duration_ms` and optionally reduce `max_concurrency` to match your OpenAI quota.

With these details you can run the service locally for contract tests or integrate it into RunPod Serverless with full webhook support, pricing telemetry, and hardened error handling.
