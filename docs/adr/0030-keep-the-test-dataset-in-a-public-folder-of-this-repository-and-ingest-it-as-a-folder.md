---
status: accepted
date: 2026-11-21
decided-by: Niko4417
ticket: https://github.com/Niko4417/OPT-cc/issues/23
---

# Keep the test dataset in a public folder of this repository and ingest it as a folder

The test dataset research proposed a private companion repository on GitHub that the project memory reads as a second GitHub source. Instead, the dataset lives under `testdata/<project>/` in this repository, which is public, and enters the project memory as a plain folder plus project artifacts.

**What is ingested.** Each `testdata/<project>/` folder has a `sources/` part that the project memory ingests. It holds only what a client would hand over. The workshop plans, prompts, the manifest and the gold questions sit beside it and are never ingested, because the plans list every planted fact and the gold questions hold the expected answers. A check fails the run if an ingested path lies outside `sources/`. The project memory never reads the rest of this repository.

**Marking.** Every synthetic file is public, so marking is required:

- Front matter with `synthetic: true`, generator, model, seed and upstream commit.
- A first-line notice in the file's language that the document is fictional.
- Invented participants with `example.com` addresses, and no name or login of a real contributor.
- A README in the folder stating that the upstream project took no part.

A format without front matter carries the notice in the document itself and in the manifest. A test fails the build when a synthetic file lacks its marking. Real documents that are copied unmodified (ADR-0034) carry no notice, and the manifest records their origin and license.

**The notice is ingested as it is.** No test-only code strips it, so the app stays free of special cases for test data.

**Versions.** One manifest names the upstream commits (Thunderbird `0fd47ca`, Shopware `470e9fe`, LedgerSMB `c32ed61`) and the checksums of everything stored outside the repository. The dataset has a version tag such as `testdata-v1`. Every bake-off and nightly result names the tag. A changed dataset or commit gets a new tag and new baselines.

**Large files.** Office files sit in the repository. Recordings are release files attached to the version tag, because in git history they would stay forever.

## Considered options

- **A private companion repository read as a second GitHub source.** Rejected. It needs one project memory fed by several GitHub repositories, which is not specified. Clients hand over workshop results as files, not as a repository. And the folder walk and the project artifact path would go untested.
- **A private repository for storage only, read as a folder.** It would keep fabricated minutes about a real project out of public view. The owner chose the public folder, which needs no access token in CI and keeps everything in one repository.
- **Stripping the synthetic notice before ingestion.** Rejected as test-only code in the app.

## Consequences

- Fabricated minutes and decisions about a real open-source project are public. The marking above is what keeps them from being read as real.
- The gold questions and their expected answers are public too.
- The answer model sees the notice. That may shift absolute scores, for example if a model hedges on text labelled fictional. The notice is the same for every arm, so comparisons between arms stay fair.
- A citation into the folder is a path plus a content fingerprint, not a commit-pinned permalink as ADR-0019 gives for GitHub files. That is the same for any client folder.
