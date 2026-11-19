---
status: accepted
date: 2026-11-19
decided-by: Niko4417
ticket: https://github.com/Niko4417/OPT-cc/issues/20
---

# Sync GitHub repositories and their embeddings incrementally

A sync of a connected GitHub repository recomputes only what changed, in both the ingestion step and the indexing step. A full rerun would mean about 6,000 GitHub calls, and about 16,000 calls of 25 records each to reload the Bedrock arm for a mid-size project memory.

**Code.** Each run does `git fetch`, then `git diff --name-status <last-sha> <new-sha>`, and chunks again only changed, added or deleted files. If the last SHA is no longer an ancestor of the new head, because of a force push or rewritten history, the code is chunked again in full.

**GitHub items.** Each run calls the repository-wide endpoints for issues, conversation comments and review comments with `since` set to the last run minus a 1-hour overlap. It lists review bodies for the pull requests updated since the last run. `since` never reports deletions, so a weekly full listing removes deleted or transferred items. An edited issue or comment rebuilds only its own file under `github/` and its own records.

**Trigger.** A "Sync now" button in the app plus one nightly scheduled run. No webhooks.

**Embeddings.** Every chunk carries a hash of its text. Each arm's indexing step embeds only chunks whose text hash and embedding model are not yet in its vector cache, and deletes vectors of chunks that are gone. Tree-sitter cuts the unchanged functions of a changed file into the same text as before, so their vectors are reused. Line ranges and commit SHAs are metadata outside the embedded text, so shifted lines are updated without embedding again. The Bedrock arm receives only added, changed and deleted chunks, by document ID. A full re-embedding happens only when the embedding model changes. Even a full re-chunk after rewritten history reuses the vector of every chunk whose text did not change.

## Considered options

- Rebuild everything on every sync. Rejected for the call counts above.
- GitHub webhooks. Rejected because they need an endpoint GitHub can reach, which a single-user V1 does not have (ADR-0015).

## Consequences

- This settles re-syncing for GitHub repositories only. Re-syncing plain folders and project artifacts stays with the folder and document ingestion review.
