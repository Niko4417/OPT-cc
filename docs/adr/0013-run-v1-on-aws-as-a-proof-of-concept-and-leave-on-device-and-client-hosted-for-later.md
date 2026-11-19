---
status: accepted
date: 2026-11-19
decided-by: Niko4417
ticket: https://github.com/Niko4417/OPT-cc/issues/41
---

# Run V1 on AWS as a proof of concept and leave on-device and client-hosted for later

The V1 reference installation runs on AWS, in the consultancy's tenant in Frankfurt, as a proof of concept. Running on-device or client-hosted is a later effort that builds on it. This supersedes ADR-0008, which left the location open until the provider review and the bake-off. The owner wants a working proof of concept on one cloud first and broadens it afterwards.

The model policy default for V1 is third-party cloud, limited to routes that process in the EU (ADR-0014). This is safe because V1 holds only public or synthetic test data. It replaces the on-device-only default that ADR-0008 and ADR-0010 kept.

## Considered options

- Keep the location open and let the bake-off compare the best on-device setup with the best AWS setup. Rejected because the owner wants a proof of concept first, and the comparison would double the install work in V1.
- Keep on-device as the V1 target. Rejected for the same reason.

## Consequences

- The bake-off compares retrieval arms on AWS only, all on the same AWS models (ADR-0016). It no longer decides the location.
- The Mac-only work leaves this map: the memory budget review closes as out of scope, the local model server review becomes a review of the LiteLLM routes on AWS, and the trial run becomes an install-to-export run on AWS.
- To keep the later move possible, the chunk text in the app's own store stays the durable layer, and the self-built retrieval arms keep running through LiteLLM without AWS-only parts.
- In the glossary, "on-premise" is not a term. A later move is on-device (the consultant's machine) or client-hosted (the client's network or tenant).
