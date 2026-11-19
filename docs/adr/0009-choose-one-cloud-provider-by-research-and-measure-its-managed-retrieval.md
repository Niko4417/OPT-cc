---
status: accepted
date: 2026-11-19
decided-by: Niko4417
ticket: https://github.com/Niko4417/OPT-cc/issues/1
---

# Choose one cloud provider by research and measure its managed retrieval in the bake-off

The candidates are Microsoft (Foundry with Foundry IQ and Azure AI Search), AWS (Bedrock and Bedrock Knowledge Bases) and OVHcloud. A research ticket compares them for this use case, and a "Review and adopt" ticket picks one. The owner has a commercial deal with OVHcloud. The deal is a scored criterion like cost, not a veto over measured quality. Public files only say "commercial deal in place".

The chosen provider's models are reached through LiteLLM routes, like every other model. If the provider has a managed retrieval layer, that layer joins the bake-off as one more arm and must win on the gold question set like any other candidate.

## Considered options

- Use Microsoft and AWS side by side. Rejected because two clouds double the glue code.
- Let the OVHcloud deal decide by itself. Rejected because the stack choice follows measured answer quality.
