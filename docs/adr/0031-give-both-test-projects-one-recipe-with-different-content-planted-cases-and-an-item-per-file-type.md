---
status: accepted
date: 2026-11-21
decided-by: Niko4417
ticket: https://github.com/Niko4417/OPT-cc/issues/23
---

# Give both test projects one recipe with different content, planted cases and an item per file type

Both test projects (ADR-0028) get synthetic content from one recipe, so their results can be compared. The content itself differs per project, so the large project stays an independent check on what was tuned on the small one.

| Same for both projects | Different per project |
|---|---|
| Number of workshops and their type mix | Topics and anchors in the real repository |
| Language split and question mix | Planted facts and their wording |
| Types of planted cases | Invented participants and the engagement story |
| Pipeline version and review step | Seeds, writer model family and prompt templates (ADR-0033) |

The recipe has about 12 workshops over 8 months, half German and half English. Every third one also has a transcript of 3,000 to 6,000 words. Gold questions include pairs that cross languages. The runs B-small and B-full of ADR-0028 share identical content on purpose.

**Planted cases.** There are nine, counting the two superseding pairs separately. Each has a known location that a test can check.

1. A sensitive personal fact about a fictional person in one transcript, for the erasure test.
2. Two pairs of `WD-` decision records where one supersedes the other.
3. One planted fact that contradicts an upstream record, labelled in its plan.
4. One unlinked contradiction between two `WD-` records (ADR-0003).
5. One stale status: the old record still says accepted although a newer one states `supersedes` (ADR-0003).
6. One Word decision log whose table headers map to the fields (ADR-0004).
7. One Excel decision log with a free layout (ADR-0004).
8. One document that contains an instruction addressed to the assistant. The test expects the answer to ignore it. Both `thunderbird-android` and LedgerSMB already carry an `AGENTS.md` with such text.

**An item per file type.** The dataset holds at least one item of every source type V1 ingests, half of them German. Besides Markdown, audio and video, these are the items the [source file types research](https://github.com/Niko4417/OPT-cc/blob/research/source-file-types/docs/research/source-file-types.md) lists:

- One `.vtt` transcript with speaker names and one Teams `.docx` transcript.
- One `.eml` and one `.msg` message, each with a Word decision log attached.
- One `.doc`, one `.xls` and one `.ppt` file.
- One PowerPoint file with speaker notes and a hidden slide, and one Word file with comments and tracked changes.
- One `.xlsm` workbook with a macro, a formula and a hidden sheet.
- One Confluence HTML export as a zip archive and one Confluence Word export.
- One draw.io diagram and its PNG export.
- One OpenAPI file, one WSDL with an XSD, and one DDL file.
- One file V1 does not read and one password-protected Word file.

An item whose file type the ingestion reviews do not adopt stays in the dataset and tests the notice "stored and not read".

## Considered options

- **The same text for both projects.** Not possible, because every workshop is anchored to its project's own decision records, issues and code.
- **Synthetic content for Thunderbird only.** Rejected in ADR-0028.
- **Markdown files only, as the research planned.** Rejected because the document ingestion and transcription paths would go untested, and ADR-0017 scores the bake-off per source type.

## Consequences

- Which file types V1 reads is decided in the folder and document ingestion review and in the ticket on sources. The owner's intent, recorded there, is to read as many file types as possible.
- The erasure test deletes a transcript from the folder. It can run only once re-syncing of plain folders is specified.
- The synthetic workshop results rendered to PDF, Word and a scanned PDF can also measure parsing loss against their Markdown originals, which is run I9 of the [ingestion architecture research](https://github.com/Niko4417/OPT-cc/blob/research/ingestion-architecture/docs/research/ingestion-architecture.md).
- The number of gold questions is left to the ticket on the gold question set.
