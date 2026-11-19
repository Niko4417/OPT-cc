---
status: accepted
date: 2026-11-19
decided-by: Niko4417
ticket: https://github.com/Niko4417/OPT-cc/issues/41
---

# Pick the cheapest model per task whose quality is good enough

The installation is not limited to the Keiko lab pair. For each task (answering, checking, the evaluation judge, query planning and tool calling for agent arms, embedding, reranking and speech to text) it uses the cheapest model whose quality is good enough. A research ticket shortlists candidates on AWS from papers, independent benchmarks and price lists, counting idle cost for models billed by the hour, and a review ticket adopts them.

"Good enough" means: on the gold question set, the model is not significantly worse than the best shortlisted candidate for that task (its paired bootstrap interval overlaps the best one's), and it passes the hard gates of 0 fabricated citations and 0 off-policy model calls. Among those, the cheapest wins.

Within the bake-off, every arm uses the same models, so arms differ only in retrieval.

## Considered options

- Keep the lab pair (Qwen3.6-35B-A3B and Gemma 4 31B) everywhere. Rejected because Bedrock in Frankfurt has no Qwen3.6, and the owner wants cost per task to decide.
- Pick the best-scoring model per task regardless of price. Rejected because a cheaper model that is not measurably worse is the better proof of concept.

## Consequences

Models outside Bedrock's serverless catalog run through Bedrock Custom Model Import or a SageMaker endpoint, billed per minute or per hour whether used or not. The research must price that idle cost in.
