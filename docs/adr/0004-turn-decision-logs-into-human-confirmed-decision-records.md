---
status: accepted
date: 2026-11-18
decided-by: Niko4417
ticket: https://github.com/Niko4417/OPT-cc/issues/9
---

# Turn decision logs in Word and Excel into decision records that a human confirms

Clients often keep decision logs in a Word table or an Excel sheet. The project memory has no dedicated parser for them. Docling converts them to Markdown, as for any other document, and the entries are then placed in the decision register next to the Markdown decision records.

- A table whose column headers map to the fields (ID, decision, status, date, and supersedes or replaced-by) is split row by row without a model.
- For any other layout, the answer model proposes candidate decision records, each citing its page, row or heading.
- A human confirms every record from a converted document in the review list before it enters the decision register.
- The date comes from the row or section itself (a date column, or a date in the heading), then the document's own metadata (`dcterms:created` in Word, Excel and PowerPoint files, `CreationDate` in PDF), then a date the consultant enters at import. The file system date is never used, because copying a file changes it. The decision register shows where each date came from.

## Considered options

- Ingest decision logs as ordinary documents with no status handling. Rejected because the decision register would then miss decisions that clients keep only in these files.
