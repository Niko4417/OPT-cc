---
status: accepted
date: 2026-11-21
decided-by: Niko4417
ticket: https://github.com/Niko4417/OPT-cc/issues/23
---

# Use Thunderbird as the small test project and Shopware as the large one, measured in three runs

The test dataset rests on two public repositories. [`thunderbird/thunderbird-android`](https://github.com/thunderbird/thunderbird-android), pinned at commit `0fd47ca` (Apache-2.0, about 230,000 production lines of Kotlin and Java), is the small test project. Every evaluation loop and all tuning run on it. [`shopware/shopware`](https://github.com/shopware/shopware), pinned at commit `470e9fe` (MIT, 1.16 million production lines), is the large test project. Nothing is tuned on it, so it checks whether a result holds on a project the tuning never saw.

Two different projects differ in language, structure and writing style as well as in size. So size is measured inside one project, in three runs:

| Run | Project memory | What the comparison shows |
|---|---|---|
| A | Thunderbird | Baseline for every evaluation loop |
| B-small | Shopware, cut down to about Thunderbird's size | A against B-small shows the effect of the project |
| B-full | Shopware, complete | B-small against B-full shows the effect of size, on the same questions |

B-small keeps every file and item that a gold citation points to and fills up with whole modules and a sample of issues and pull requests. How the cut is made in detail is left to the bake-off.

Both projects are ingested by the same rules. Code and documentation come from the pinned commit. Issues and pull requests are those created since 2024-01-01, without accounts of type Bot and without translation service logins such as `weblate`. For Thunderbird that is 2,084 issues and 1,795 pull requests. For Shopware it is about 19,300 items before bots are removed. Both projects get the same source types, the same question mix and the same language split (ADR-0031).

## Considered options

- **[`ledgersmb/LedgerSMB`](https://github.com/ledgersmb/LedgerSMB) as the small test project.** It is an accounting system with 41 dated ADRs, one of them marked superseded and two reversed without a status change. It lost because it has too little issue and pull request text: 168 issues since 2024 against 2,084, a median pull request description of 13 characters against 190, and 1 of 41 ADRs that links an issue. Issues, pull requests and their links to decision records are where the project memory differs from a document search. Its decision records are used on their own (ADR-0029). The comparison is in an [input comment on the ticket](https://github.com/Niko4417/OPT-cc/issues/23#issuecomment-7051434062).
- **Shopware with questions on code and its real ADRs only.** Rejected because a lower score on Shopware could then come from its size or from the different question mix.
- **Thunderbird against Shopware as the measure of size.** Rejected because the two differ in everything else too.
- **A third test project.** `openemr/openemr` would have separated size from project, which B-small against B-full now does. `kubernetes/kubernetes` with its 686 KEPs needs several GitHub repositories per project memory and has 3.1 million lines. `ministryofjustice/modernisation-platform` has the richest real status mix and no legacy code. Each one would need its own human-reviewed gold questions. All three are rejected for V1.
- **The other small repositories of the survey.** None of the eight has 20 or more decision records with a status field.

## Consequences

- Shopware needs its own synthetic content and its own gold question set.
- No test project needs one project memory fed by several GitHub repositories.
- Thunderbird is a mail client, not a business system, and none of its decision records is superseded. Synthetic decision records (ADR-0031) and LedgerSMB's real ones (ADR-0029) cover that.
- Shopware's ADRs have no status field, its changelog points to Jira before 2024, and it has almost no German. German and decision status are tested through its synthetic content.
- A small project memory cannot stand in for a large one. In one study that grew a corpus around the same 500 questions, the ranking of methods flipped at about 7,000 to 8,500 documents ([domain documentation volume research](https://github.com/Niko4417/OPT-cc/blob/research/domain-documentation-volume/docs/research/domain-documentation-volume.md)).

Sources: [test dataset research](https://github.com/Niko4417/OPT-cc/blob/research/test-dataset/docs/research/test-dataset.md), [large legacy test repositories](https://github.com/Niko4417/OPT-cc/blob/research/large-legacy-test-repositories/docs/research/large-legacy-test-repositories.md).
