---
status: accepted
date: 2026-11-20
decided-by: Niko4417
ticket: https://github.com/Niko4417/OPT-cc/issues/21
---

# Carry sources and human-made knowledge as plain files and rebuild derived data on import

ADR-0018 made a project memory complete as plain files. The project memory bundle carries that plain-file form next to the chunks, so an imported project memory can be read with grep and file reads, and citations open the cited text without GitHub access. A bundle carries what cannot be recomputed and leaves out what the importer can rebuild.

What travels:

- **Repository content.** The working tree at the recorded commit, without the `.git` history.
- **Issues and pull requests.** The Markdown files under `github/`, one per item. The GitHub JSON that the bundle research proposed is not carried.
- **Documents.** The original files of project artifacts, such as PDF, Office files and images.
- **Recordings.** The transcript always travels. The audio or video file travels only when the exporter chooses "include recordings".
- **Human-made knowledge.** Notes typed into the app, rulings and decision records a human confirmed from a decision log travel as files with their history (ADR-0027), plus their chunks and vectors.

What does not travel:

- **The decision register, the link graph and the static code graph.** The importer rebuilds them from the files without a model call. A rebuilt decision register applies the imported rulings, so a decision conflict that the exporter resolved stays resolved.
- **The document parse cache.** Re-embedding needs only chunk text.

## Considered options

- Issues and pull requests as GitHub JSON. Rejected because ADR-0018 made the Markdown files the plain form, and two forms of one item can disagree.
- The clone with its `.git` history. Rejected for size. An importer with a GitHub token clones again when it wants to sync.
- Always include recordings. Rejected because one recorded workshop is hundreds of megabytes, next to about 90 MB for the rest of a mid-size bundle.
- Ship the decision register and the graphs. Rejected because a second copy can disagree with the files it was computed from.
- Ship the parse cache. Rejected because only a re-chunk of a document needs it, and that parses the document again.

## Consequences

- Without the recording, a citation into a workshop shows the transcript turn and its time range but cannot play it.
- An importer without a GitHub token can read and search the project memory but cannot sync it.
- The size estimate in ADR-0021 counts text only. Original document files add to it.
