---
status: accepted
date: 2026-11-18
decided-by: Niko4417
ticket: https://github.com/Niko4417/OPT-cc/issues/9
---

# Define project artifacts by how they arrive, and take audio and video in V1

A project artifact is anything a consultant adds to a project memory by hand, beyond the repositories the project memory is connected to. Examples are a Word decision log, slides, a spreadsheet, an interview recording or a recorded workshop. The term describes how a source arrived. "Decision record" and "workshop result" describe what a source says, so a decision record can be repository content (Thunderbird's ADR 0010 in `docs/`) or a project artifact (a decision log emailed by the client).

V1 takes audio and video project artifacts. On-device speech-to-text turns each one into a transcript in the workshop result transcript format (ADR-0002), with speaker turns and timestamps, and answers cite time ranges. The model policy covers the transcription model like any other model.

## Considered options

- "Artifact" alone. Rejected because it collides with build artifacts, which the code ingestion meets in Gradle outputs and GitHub Actions, and with Claude artifacts.
- Define project artifacts by content, as everything that is not code or its documentation. Rejected because an ADR in `docs/` would then be both, and the boundary could not be tested.

## Consequences

How ingestion, the bundle and citations are organized around the two source origins is decided in a separate ticket after local transcription is researched.
