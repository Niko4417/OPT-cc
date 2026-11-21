---
status: accepted
date: 2026-11-21
decided-by: Niko4417
ticket: https://github.com/Niko4417/OPT-cc/issues/23
---

# Check the decision register on LedgerSMB's real decision records

The 41 ADRs in `doc/adr` of [`ledgersmb/LedgerSMB`](https://github.com/ledgersmb/LedgerSMB), pinned at commit `c32ed61`, are read as a check of the decision register (ADR-0005), of status reading (ADR-0001) and of supersession handling (ADR-0003). Neither test project has superseded or stale decision records of its own, and synthetic records only imitate them. LedgerSMB's records were written by a project for itself:

- ADR 0011 is marked "Superseded by ADR 0105".
- ADR 0005 and ADR 0022 are reversed by ADR 0104 and still say "Accepted".
- The repository uses three words for one status: "Replaced by", "Deprecated" and "Superseded by".
- 21 of the 41 dates are vague, such as "Early in the 1.3 release cycle", and ADR 0112 links a file that does not exist.

The check expects the one stated supersession as a trusted edge and the reversed decisions as decision conflicts. A person confirms the list of expected results before it is used, because some of the stale cases rest on one reading of the texts.

The files are fetched at test time at the pinned commit. They are never copied into this repository, because LedgerSMB is under GPL-2.0-or-later. LedgerSMB gets no synthetic content, no answer questions and no scored project memory.

## Considered options

- **LedgerSMB as a full test project.** Rejected in ADR-0028 for its thin issue and pull request text.
- **`kubernetes/enhancements` with 686 KEPs, or `ministryofjustice/modernisation-platform` with 45 ADRs.** Both have a wider status mix. LedgerSMB gives real stale and reversed records at the cost of 41 files and needs no further repository support.
- **Synthetic decision records only.** Kept as well (ADR-0031), because answer questions need them inside the test projects. They cannot show how the reader copes with records nobody wrote for the test.

## Consequences

- The part that finds reversed decisions needs the checker model, so it cannot run in the checks without a language model. Where it runs is left to the quality gates review.

Source: [comparison of Thunderbird and LedgerSMB](https://github.com/Niko4417/OPT-cc/issues/23#issuecomment-7051434062).
