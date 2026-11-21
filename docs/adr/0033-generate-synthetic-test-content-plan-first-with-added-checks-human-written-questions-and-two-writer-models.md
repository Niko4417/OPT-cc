---
status: accepted
date: 2026-11-21
decided-by: Niko4417
ticket: https://github.com/Niko4417/OPT-cc/issues/23
---

# Generate synthetic test content plan-first, with added checks, human-written questions and two writer models

The test dataset research designed a six-step generation pipeline and cited no evidence for it. The owner did not accept it unproven. The [synthetic test data research](https://github.com/Niko4417/OPT-cc/blob/research/synthetic-test-data/docs/research/synthetic-test-data.md) found no study that proves one pipeline best. It found that recent benchmarks plan before they write, and it found measurements of where synthetic test data goes wrong. The pipeline keeps its plan-first core and changes what those measurements contradict.

1. **Fact sheet.** A script reads the pinned commit and writes a manifest of real anchors. The writer model may cite only those.
2. **Plan.** A plan per document lists the anchors and the planted facts. It also lists distractor statements that differ from a planted fact in one detail, and every document that states each fact, so gold citations are complete.
3. **Write.** Minutes are written in one pass. Transcripts are written scene by scene with one call per speaker, where each speaker sees only its own brief. The generated files are committed and tagged, so the dataset does not depend on reproducing a run.
4. **Check.** A script checks that every reference resolves, and a checker model compares each citing sentence with its anchor. Two checks are added. Every planted fact and anchor must appear in the text, with no distractor in its place. And the answer model, given only the gold passages of a question, must return the key facts. A question that fails is fixed or dropped.
5. **Gold questions.** Questions come from the plans, not from the generated text. They have a fixed count per question type, 10% to 20% questions that must be refused, and varied phrasing that includes short keyword questions and wording unlike the source. A filter rejects a question with ROUGE-L F1 above 0.35 against its gold passage or with a shared 4-gram. Every question also runs without retrieval, and one the model answers from memory is marked and reported apart. Each project also gets questions written by people who know the workshop agenda and have not read the generated documents.
6. **Review.** A person reads every gold question and every synthetic document that a gold question cites. Other synthetic documents are sampled. The review logs how many items were rejected or changed. Small hand edits of generated files are allowed and logged in the manifest.
7. **Reporting.** Results are reported apart for synthetic and real sources, for generated and human-written questions, and per language. Scores on generated questions count as an upper bound.

**Models.** Any model on an EU route that the model policy allows may write and check, and each file's front matter names both. Shopware's content comes from a different writer model family and reworded templates than Thunderbird's. The writer is never the judge model. The recorded workshops get their transcripts from the app's own transcription of synthesized speech, so they carry real recognition errors.

The evidence behind the changes:

- Systems ranked on synthetic questions kept nearly the same order on human questions in three studies (Kendall tau 0.82 to 0.93). Score levels did not carry over. In Kucia and Gawlik (CIKM 2026), Hit@5 fell from 0.82 to 0.90 on synthetic questions to 0.53 to 0.55 on authentic ones, and the leading setup changed.
- When only the passage a question was written from counted as correct, the agreement fell to tau 0.157 (Rahmani et al., SIGIR 2024).
- With BM25, questions worded like the document scored a MAP of 0.557 and differently worded ones 0.201 (Fensore et al., a 2026 preprint).
- Single-pass transcripts were rated 3 of 5 for naturalness against 4.5 for one call per speaker (FAME).
- Human review changed a lot in published benchmarks: nearly 15% of dialog turns in LoCoMo and about 70% of sessions in LongMemEval.

Three parts are judgment, not tested results: the second writer model family for Shopware, which rests on an analogy to Geva et al., where accuracy fell by up to 23 points on text from annotators a system had not seen, the logged hand edits, and the speech-like transcripts.

## Considered options

- **The six steps as the research wrote them.** Rejected in three details: single-pass transcripts, reading a sample of 3 workshop results, and one gold citation per fact.
- **Model-phrased questions only.** Rejected for the gap between synthetic and authentic questions above.
- **A person reads every synthetic document.** Feasible for about 12 workshop results per project, not for the domain knowledge part of ADR-0034. So every document a gold question cites is read, and the rest is sampled.
- **One writer model for both projects.** Rejected because tuning on Thunderbird could fit the habits of that writer, and Shopware would then confirm a result it should test.

## Consequences

- Nothing was found on synthetic German workshop minutes, on retrieval over synthetic against real meeting transcripts, or on tuning on one synthetic set and checking on another from the same pipeline. This pipeline follows published practice. It is not proven.
- How many human-written questions each project gets, and who writes them, is decided in the ticket on the gold question set. The research proposes at least 50.
- With about 20 questions per question type, results per type describe behavior and cannot rank stacks.
- Which models write and check is left to the models review.
