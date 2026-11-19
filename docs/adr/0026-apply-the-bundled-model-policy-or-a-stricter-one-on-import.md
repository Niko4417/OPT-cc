---
status: accepted
date: 2026-11-20
decided-by: Niko4417
ticket: https://github.com/Niko4417/OPT-cc/issues/21
---

# Apply the bundled model policy or a stricter one on import

A project memory bundle carries the model policy of its project memory. The importing installation applies that model policy or a stricter one, never a looser one. It refuses a bundle whose model policy has a value it does not know.

The model policy limits where a model runs, not which model it is. So this rule does not restrict a swap to a different model:

- A different answer model or reranker in an allowed location is a change of LiteLLM route. Nothing in a bundle depends on these models.
- A different embedding model in an allowed location makes the importer re-embed (ADR-0025).
- A model in a nearer location, for example after a later move from AWS to on-device, is always inside the model policy.
- A model in a farther location than the model policy allows stops the import.

## Consequences

- In V1 the default model policy is third-party cloud (ADR-0013), the farthest location, so this rule blocks nothing in practice. The EU processing rule (ADR-0014) applies to every route regardless of any model policy.
- This decision covers the moment of import only. Whether a consultant may loosen the model policy of an imported project memory afterwards is not decided. It is left to the later on-device and client-hosted effort.
