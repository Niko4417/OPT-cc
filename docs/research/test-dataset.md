# Public test dataset with generated workshop results

Research for [issue #8](https://github.com/Niko4417/OPT-cc/issues/8), part of the map in [issue #1](https://github.com/Niko4417/OPT-cc/issues/1). Checked on 2026-11-17 against the GitHub REST API (`gh api repos/...`, `gh api search/issues`, recursive git trees). All counts below come from those calls on that date unless a link says otherwise.

## Question

Which public GitHub repository is a realistic stand-in for a modernization project: a legacy system being modernized, with code, documentation, decision records, issues and pull requests, under a license that allows this use? How can workshop results for it be generated synthetically in German and English so that they reference real decisions in the repo?

## Answer

Use [`thunderbird/thunderbird-android`](https://github.com/thunderbird/thunderbird-android), pinned at commit [`0fd47ca`](https://github.com/thunderbird/thunderbird-android/commit/0fd47ca6c1346c21d4d18a8abfb5e6b17b82e212) (2026-11-17). It is the only candidate where the legacy code, the target architecture, the decision records and an active issue tracker all sit in one repository:

- The repo describes itself as "transitioning from a monolithic architecture to a modular one" ([legacy-module-integration.md](https://github.com/thunderbird/thunderbird-android/blob/main/docs/architecture/legacy-module-integration.md)). The `legacy/` folder holds 1,412 files and its README says the code there "will be replaced with new features and improvements and will be removed" ([legacy/README.md](https://github.com/thunderbird/thunderbird-android/blob/main/legacy/README.md)).
- ADR 0001 decides to rewrite the Java codebase in Kotlin ([ADR 0001](https://github.com/thunderbird/thunderbird-android/blob/main/docs/engineering/adr/0001-switch-from-java-to-kotlin.md)). The tree still has 287 Java files next to 3,509 Kotlin files, so the migration is visibly in progress.
- RFC 0007 replaces "the legacy per-account mail stores" with one Room 3 database and defines a migration and cutover ([RFC 0007](https://github.com/thunderbird/thunderbird-android/blob/main/docs/engineering/rfcs/0007-global-database.md)), with a table-by-table legacy inventory ([legacy-table-inventory.md](https://github.com/thunderbird/thunderbird-android/blob/main/docs/engineering/technical-designs/0003-global-database/legacy-table-inventory.md)).
- The license is Apache-2.0.

Size at the pinned commit: 8,601 files and 74.9 MB in the tree, of which 3,796 Kotlin and Java files (12.4 MB), 128 Markdown files (0.9 MB) and 220 images (28.2 MB). There are 10 ADRs, 9 RFCs and 3 technical designs, 6,301 issues and 5,502 pull requests. For the first evaluation, index the code and docs plus the issues and pull requests created since 2024-01-01: 2,080 issues and 1,915 pull requests without Dependabot.

Thunderbird's decision records have one gap: none is superseded or rejected. Add synthetic decision records that supersede each other in the companion repository (section "Generating workshop results"), or add [`ministryofjustice/modernisation-platform`](https://github.com/ministryofjustice/modernisation-platform) as a second test project for status handling.

## Selection criteria

From the question and the map's notes:

1. A legacy system is modernized, and the repo says so.
2. Decision records are Markdown files in the repo, with status fields.
3. Issues and pull requests are on GitHub, not in Jira.
4. The license allows internal commercial use as test data.
5. The size fits one Mac with 64 GB RAM and keeps re-indexing fast.
6. The project is active, so the issues and pull requests reflect current work.

## Ranked candidates

| Rank | Repository | License | Modernization story | Decision records | Issues / PRs | Tree size | Fit |
|---|---|---|---|---|---|---|---|
| 1 | [thunderbird/thunderbird-android](https://github.com/thunderbird/thunderbird-android) | Apache-2.0 | K-9 Mail monolith to modules, Java to Kotlin, legacy mail stores to one database | 10 ADRs + 9 RFCs + 3 technical designs, linked to issues and PRs | 6,301 / 5,502; 825 PRs merged in the last 12 months | 8,601 files, 74.9 MB | Best overall |
| 2 | [ministryofjustice/modernisation-platform](https://github.com/ministryofjustice/modernisation-platform) | MIT | Hosting platform for migrating legacy MoJ applications | 45 ADRs: 35 accepted, 6 rejected, 2 superseded, 2 proposed | 2,645 / 11,691; 1,656 merged in 12 months | 1,995 files, 16.2 MB | Best status mix, but no legacy code |
| 3 | [alphagov/govuk-infrastructure](https://github.com/alphagov/govuk-infrastructure) | MIT | GOV.UK replatforming from `govuk-aws` to EKS | 25 ADRs: 24 accepted, 1 deprecated | 794 / 4,275; 1,403 merged in 12 months | 891 files, 43.6 MB | Small and clean, target state only |
| 4 | [ministryofjustice/cloud-platform](https://github.com/ministryofjustice/cloud-platform) | MIT | Kubernetes hosting platform, migration of clusters to EKS | 54 ADR files, 45 with a status: 34 accepted, 6 proposed, 2 rejected, 2 incomplete, 1 superseded | 6,893 / 1,698; 116 merged in 12 months | 300 files, 5.6 MB | Many issues, almost no code |
| 5 | [CMSgov/beneficiary-fhir-data](https://github.com/CMSgov/beneficiary-fhir-data) | CC0-1.0 | Medicare claims API, JBoss replacement, monorepo | 28 RFCs in status folders: 14 implemented, 3 approved, 4 deferred, 4 rejected, 3 BFDv2 | 5 / 3,361 | 5,351 tree entries, 187 MB repo | No GitHub issues |

### 1. thunderbird/thunderbird-android

Thunderbird for Android "is based on K-9 Mail" ([README](https://github.com/thunderbird/thunderbird-android/blob/main/README.md)). The GitHub repository was created on 2011-02-04. Languages by bytes: Kotlin 11.0 MB, Java 2.0 MB ([languages API](https://api.github.com/repos/thunderbird/thunderbird-android/languages)).

Decision records live in `docs/engineering/`:

- 10 ADRs in [`docs/engineering/adr`](https://github.com/thunderbird/thunderbird-android/tree/main/docs/engineering/adr). ADRs 0001 to 0009 have status "Accepted". [ADR 0010](https://github.com/thunderbird/thunderbird-android/blob/main/docs/engineering/adr/0010-adopt-project-wide-repository-pattern.md) (repository pattern) has status "Proposed".
- 9 RFCs in [`docs/engineering/rfcs`](https://github.com/thunderbird/thunderbird-android/tree/main/docs/engineering/rfcs), for example [RFC 0004](https://github.com/thunderbird/thunderbird-android/blob/main/docs/engineering/rfcs/0004-feature-flag-new-architecture.md) (declarative feature flag catalog, "Accepted") and [RFC 0009](https://github.com/thunderbird/thunderbird-android/blob/main/docs/engineering/rfcs/0009-uuidv7-identifier-migration.md) (UUIDv7 identifier migration).
- 3 technical designs in [`docs/engineering/technical-designs`](https://github.com/thunderbird/thunderbird-android/tree/main/docs/engineering/technical-designs).
- 13 architecture documents in [`docs/architecture`](https://github.com/thunderbird/thunderbird-android/tree/main/docs/architecture), including `legacy-module-integration.md` and `database.md`.

The records link to issues and pull requests, which gives ground truth for cross-source questions. ADR 0001 links [PR #7221](https://github.com/thunderbird/thunderbird-android/pull/7221) (merged 2023-10-06). [ADR 0009](https://github.com/thunderbird/thunderbird-android/blob/main/docs/engineering/adr/0009-api-internal-split.md) links issue #10197 and PR #10198. RFC 0007 links issue [#11104](https://github.com/thunderbird/thunderbird-android/issues/11104) and names Room 3 as "selected by the completed database spike" in issue [#11195](https://github.com/thunderbird/thunderbird-android/issues/11195).

The issue tracker carries engineering work, not only bug reports. Recent examples are #11929 "Create RFC for message list re-architecture" and #11928 "Assess state of existing compose-based message list" ([issues API](https://api.github.com/repos/thunderbird/thunderbird-android/issues?state=all&sort=created)). A sample of the 80 non-bot issues and pull requests updated in the last 12 months had bodies of 1,425 characters on average and 1 comment each. The 2024-onward subset of about 4,000 items is therefore about 5.7 MB of body text before comments.

German content: the tree has 37 files under `values-de/` (German UI strings), which gives German terms on the code side too.

Weaknesses:

- No decision record is superseded or rejected, and the ADRs carry no date line. The git history has the dates.
- It is a mobile app, not an enterprise business system. The modernization patterns still match client work: strangling a monolith into modules, a language migration, a data migration with cutover and fallback.
- Issues contain real GitHub logins and user bug reports. These are personal data under GDPR. Keep the indexed copy on the consultant's machine and do not ship it in shared demo exports without pseudonymization.

### 2. ministryofjustice/modernisation-platform

The README says the team "provides a hosting platform for Ministry of Justice applications which cannot be hosted on the Cloud Platform" ([README](https://github.com/ministryofjustice/modernisation-platform/blob/main/README.md)). [ADR 0028](https://github.com/ministryofjustice/modernisation-platform/blob/main/architecture-decision-record/0028-expand-the-scope-of-the-platform.md) (2023-11-14) says the platform was "originally developed with the intention that it would only be used for migrating legacy applications". This repo has the richest status mix of all candidates: [ADR 0004](https://github.com/ministryofjustice/modernisation-platform/blob/main/architecture-decision-record/0004-use-bash-node-python-as-core-languages.md) is superseded by ADR 0019, ADR 0025 is superseded, six ADRs are rejected and two are proposed. Each ADR has a `Date:` line. The code is Terraform (1,212 `.tf` files), and the migrated legacy applications live in other repositories. Most of the 11,691 pull requests are small environment changes, many from bots.

### 3. alphagov/govuk-infrastructure

The repository tags its resources with `project = "replatforming"` and lists `govuk-aws` as the previous repository ([tagging guide](https://github.com/alphagov/govuk-infrastructure/blob/main/terraform/docs/tagging-guide.md)). Its 25 ADRs in [`docs/architecture/decisions`](https://github.com/alphagov/govuk-infrastructure/tree/main/docs/architecture/decisions) have `Date:` lines and cover the move to EKS, Helm, Concourse and the replacement of Terraform Cloud ([ADR 0014](https://github.com/alphagov/govuk-infrastructure/blob/main/docs/architecture/decisions/0014-replace-terraform-cloud.md)). Only [ADR 0007](https://github.com/alphagov/govuk-infrastructure/blob/main/docs/architecture/decisions/0007-use-fluentbit-elasticsearch-and-kibana-for-application-logs.md) has a status other than "Accepted": "Deprecated (no superseding ADR)". The repository description says its issues are "for all of GOV.UK Platform Engineering". It contains only the target state. The legacy side lives in other repositories.

### 4. ministryofjustice/cloud-platform

This is mostly a documentation and issue-tracking repository: 300 files, 70 of them Markdown, no Terraform. It has the most decision records (54 files in [`architecture-decision-record`](https://github.com/ministryofjustice/cloud-platform/tree/main/architecture-decision-record) and its `cp30` subfolder) and 6,893 issues. ADR 042 is named "use-modernisation-platform-accounts", which links it to candidate 2. With almost no code, it cannot test code questions.

### 5. CMSgov/beneficiary-fhir-data

The RFC process sorts records into folders by status (`approved`, `deferred`, `implemented`, `rejected`). [RFC 0000](https://github.com/CMSgov/beneficiary-fhir-data/blob/master/docs/rfcs/implemented/0000-jboss-replacement.md) replaces JBoss, a Java EE application server, which is a classic modernization step. The license is CC0-1.0. The repository has only 5 issues because the team tracks work elsewhere, so it fails criterion 3.

### Excluded

- [backstage/backstage](https://github.com/backstage/backstage), suggested in the background report: 15 ADRs, but it is not a legacy modernization and the repository is 8.5 GB.
- [digitalservicebund/ris-backend-service](https://github.com/digitalservicebund/ris-backend-service): a German federal project with 25 ADRs, two marked "[Superseded]" in their file names, but archived, issues disabled and GPL-3.0. The related [ris-search](https://github.com/digitalservicebund/ris-search) has 26 ADRs but only 4 issues in total.
- [JabRef/jabref](https://github.com/JabRef/jabref): 85 MADR decision records, MIT, 5,290 issues. The records cover small design choices, not a migration, and the repository is 288 MB.
- [shopware/shopware](https://github.com/shopware/shopware): 181 ADR files, 7 of them in `adr/_superseded`, MIT. 35,520 tree entries and a 577 MB repository are too large for fast evaluation loops.
- [aws-samples/aws-mainframe-modernization-carddemo](https://github.com/aws-samples/aws-mainframe-modernization-carddemo): real COBOL legacy code under Apache-2.0, but no decision records, 29 issues and no merged pull request in the last 12 months.
- [dotnet-architecture/eShopModernizing](https://github.com/dotnet-architecture/eShopModernizing): archived since 2023, no decision records.
- Apache Kafka with KIPs: the KIPs live on Confluence, which the map puts out of scope.

## Existing datasets

No public dataset combines code, decision records and meeting records of one project. The closest are:

- [SWE-QA](https://github.com/peng-weihan/SWE-QA-Bench) (Apache-2.0): repository-level code question answering with "question-answer pairs tied to pinned commits" over 15 Python projects such as Django and Flask ([README](https://github.com/peng-weihan/SWE-QA-Bench/blob/main/README.md)). Its question format is a good template for the code-location questions.
- [RepoQA](https://github.com/evalplus/repoqa) (Apache-2.0): long-context code understanding, last pushed 2024-11-01.
- [CodeRAG-Bench](https://github.com/code-rag-bench/code-rag-bench): retrieval for code generation, no license in the GitHub API, last pushed 2024-11-15.
- [QMSum](https://github.com/Yale-LILY/QMSum) (MIT): "1,808 query-summary pairs over 232 meetings" in English ([README](https://github.com/Yale-LILY/QMSum/blob/main/README.md)). Its query and topic-segment format is a useful pattern for questions over transcripts.

None covers decision records, German text or the link between a meeting and a decision in the code repository. That is why the workshop results must be generated.

## Generating workshop results

### Where they live

Create a separate GitHub repository, for example `opt-cc-testdata`, next to the pinned Thunderbird commit. The project memory then has two GitHub sources: `thunderbird/thunderbird-android` (files, issues, pull requests, read only) and the companion repository with Markdown files:

```
workshops/
  2026-03-12-kickoff-de.md
  2026-03-12-kickoff-transcript-de.md
  2026-04-02-global-database-en.md
  ...
decisions/
  WD-001-pilot-global-database-on-beta.md
  WD-002-...md
gold/
  questions.yaml
manifest.yaml
```

`WD-` records are synthetic decision records made in workshops. They fill the status gap: WD-004 supersedes WD-001, WD-006 is rejected, and so on. Never open issues or pull requests on the upstream repository.

### Story

Frame the workshops as a fictional consulting engagement that supports the Thunderbird team. Generate about 12 workshops over 8 months (March to October 2026, matching the dates of the real RFCs), half German and half English, for example:

| Workshop | Language | Real anchors |
|---|---|---|
| Kick-off and legacy inventory | de | `legacy/README.md`, `legacy-module-integration.md`, ADR 0001 |
| Module structure review | en | ADR 0007, ADR 0009, issue #10197 |
| Global database options | de | RFC 0007, issue #11195 (Room 3 spike) |
| Feature flag catalog | en | RFC 0004, issue #11222 |
| Repository pattern | de | ADR 0010 (proposed), PR #11452 |
| Identifier migration | en | RFC 0009 |
| Message list re-architecture | de | issues #11928, #11929 |
| Retrospective on the Kotlin migration | en | ADR 0001, PR #7221 |

Each workshop produces minutes. Every third workshop also produces a transcript with speaker turns (about 3,000 to 6,000 words), because transcripts are noisier and test retrieval differently.

### Pipeline

1. **Fact sheet.** A script reads the pinned commit through the GitHub API and writes `manifest.yaml`: for each anchor its type, number or path, title, status, date and a 3 to 5 line summary taken from the source text. The language model never invents an anchor. It may only cite entries from the manifest.
2. **Plan.** A plan file per workshop lists date, language, participants (fictional), agenda, the anchors to cite, and the planted facts: 2 to 4 statements that exist only in this workshop (for example "the team pilots the global database migration on the beta channel first"), plus any `WD-` decision it creates or supersedes.
3. **Write.** The answer model (`Qwen3.6-35B-A3B` through LiteLLM, on-device) writes the minutes or transcript from the plan and the fact sheet. Fix the temperature and seed and store the prompt next to the output.
4. **Check.** A script checks that every link and issue number in the output appears in the manifest and resolves at the pinned commit. The checker model (`gemma-4-31b-it`) then compares each sentence that cites an anchor against the anchor's summary and flags contradictions. A flagged file is regenerated, not hand-edited.
5. **Gold questions.** Generate questions from the plans, not from the outputs, so the expected answer and its citations are known before generation: 50 to 100 questions in the categories of the background report (fact lookup, decision rationale, current status, code location, cross-source). Write half in German, half in English, and include cross-language pairs (German question, English ADR as the answer source).
6. **Review.** A person reads every gold question and a sample of 3 workshops before the dataset is tagged.

### Marking as synthetic

- Front matter in every generated file:

  ```yaml
  synthetic: true
  generator: opt-cc-testdata/scripts/generate.py
  model: mlx-community/Qwen3.6-35B-A3B-4bit
  checker: mlx-community/gemma-4-31b-it-4bit
  seed: 42
  upstream: thunderbird/thunderbird-android@0fd47ca6c1346c21d4d18a8abfb5e6b17b82e212
  ```

- A first line in the body, in the file's language: "SYNTHETISCH: Fiktiver Workshop für Testzwecke. Kein Dokument des Thunderbird-Projekts oder von MZLA." or "SYNTHETIC: Fictional workshop for testing. Not a record of the Thunderbird project or MZLA."
- Participants come from a fixed list of invented names with `@example.com` addresses ([RFC 2606](https://www.rfc-editor.org/rfc/rfc2606) reserves `example.com`). Never use names or logins of real Thunderbird contributors.
- The companion repository README states that all files under `workshops/` and `decisions/` are synthetic and that the upstream project did not take part.
- Keep the companion repository private. Fabricated minutes about a real open-source project could mislead readers outside the team.

### Planted test data

Each planted item has a known location, so tests can check it:

- One fictional person mentions a sensitive personal fact in one transcript (for example a sick leave). The erasure test deletes this transcript and checks that no answer or citation still contains the fact.
- Two `WD-` decisions supersede earlier ones. The status test asks for the current decision and expects the newer record.
- One planted fact contradicts an upstream record on purpose, labeled in the plan file. The test checks that the answer reports the conflict and cites both sources.

## Open points for other tickets

- Whether one project memory may have two GitHub repositories as sources needs confirmation in the connector ticket.
- The issue and pull request subset (since 2024-01-01, without bots) is a first cut. The evaluation ticket should check how indexing time scales with the full 11,803 items.
- If status handling needs real superseded records, add `ministryofjustice/modernisation-platform` as a second test project rather than replacing Thunderbird.

## Sources

Repository metadata, trees, languages and counts (GitHub REST API, 2026-11-17):

- https://api.github.com/repos/thunderbird/thunderbird-android
- https://api.github.com/repos/ministryofjustice/modernisation-platform
- https://api.github.com/repos/alphagov/govuk-infrastructure
- https://api.github.com/repos/ministryofjustice/cloud-platform
- https://api.github.com/repos/CMSgov/beneficiary-fhir-data
- https://api.github.com/repos/backstage/backstage
- https://api.github.com/repos/digitalservicebund/ris-backend-service
- https://api.github.com/repos/digitalservicebund/ris-search
- https://api.github.com/repos/JabRef/jabref
- https://api.github.com/repos/shopware/shopware
- https://api.github.com/repos/aws-samples/aws-mainframe-modernization-carddemo
- https://api.github.com/repos/dotnet-architecture/eShopModernizing
- Issue and pull request counts: `https://api.github.com/search/issues?q=repo:<owner>/<repo>+is:issue` and `+is:pr`, with `+is:merged+merged:>=2025-11-17` for the 12-month figures.

Decision records and docs cited above are linked inline. Background: `docs/research/rag-stack-landscape.md` on branch `research/rag-stack-landscape`, section 6.
