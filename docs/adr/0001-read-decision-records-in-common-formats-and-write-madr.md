---
status: accepted
date: 2026-11-18
decided-by: Niko4417
ticket: https://github.com/Niko4417/OPT-cc/issues/9
---

# Read decision records in common formats and write new ones in MADR 4.0

Client repositories arrive with decision records in whatever style they already use. Thunderbird puts the status in a bullet line and has no dates, while other repositories use `Date:` lines, status folders or MADR front matter. The project memory therefore reads several common styles and maps each record to the same fields. Records the consulting team writes itself follow one prescribed format.

What the reader does:

- A decision record is any Markdown record that states one decision and a decision status: ADRs, RFCs with a status, and workshop decisions. The record type is a field: `adr`, `rfc` or `workshop-decision`.
- Front matter `type` decides first. Without it, a file is a decision record only if it matches the project memory's path patterns and the reader finds a status. The default patterns are `**/adr/**`, `**/decisions/**`, `**/rfcs/**`, `**/architecture-decision-record*/**` and `**/architecture/decisions/**`. A matching file without a status stays an ordinary document and is listed in the ingestion report. No model classifies files.
- Statuses are mapped to proposed, accepted, rejected, deprecated or superseded through a per-project mapping table with English and German defaults (`approved` and `implemented` to accepted, `deferred` to proposed, `Angenommen` to accepted, `Abgelöst` to superseded, and status folder names likewise). An unmapped status is kept verbatim and shown as unknown.
- The date comes from front matter `date`, then a `Date:` line, then the git date of the commit that added the file. Dates order and display records. They never decide which record is current.
- Path patterns and the mapping table are project memory settings and travel in the project memory bundle.

The prescribed format is [MADR 4.0](https://adr.github.io/madr/) with its front matter (`status`, `date`, `decision-makers`, `consulted`, `informed`) plus five fields: `id` (for example `WD-004`), `type`, `supersedes` (a list of ids), `workshop` (the id of the workshop result that produced it) and `language`. Field names are English. The body uses MADR's sections in the record's own language. A writer who supersedes a record also sets the old record's status to `superseded by <id>`.

## Considered options

- Require one format in every client repository. Rejected because client repositories are read-only and already have their own styles.
- Classify decision records with a language model. Rejected because path patterns and front matter are exact and cost nothing.
