# VectorSpace Publisher Spec

The skills tell you *how*. This document tells you *what*.

API contract: https://www.june.kim/publisher-api (OpenAPI spec, endpoint details, event types, catalog schema).

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

Server-side PID controller that adjusts tau to converge on `VECTORSPACE_TARGET_RATE`.

- Design rationale: https://www.june.kim/set-it-and-forget-it
- Reference implementation: https://github.com/kimjune01/tau-controller

Port the PID logic from the reference implementation into the publisher's language and patterns.

## Proximity Indicator

Full UX spec (state machine, consent flow, recommendation card): https://www.june.kim/publisher-ux
