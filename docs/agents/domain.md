# Domain Docs

How the engineering skills should consume this repo's domain documentation when exploring the codebase.

## Before exploring, read these

- **`GLOSSARY.md`** at the repo root, or
- **`GLOSSARY-MAP.md`** at the repo root if it exists: it points at one `GLOSSARY.md` per context. Read each one relevant to the topic.
- **`docs/adr/`**: read ADRs that touch the area you're about to work in. In multi-context repos, also check `src/<context>/docs/adr/` for context-scoped decisions.

If any of these files don't exist, **proceed silently**. Don't flag their absence; don't suggest creating them upfront. The `/domain-modeling` skill (reached via `/grill-with-docs` and `/improve-codebase-architecture`) creates them lazily when terms or decisions actually get resolved.

## File structure

Single-context repo (most repos):

```
/
├── GLOSSARY.md
├── docs/adr/
│   ├── 0001-event-sourced-orders.md
│   └── 0002-postgres-for-write-model.md
└── src/
```

Multi-context repo (presence of `GLOSSARY-MAP.md` at the root):

```
/
├── GLOSSARY-MAP.md
├── docs/adr/                          ← system-wide decisions
└── src/
    ├── ordering/
    │   ├── GLOSSARY.md
    │   └── docs/adr/                  ← context-specific decisions
    └── billing/
        ├── GLOSSARY.md
        └── docs/adr/
```

## Use the glossary's vocabulary

When your output names a domain concept (in an issue title, a refactor proposal, a hypothesis, a test name), use the term as defined in `GLOSSARY.md`. Don't drift to synonyms the glossary explicitly avoids.

If the concept you need isn't in the glossary yet, that's a signal: either you're inventing language the project doesn't use (reconsider) or there's a real gap (note it for `/domain-modeling`).

## Flag ADR conflicts

If your output contradicts an existing ADR, surface it explicitly rather than silently overriding:

> _Contradicts ADR-0007 (event-sourced orders), but worth reopening because…_

## Record every owner decision as an ADR

This repo records more ADRs than the domain-modeling skill's default. Write one for every decision the owner makes final, whether or not it passes the three-criteria test. A research finding becomes an ADR only once the owner adopts it, usually in a grilling ticket.

Write the ADR in the same session the decision is made, before closing the ticket. Link the ADR from the ticket's resolution comment, and link the ticket from the ADR.

Numbering: scan `docs/adr/` for the highest number and add one: `0001-slug.md`, `0002-slug.md`.

Template:

```md
---
status: accepted
date: YYYY-MM-DD
decided-by: <owner's GitHub login>
ticket: <issue URL>
---

# <Decision as a short statement>

<1 to 3 sentences: the context, what was decided, and why.>

## Considered options

<Only when real alternatives existed: each one and why it lost.>

## Consequences

<Only when a downstream effect is not obvious.>
```

Status is one of `proposed`, `accepted`, `deprecated` or `superseded by ADR-NNNN`. To change a decision, write a new ADR and set the old one's status to `superseded by ADR-NNNN`. Edit an accepted ADR's text only to fix typos or links.
