---
status: accepted
date: 2026-11-19
decided-by: Niko4417
ticket: https://github.com/Niko4417/OPT-cc/issues/41
---

# Keep V1 single-user, with no shared team server or single sign-on

V1 is one user per installation. Consultants share a project memory by exporting and importing a project memory bundle. A shared team server with single sign-on is out of scope for this map. Within one modernization project consultants usually see the same material, so per-user permission trimming does not count as a criterion for the provider choice.

## Considered options

- A team server with one project memory per team and no per-user trimming. Rejected for V1 to keep the proof of concept small.
- A team server with per-user trimming, which favours Microsoft's Entra-based trimming. Rejected for the same reason. A later effort can add it on AWS.
