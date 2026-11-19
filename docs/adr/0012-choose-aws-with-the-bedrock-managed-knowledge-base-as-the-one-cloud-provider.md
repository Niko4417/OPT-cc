---
status: accepted
date: 2026-11-19
decided-by: Niko4417
ticket: https://github.com/Niko4417/OPT-cc/issues/41
---

# Choose AWS with the Bedrock Managed Knowledge Base as the one cloud provider

The one cloud provider is AWS. The project memory's managed retrieval layer is the Amazon Bedrock Managed Knowledge Base in Frankfurt (eu-central-1), and it joins the bake-off as one more arm. The provider must offer a managed retrieval layer: without one, the bake-off cannot test whether a managed platform answers better than the self-built stack, which is why the cloud option exists. This replaces the "if it has one" in ADR-0009. The commercial deal with OVHcloud is no longer a criterion. The rest of ADR-0009 stands: exactly one provider, its models reached through LiteLLM routes, and its managed layer must win on the gold question set like any other arm.

AWS runs Bedrock, the Managed Knowledge Base and AgentCore in Frankfurt, has Bedrock in its C5 scope, has the lowest cost for a small project memory ($49 a month, $44 idle, at list prices) and has a reranker built into the Managed Knowledge Base at no extra cost. Its weaknesses are a US parent, permission filtering that trusts an email address the app passes, and a Managed Knowledge Base whose embedding model is unnamed and whose vectors cannot be exported. Per-user permission trimming no longer counts (ADR-0015), and the chunk text in our own store stays the durable layer (ADR-0017), so neither weakness decides the choice.

Evidence: [cloud provider research](https://github.com/Niko4417/OPT-cc/blob/research/cloud-provider/docs/research/cloud-provider.md) and the open points checked in the review ticket.

## Considered options

- Microsoft (Foundry, Foundry IQ). Runner-up. It has the most mature agentic retrieval, takes our own chunks and vectors, and has a free knowledge graph preview. It lost because neither lab model runs there pay-per-token with EU processing, Germany West Central accepts no new search services, the scale-to-zero Serverless tier is a preview outside Germany, and its main advantage, Entra-based permission trimming, does not count for a single-user V1. It stays the first choice for a client that runs on Azure and wants a client-hosted deployment in its own tenant (ADR-0010).
- OVHcloud. It is the only provider with a non-US parent and serves Qwen3.6-35B-A3B pay-per-token in the EU, but it has no managed knowledge base, no agentic retrieval, no graph product and no reranker. Rejected by the managed retrieval requirement. Its AI Endpoints also runs only in France and appears in no published SecNumCloud, C5 or ISO 27001 scope.
- Keep the OVHcloud deal as a scored criterion (ADR-0009). Rejected by the owner, who took the deal out of the comparison.

## Consequences

- The app still runs its own GitHub ingestion, because no managed knowledge base has a GitHub connector.
- The bake-off arm feeds the Managed Knowledge Base our own chunks (ADR-0017).
- AWS GraphRAG needs a customer-managed knowledge base, reads only from S3 and costs about $424 a month in Frankfurt. Whether it joins the bake-off is left to the graph review.
