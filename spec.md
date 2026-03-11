# VectorSpace Publisher Spec

API schemas, payload formats, system prompts, and config values. The skills tell you *how*. This document tells you *what*.

## Config Values

| Config | Type | Default | Purpose |
|---|---|---|---|
| `VECTORSPACE_ENABLED` | boolean | `false` | Feature flag. When `false`, the entire ad branch is skipped. |
| `VECTORSPACE_TARGET_RATE` | float (0–1) | `0.05` | Target fraction of conversations that show the proximity indicator. |
| `VECTORSPACE_PUBLISHER_ID` | string | — | Publisher identifier, provided by VectorSpace. |

Internal (managed by the server, not set by the publisher):

| Value | Type | Initial | Description |
|---|---|---|---|
| `tau` | float (0–1) | `0.8` | Relevance threshold. Auto-tuned to converge on `VECTORSPACE_TARGET_RATE`. Clamped to [0.5, 0.99]. Persisted across restarts. |

## Embedding Model

Model: `BAAI/bge-small-en-v1.5` (384 dimensions). Must match the exchange's advertisers.

**Hugging Face Inference API (free, good for intent extraction):**
```
POST https://api-inference.huggingface.co/pipeline/feature-extraction/BAAI/bge-small-en-v1.5
Authorization: Bearer {HF_TOKEN}
{"inputs": "physical therapist specializing in climbing injuries"}
→ 384-dim float array
```

**Local Python (required for proximity scoring — runs on every message):**
```python
from sentence_transformers import SentenceTransformer
model = SentenceTransformer("BAAI/bge-small-en-v1.5")
vector = model.encode(text).tolist()
```

Use local for Phase 1 (proximity indicator — every message). HF API is acceptable for Phase 2 (intent extraction — only on user tap).

## GET /advertiser-positions.json

Public endpoint. Fetch periodically (e.g., hourly) and cache locally.

```
GET https://api.vectorspace.exchange/advertiser-positions.json
```

Response:

```json
[
  {
    "advertiser_id": "adv-7",
    "embedding": [0.023, -0.041, ...],
    "sigma": 0.3,
    "category": "physical-therapy",
    "updated_at": "2026-03-08T12:00:00Z"
  }
]
```

| Field | Type | Description |
|---|---|---|
| `advertiser_id` | string | Unique advertiser identifier |
| `embedding` | float[384] | Advertiser's position in `bge-small-en-v1.5` embedding space |
| `sigma` | float (0–1) | How broadly the advertiser bids — tight (0.1) = niche, wide (0.5) = broad |
| `category` | string | Human-readable label for the advertiser's vertical |
| `updated_at` | ISO 8601 | When this position was last updated |

## POST /ad-request

Auction request. Sent only after user taps the proximity indicator and consents.

```
POST https://api.vectorspace.exchange/ad-request
Content-Type: application/json

{
  "embedding": [384 floats],
  "publisher_id": "PUBLISHER_ID",
  "tau": 0.8
}
```

| Field | Type | Description |
|---|---|---|
| `embedding` | float[384] | Intent embedding from `bge-small-en-v1.5` |
| `publisher_id` | string | From `VECTORSPACE_PUBLISHER_ID` config |
| `tau` | float (0–1) | Server's current auto-tuned threshold |

Response:

```json
{
  "auction_id": 42,
  "winner_id": "adv-7",
  "payment": 3.47,
  "currency": "USD",
  "bid_count": 12
}
```

If no match or error, the response may be empty or an error status. Handle gracefully.

## POST /event/impression

Fire when the ad suggestion is rendered and visible to the user.

```
POST https://api.vectorspace.exchange/event/impression
Content-Type: application/json

{"auction_id": 42, "advertiser_id": "adv-7", "publisher_id": "PUBLISHER_ID"}
```

## POST /event/click

Fire when the user taps/clicks the ad suggestion.

```
POST https://api.vectorspace.exchange/event/click
Content-Type: application/json

{"auction_id": 42, "advertiser_id": "adv-7", "publisher_id": "PUBLISHER_ID"}
```

No user identity in attribution payloads. Just IDs for the auction, advertiser, and publisher.

## Intent Extraction System Prompt

Call the publisher's existing LLM with this as the system prompt:

```
Given a conversation, decide whether the person could benefit from a professional service. If yes, write a single sentence describing that service — as if the provider were writing their own position statement. If the conversation is casual, off-topic, or doesn't suggest any professional need, respond with exactly "NONE".

Format: [value prop] + [ideal client profile] + [qualifier]
Example: "Sports injury knee rehab for competitive endurance athletes recovering from overuse."

Rules:
- Match the most obvious need. A health complaint needs a health provider, not a lawyer. A legal issue needs legal help, not a therapist.
- Be specific to the situation but don't embellish beyond what's stated.
- Do NOT extract demographics or personal data about the user.
- If there is no clear professional need, respond with "NONE".

Respond with ONLY the one-sentence service description or "NONE", nothing else.
```

If the LLM returns "NONE", stop. No ad opportunity.

## Consent Record

Persisted per user (lifetime of the app, not per session).

```json
{
  "consent_given": true,
  "consent_timestamp": "2026-03-08T14:30:00Z"
}
```

Default consent dialog wording:

> "Would you like to see a recommendation from our sponsor?"
>
> **[Yes, show me]**  **[No thanks]**

Adapt to match the chatbot's voice. If declined, don't show the proximity indicator on future sessions.

## Auto-Tuning Algorithm

Server-side. Adjusts tau to converge on `VECTORSPACE_TARGET_RATE`.

Reference implementation: https://github.com/kimjune01/tau-controller (MIT, simulation evidence, interactive notebook).

The controller uses a PID feedback loop:
- **Proportional**: if the recommendation rate exceeds the target, tighten tau. If below, loosen.
- **Integral**: accumulated drift over time forces correction that the proportional term alone wouldn't catch.
- **Derivative**: sudden changes (new advertiser category enters) are detected and corrected before error accumulates.

Implementation requirements:
1. Track recommendation rate per conversation (not per turn). One recommendation per conversation, max.
2. Update tau every N completed conversations (default: 100).
3. Clamp tau to [0.5, 0.99]. Clamp the integral to prevent windup.
4. Initialize tau to `0.8` on first run.
5. Persist tau across restarts.

Port the PID logic from the reference implementation into the publisher's language and patterns. The reference is Python; adapt to whatever the codebase uses.

## Proximity Indicator Styles

Ask the operator which fits their UI:

| Style | Description | Best for |
|---|---|---|
| **Glowing dot** | A small dot beside the latest assistant message that brightens as relevance increases | Chat UIs with message bubbles |
| **Avatar ring** | A ring around the chatbot's avatar that glows or pulses with relevance | UIs with a bot avatar/icon |
| **Subtle shimmer** | A faint highlight or shimmer on the assistant message border | Minimal / text-heavy UIs |
