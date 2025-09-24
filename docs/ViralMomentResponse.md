# Viral Moment Response Contract

Reference specification for the `ServiceResult` payload returned by both synchronous runs and RunPod webhooks. All fields use camelCase to stay interoperable with Laravel consumers.

## 1. Top-Level Envelope (`ServiceResult`)

| Field | Type | Description |
| --- | --- | --- |
| `job_id` | string | Identifier supplied in the request payload. Mirrors RunPod job id when invoked via Serverless. |
| `total_chunks` | integer | Total number of transcript chunks scheduled for processing. |
| `successful_chunks` | integer | Count of completed chunks. Equals `total_chunks` unless a failure short-circuits processing. |
| `failed_chunks` | integer | Chunks that did not complete. Non-zero implies the request raised before all results were aggregated. |
| `moments` | `ViralMoment[]` | Deduplicated viral moment records ready for persistence in Laravel. Ordered by start time after pruning. |
| `metadata` | object | Execution metadata described below. Always present. |
| `usage` | object \| `null` | Aggregated OpenAI token usage; omitted when OpenAI does not return token data. |

### 1.1 `metadata`

| Field | Type | Description |
| --- | --- | --- |
| `chunk_duration_ms` | integer | Effective chunk duration in milliseconds after applying request overrides. |
| `overlap_ms` | integer | Overlap between adjacent chunks. |
| `model` | string | OpenAI model actually invoked (post override). |
| `moments_before_dedup` | integer | Number of raw moments emitted across all chunks prior to duplicate pruning. |
| `moments_after_dedup` | integer | Length of the final `moments` array. |
| `usage_totals` | object | Flattened token totals (same structure as `usage.totals`). Empty when token data is unavailable. |
| `pricing` | object | Cost estimate generated from `usage_totals` and the internal pricing table. Empty object if usage could not be calculated. |
| `video_id`, `video_title`, `audience`, `tone`, ... | any | All custom metadata values from the request are forwarded verbatim when present. |

### 1.2 `usage`

```json
{
  "totals": {
    "input_tokens": 220312,
    "output_tokens": 135096,
    "input_cached_tokens": 218752,
    "output_reasoning_tokens": 117376,
    "prompt_cached_tokens": 110000,
    "prompt_cache_read_tokens": 110000,
    "prompt_cache_write_tokens": 0
  },
  "by_chunk": {
    "0": {
      "input_tokens": 10986,
      "output_tokens": 6048,
      "input_cached_tokens": 10920,
      "response_time_ms": 8892,
      "input_tokens_details": {
        "cached_tokens": 10920,
        "audio_tokens": 0
      },
      "output_tokens_details": {
        "reasoning_tokens": 512
      },
      "prompt_tokens_details": {
        "cached_tokens": 10920,
        "cache_read_tokens": 10920,
        "cache_write_tokens": 0
      }
    }
  }
}
```

- Totals normalise OpenAI aliases (text vs. audio) into canonical keys.
- `by_chunk` keys are integers stringified to satisfy JSON object requirements.
- Additional OpenAI-provided attributes (e.g. `response_time_ms`) are preserved verbatim under each chunk entry.

### 1.3 `pricing`

```json
{
  "model": "gpt-5",
  "rates_per_million": {
    "input": 1.25,
    "cached": 0.125,
    "output": 10.0
  },
  "tokens": {
    "input": 220312,
    "cached": 218752,
    "output": 135096
  },
  "cost": {
    "input": 0.27539,
    "cached": 0.027344,
    "output": 1.35096,
    "total": 1.653694
  }
}
```

- The calculator resolves pricing for `gpt-5`, `gpt-4o`, `gpt-4o-mini`, `o3`, and `o3-mini` (case-insensitive, prefix match).
- All monetary values are six decimal floats rounded half up.

## 2. Viral Moment Schema (`ViralMoment`)

| Field | Type | Required | Description |
| --- | --- | --- | --- |
| `title` | string | ✓ | Title-cased short headline (<120 chars enforced downstream). |
| `startTime` | integer | ✓ | Clip start in **absolute** milliseconds within the source transcript. |
| `endTime` | integer | ✓ | Clip end in absolute milliseconds. Always greater than `startTime`. |
| `duration` | integer | ✓ | `endTime - startTime`. Guaranteed between 40 000 and 90 000 ms after refinement. |
| `hook` | string | ✓ | 120-character opening designed to land within the first 2 000 ms. |
| `category` | string | ✓ | Title Case label; combine multiple facets with ` / ` (e.g. `Growth / Mindset`). |
| `engagementValue` | string | ✓ | Free-form rationale describing why the clip is compelling. |
| `quote` | string | — | Word-for-word snippet reconstructed from transcript words; falls back to AI text. |
| `platforms` | string[] | ✓ | Platforms this moment targets. Defaults to `twitter`, `instagram_reels`, `youtube_shorts`, `tiktok`, `linkedin`. |
| `socialCopy` | object | ✓ | Platform-specific copy; keys mirror `platforms`. Length limits enforced per platform (see below). |
| `scorecard` | object | ✓ | Scoring rubric consumed by Laravel. Always mirrored to `viralScore`. |
| `viralScore` | object | — | Alias for `scorecard`. Added automatically if omitted by the model. |
| `chunk_index` | integer | — | Index of the source chunk (0-based). Included for debugging/analytics. |

> Validation guarantees `title`, `hook`, and `platforms` are non-empty. Any moment failing these rules is discarded.

### 2.1 `socialCopy`

- Keys: `twitter`, `instagram_reels`, `youtube_shorts`, `tiktok`, `linkedin` (additional keys from the model are dropped).
- Character limits: 280 (Twitter/X), 400 (Instagram Reels & LinkedIn), 150 (TikTok), 100 (YouTube Shorts).
- Text is rewritten to fit limits; failures raise validation errors and remove the moment.

### 2.2 `scorecard` / `viralScore`

```json
{
  "shareability": 9,
  "emotionalImpact": 7,
  "surpriseFactor": 8,
  "rewatchability": 6,
  "hookStrength": 9
}
```

- Integers 1–10. The validator ensures both `scorecard` and `viralScore` exist; whichever is missing is cloned from the other.
- Additional metrics from future prompt iterations are preserved if supplied, but parity with Laravel today expects the five core keys above.

### 2.3 Additional Notes

- **Duration Enforcement** – Moments shorter than 40 s or longer than 90 s are rejected after transcript-aware boundary adjustments.
- **Hook Placement** – Hooks must land within the first 2 000 ms; outliers are removed during validation.
- **Quotes** – Reconstructed from the word timeline to avoid truncation mid-word; falls back to sentence text or AI text if words are missing.
- **Deduplication** – IoU and score heuristics remove overlapping clips, keeping the richest candidate (best average score, highest richness, earliest start).

## 3. Example Response

```json
{
  "job_id": "runpod-test-123",
  "total_chunks": 22,
  "successful_chunks": 22,
  "failed_chunks": 0,
  "moments": [
    {
      "title": "The 3-Hour Rule That Tripled Our Retention",
      "startTime": 174000,
      "endTime": 224000,
      "duration": 50000,
      "hook": "Want customers to stick around? Give them this one habit day one.",
      "category": "Retention Strategy / Customer Onboarding",
      "engagementValue": "Reframes onboarding as a time-bound game, sparking urgency and curiosity.",
      "quote": "So we made a three-hour challenge: if they ship one action by lunch, they stay for months.",
      "platforms": ["twitter", "instagram_reels", "youtube_shorts", "tiktok", "linkedin"],
      "socialCopy": {
        "twitter": "We cut churn 41% with a single day-one habit. Give new users a 3-hour mission and watch retention pop.",
        "instagram_reels": "Onboarding hack: turn day one into a 3-hour mission. Sprint, win, stay.",
        "youtube_shorts": "This 3-hour onboarding mission kept 41% more users.",
        "tiktok": "Give new users a 3-hour mission and retention jumps.",
        "linkedin": "Turn onboarding into a 3-hour mission. It tripled our retention."
      },
      "scorecard": {
        "shareability": 9,
        "emotionalImpact": 7,
        "surpriseFactor": 8,
        "rewatchability": 6,
        "hookStrength": 9
      },
      "viralScore": {
        "shareability": 9,
        "emotionalImpact": 7,
        "surpriseFactor": 8,
        "rewatchability": 6,
        "hookStrength": 9
      },
      "chunk_index": 6
    }
  ],
  "metadata": {
    "chunk_duration_ms": 450000,
    "overlap_ms": 45000,
    "model": "gpt-5",
    "moments_before_dedup": 13,
    "moments_after_dedup": 5,
    "usage_totals": {
      "input_tokens": 220312,
      "output_tokens": 135096,
      "input_cached_tokens": 218752
    },
    "pricing": {
      "model": "gpt-5",
      "rates_per_million": {
        "input": 1.25,
        "cached": 0.125,
        "output": 10.0
      },
      "tokens": {
        "input": 220312,
        "cached": 218752,
        "output": 135096
      },
      "cost": {
        "input": 0.27539,
        "cached": 0.027344,
        "output": 1.35096,
        "total": 1.653694
      }
    }
  },
  "usage": {
    "totals": {
      "input_tokens": 220312,
      "output_tokens": 135096,
      "input_cached_tokens": 218752
    },
    "by_chunk": {
      "6": {
        "input_tokens": 10986,
        "output_tokens": 6048,
        "input_cached_tokens": 10920,
        "response_time_ms": 8892
      }
    }
  }
}
```

Use this contract to validate deserialisation on the Laravel side, populate analytics dashboards, or snapshot results in the dedicated documentation repository.
