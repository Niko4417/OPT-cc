---
status: accepted
date: 2026-11-18
decided-by: Niko4417
ticket: https://github.com/Niko4417/OPT-cc/issues/9
---

# Prescribe a Markdown format for workshop results and give each workshop decision its own record

Workshop results the team writes are Markdown with front matter `id`, `type: workshop-result`, `kind` (`minutes` or `transcript`), `date`, `language`, `title`, `participants` and `decisions` (the ids of the decision records the workshop produced). Synthetic test files add the `synthetic: true` block from the test dataset research to the same front matter.

Minutes have the sections agenda, discussion, decisions and action items. Every entry under decisions links to its decision record. Transcripts have one `**Name:**` line per speaker turn, with optional timestamps.

Only decision records take part in status handling and the decision register. A decision stated only in minutes is ordinary text that the answer model reads. The prescribed format therefore asks the writer to create a `WD-` decision record for each decision a workshop makes.

## Considered options

- Track decisions that exist only in minutes, including later minutes that reverse them, with an extraction model such as Graphiti. Deferred, as the [graph knowledge research](https://github.com/Niko4417/OPT-cc/blob/research/graph-knowledge/docs/research/graph-knowledge.md) recommended, until the gold questions show that metadata alone fails.
