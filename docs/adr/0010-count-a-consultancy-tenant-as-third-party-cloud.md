---
status: accepted
date: 2026-11-19
decided-by: Niko4417
ticket: https://github.com/Niko4417/OPT-cc/issues/1
---

# Count the consultancy's cloud tenant as third-party cloud

The model location keeps three values: on-device, client-hosted and third-party cloud. A model or project memory in the consultancy's own tenant at the chosen cloud provider counts as third-party cloud. One in the client's own tenant counts as client-hosted. A project memory with the default on-device-only model policy therefore never sends content to the consultancy's tenant.

## Considered options

- A fourth value, "consultancy-hosted". Rejected because no client is known to accept the consultancy's tenant while refusing a model vendor's API, so the value would change no policy check.
