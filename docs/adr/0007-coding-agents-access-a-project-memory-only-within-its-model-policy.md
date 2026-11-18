---
status: accepted
date: 2026-11-18
decided-by: Niko4417
ticket: https://github.com/Niko4417/OPT-cc/issues/9
---

# Let coding agents work on a project memory, but only within its model policy

Consultants use Codex, Claude Code, OpenCode and GitHub Copilot. V1 lets these coding agents read and edit a project memory, and the app indexes their edits back. A coding agent sends what it reads to its own model, so Claude Code on Anthropic's cloud would break an on-device-only model policy.

Until the ticket on coding agents decides how the model policy is enforced, an agent gets access to a project memory only when its model location is within that project memory's model policy.

What agents edit is a separate concept from the project memory bundle: a plain folder of the project memory's project artifacts and records, in the formats of ADR-0001 and ADR-0002. The bundle stays the signed form for handing a project memory to another consultant, because any edit breaks its signature. The ticket on coding agents names and defines the folder.

## Considered options

- Let agents edit the project memory bundle directly. Rejected because every edit breaks the signature.
- Name the editable form after Keiko. Rejected because the base stack is still open.
