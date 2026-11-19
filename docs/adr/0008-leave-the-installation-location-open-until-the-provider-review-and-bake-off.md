---
status: accepted
date: 2026-11-19
decided-by: Niko4417
ticket: https://github.com/Niko4417/OPT-cc/issues/1
---

# Leave the installation location open until the provider review and the bake-off

The map's destination was a local-first reference installation on one 64 GB Apple Silicon Mac. After a meeting with the Keiko team, the owner wants to consider one cloud provider instead. The destination no longer fixes a location: the reference installation runs on-device or on one cloud provider, and the provider review and the bake-off decide which. Two cloud providers are never used side by side.

Until that decision, the tickets that only make sense for a Mac target are on hold: the model server and LiteLLM review, the memory budget review, and the install part of the trial run. The model policy default stays on-device only until an ADR changes it.

## Considered options

- Keep on-device as the fixed target and allow one cloud provider only as an extra model location. Rejected because the owner wants the cloud option measured as a full alternative, not as an add-on.
- Move the installation to one cloud now and start a new map. Rejected because the Keiko team's claims are not yet verified and ten finished research tickets still apply.

## Consequences

Research tickets on code retrieval, knowledge organization, embedding lock-in, graph knowledge and the cloud provider choice feed the provider review and the bake-off. The research on the memory budget, the model server and the start script stays valid only if the decision lands on-device.
