---
status: accepted
date: 2026-11-21
decided-by: Niko4417
ticket: https://github.com/Niko4417/OPT-cc/issues/23
---

# Freeze issues and pull requests in a private snapshot and keep sync off for scored project memories

A commit pins the code of a test project, but nothing pins its issues and pull requests. People edit, comment and delete after the pin date, so two runs a month apart would read different text. The issues also hold real GitHub logins and user bug reports.

**Ingested unchanged.** The upstream issues and pull requests are ingested as they are, on AWS in Frankfurt. The data is public and every route processes in the EU (ADR-0014). A project memory bundle of a test project is never shared outside the consultancy.

**One snapshot per project.** When the dataset is tagged (ADR-0030), the ingestion job runs once and its output is kept: the Markdown file per issue and pull request that ADR-0018 defines. The snapshot is stored in a private S3 bucket of the installation's AWS account, with its checksum in the manifest. It never goes into this repository. Every evaluation runs on the snapshot. Text that may be fetched but not republished (ADR-0034) is kept there too.

**Sync is off for scored project memories.** The scheduled sync of ADR-0020 is switched off for the Thunderbird and Shopware project memories from the moment the dataset is tagged. "Sync now" on a scored project memory counts as a dataset change and needs a new version tag.

**A separate sync test.** Sync is tested apart from the scored runs, as a pass or fail check without a language model.

- Code. The dataset names a second, later Thunderbird commit. The test ingests at the first commit, syncs to the second and compares the result with a fresh ingestion at the second. It also counts how many chunks were embedded again, because ADR-0020 promises that unchanged chunks are reused.
- Issues and pull requests. On a separate, unscored project memory the test syncs against live GitHub, then lists everything in full and reports every difference.

## Considered options

- **Keep the indexed copy on the consultant's machine**, as the test dataset research asked. Overtaken by ADR-0013, which runs V1 on AWS.
- **Replace logins with pseudonyms.** Rejected because it would break the `html_url` citations that ADR-0019 requires.
- **Store the snapshot in this repository.** Rejected because the repository is public and the snapshot holds personal data.
- **Let the nightly sync run during a bake-off.** Rejected because scores from different days would come from different content.

## Consequences

- Sync must be switchable per project memory. ADR-0020 does not say so. The product side is decided in the ticket on sources.
- A new snapshot only comes with a new dataset version and new baselines.
