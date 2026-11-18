---
status: accepted
date: 2026-11-18
decided-by: Niko4417
ticket: https://github.com/Niko4417/OPT-cc/issues/9
---

# Show every decision in a decision register

V1 has a decision register view next to question answering. It lists every decision record in the project memory with its ID, title, decision status, date and date source, a source citation, its supersession chain and its open decision conflicts. It also shows the links the ingestion job already parses: the workshop result that produced a decision and the issues, pull requests and code it references.

## Considered options

- Supersession chains only. Rejected because the parsed links to workshops, issues and code cost nothing extra.
- Group decisions by topic or system area. Deferred until the gold questions show it is needed.
