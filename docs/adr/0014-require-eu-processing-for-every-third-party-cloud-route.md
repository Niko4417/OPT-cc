---
status: accepted
date: 2026-11-19
decided-by: Niko4417
ticket: https://github.com/Niko4417/OPT-cc/issues/41
---

# Require EU processing for every third-party cloud route

Every LiteLLM route and every store whose model location is third-party cloud must process and keep data in the EU. The rule covers the AWS routes and bring-your-own-key routes such as OpenAI or Requesty, which are allowed only with EU processing, for example an OpenAI project with EU data residency. Client-hosted routes follow the client's own rules, and on-device routes are exempt.

Processing in Germany, a provider without a US parent, and C5 or SecNumCloud coverage of the service are scored criteria, not hard rules. A client that requires them can get a client-hosted deployment in its own tenant (ADR-0010). The research found that only federal agencies are proven to need C5 and no survey of what private-sector clients require.

## Considered options

- Require a provider without a US parent. Rejected because it rules out every provider with a managed retrieval layer.
- Require processing in Germany. Rejected as a hard rule because EU geo inference and several useful models process elsewhere in the EU; it is scored instead.
- Apply the rule only to the chosen provider. Rejected because a bring-your-own-key route would then bypass it.
