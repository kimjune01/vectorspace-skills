# Skill: Uninstall Ad Integration

> **Operator note:** Run this skill interactively. Review the proposed deletions before allowing the assistant to commit. Verify that no publisher-authored code is being removed.

You are a Forward Deployed Engineer working on behalf of the publisher. They want to remove the VectorSpace ad integration from their codebase. Remove it cleanly, leaving no dead code, no orphaned config, and no broken references.

## How to find integration code

The Install skill tagged every code block it added with `[vectorspace]...[/vectorspace]` comment markers. Search the entire codebase for these markers:

```
[vectorspace]    — start of an integration block
[/vectorspace]   — end of an integration block
```

These markers use the language's native comment syntax (`//`, `#`, `/* */`, `<!-- -->`). Search for the string `[vectorspace]` across all files to find every block.

## Removal process

### 1. Inventory

Search the codebase for all `[vectorspace]` markers. List every file and checkpoint found:

| File | Checkpoint | Lines |
|---|---|---|
| ... | ... | ... |

Show this inventory to the operator before removing anything.

### 2. Remove tagged blocks

For each `[vectorspace]...[/vectorspace]` block:
- Delete the block including the start and end markers
- Check if the removal leaves empty functions, unused imports, or orphaned variables
- Clean up any artifacts (empty lines, broken indentation, dangling commas)

### 3. Remove config values

Search for and remove:
- `VECTORSPACE_ENABLED` — from config files, env files, environment variable declarations
- `VECTORSPACE_TARGET_RATE` — same
- `VECTORSPACE_PUBLISHER_ID` — same
- Any stored `tau` value in the database or config store
- Any `consent_given` / `consent_timestamp` records or schema

### 4. Remove dependencies

If the Install skill added any of these, remove them:
- `sentence-transformers` Python package (check if anything else uses it first)
- `BAAI/bge-small-en-v1.5` model references
- Hugging Face API token references (only if added for this integration)

### 5. Remove generated files

Delete any files created entirely by the integration:
- Compliance audit reports (`ad-integration-audit.md`, `COMPLIANCE.md`, etc.)
- The `.vectorspace/` directory if it exists

### 6. Verify

- Run existing tests — they should pass without the integration code
- Check for any remaining references to `vectorspace`, `ad-request`, `ad_request`, or `proximity_score`
- Confirm no outbound HTTP calls to `api.vectorspace.exchange` remain

### 7. Create a PR

1. Create a branch (e.g., `remove/vectorspace-ads`)
2. Commit with a clear message: "Remove VectorSpace ad integration"
3. Push and open a PR

The PR description should confirm:
- All `[vectorspace]` blocks removed
- All config values removed
- All dependencies removed
- Existing tests still pass
