---
status: accepted
date: 2026-11-18
decided-by: Niko4417
ticket: https://github.com/Niko4417/OPT-cc/issues/9
---

# Settle supersession from explicit links and send every other case to a human ruling

A newer record that contradicts an older one may well replace it, but the project memory must not invent that rule. It accepts a supersession only when a record states it and creation order agrees. Every other case becomes a decision conflict that a human rules on, with evidence and a recommended action.

- **Trusted edge.** One record states the supersession, either `supersedes: WD-001` on the new record or `superseded by WD-004` on the old one, and the superseding record was created later. The edge counts with no review. If the old record still says accepted, the edge carries a note about the stale status, not a conflict.
- **Decision conflict.** An older record claims to supersede a newer one, the links form a cycle, or two records supersede the same one.
- **Unlinked contradiction.** Ingestion compares each decision record with its 5 most similar decision records by embedding. The checker model, `gemma-4-31b-it-4bit`, decides whether the newer one contradicts or replaces the older one and quotes the evidence sentences from both. A hit never becomes an edge on its own. It becomes a decision conflict. For 30 records this is at most 150 checker calls, kept on-device by the model policy.
- **Ruling.** The app lists each decision conflict with both records, the evidence quotes, the conflict type and a recommended action: "B supersedes A", "both stay current (different scope)" or "not a conflict". The ruling is stored in the project memory, never in the source files, and travels in the project memory bundle. It expires when either record changes.
- **Answers.** When retrieval returns a decision record, the project memory adds the rest of its supersession chain in both directions plus its rulings and open decision conflicts. When it returns a workshop result, it adds the decision records that result lists. The prompt labels each record with its decision status. The answer leads with the current record, names what it replaced and cites both. With an open conflict, the answer shows both records with their statuses, says the conflict is open, cites both and picks neither.

## Considered options

- Take the union of both link directions and let any supersession edge override the old record's own status. Rejected because it decides cases that need a human.
- Let the newest date win. Rejected because dates are missing in some repositories and say nothing about whether two decisions cover the same scope.
