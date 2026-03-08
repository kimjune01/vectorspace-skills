# VectorSpace Publisher Skills

These are prompt documents for your coding assistant (Claude Code, Codex, or similar). They guide the assistant through evaluating, integrating, and verifying embedding-based ads for your chatbot.

## What these are

Three skills that your coding assistant runs inside your codebase:

1. **[evaluate.md](evaluate.md)** — Assesses whether embedding-based ads make sense for your chatbot
2. **[install.md](install.md)** — Adds the ad integration to your codebase (8 auditable checkpoints, with API reference appendix)
3. **[verify.md](verify.md)** — Audits the integration for HIPAA, FTC, and state privacy compliance

They are not an SDK. There is no package to install, no binary, no dependency. Your coding assistant reads the skill document and writes code directly into your existing files.

## How to use them

Copy the skill files into your repository (e.g., in a `.vectorspace/` directory or wherever you keep agent instructions). Then tell your coding assistant to follow the skill.

**Example with Claude Code:**
```
> Read .vectorspace/evaluate.md and follow the instructions to evaluate revenue potential for our chatbot.
```

**Example with Codex:**
```
Follow the instructions in .vectorspace/install.md to add ad integration to our chatbot.
```

## Important: Do not run in autonomous mode

These skills modify your codebase and make decisions about data flow, privacy, and compliance. **Run them interactively — not in fully autonomous (YOLO) mode.**

Each skill has checkpoints where you should review what the assistant is doing:

- **Evaluate**: Review the architecture mapping and traffic estimates before accepting the revenue projection
- **Install**: Review each checkpoint's code before the assistant moves to the next one. Especially review:
  - The opt-in trigger logic (Checkpoint 1) — when does the ad flow activate?
  - The intent extraction prompt (Checkpoint 2) — what gets extracted from conversations?
  - The API call payload (Checkpoint 4) — what leaves your infrastructure?
  - The rendered ad (Checkpoint 6) — how does it appear to your users?
- **Verify**: Review the audit findings and the compliance report before it's committed

The assistant will create a GitHub PR with its changes. Review the PR diff before merging. This is your code, your users, your compliance obligation.

## What happens to your data

The core privacy guarantee:

| Data | Where it stays |
|---|---|
| Conversation text | Your infrastructure only |
| Extracted intent | Your infrastructure only |
| User identity | Your infrastructure only |
| Embedding vector (384 numbers) | Sent to the exchange for ad matching |
| Auction result | Returned from the exchange |
| Attribution events (IDs only) | Sent to the exchange |

The exchange never sees your users' conversations, health information, or identity. It receives a list of 384 numbers and returns an auction result.

## Requirements

- A chatbot with an LLM (any provider — you already have this)
- Access to `BAAI/bge-small-en-v1.5` for embedding (free via Hugging Face API, or run locally)
- A publisher ID from VectorSpace (contact us or register at vectorspace.exchange)

## The skills in order

Run them in sequence. Each builds on the previous:

1. **Evaluate** first — make sure the numbers work before writing any code
2. **Install** second — add the integration with your coding assistant reviewing each step
3. **Verify** third — audit the integration and generate compliance documentation

You can stop after any skill. Evaluate commits nothing. Install creates a PR you can close. Verify produces a report you can discard.

## Questions?

- Architecture and pricing: https://vectorspace.exchange
- Why health chatbots can monetize without a BAA: https://www.june.kim/monetizing-the-untouchable/
- How the auction works: https://www.june.kim/the-easiest-sale/
- How attribution works without tracking: https://www.june.kim/attested-attribution/
