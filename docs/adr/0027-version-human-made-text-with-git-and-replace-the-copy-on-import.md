---
status: accepted
date: 2026-11-20
decided-by: Niko4417
ticket: https://github.com/Niko4417/OPT-cc/issues/21
---

# Version human-made text with git and replace the copy on import

Repository content is versioned by its commit SHA and GitHub items by their `updated_at`. The human-made text of a project memory (notes, rulings and decision records) had no version history. Without one, an importer cannot tell which of two copies changed a file. So the app keeps human-made text as Markdown in one folder under git. Every write, through the app or by a coding agent, is a commit. A project memory bundle carries that folder with its history as one git bundle file.

V1 does not merge two copies of a project memory. Importing a bundle whose project memory already exists in the installation replaces the copy after one confirmation. Before it replaces anything, the import saves the local state of the human-made text as a git branch named after the import date. A consultant who wants replaced additions back copies them over by hand from that branch.

V1 is a single-user proof of concept (ADR-0015), so versioning is built now and merging is left for the iteration after V1.

## Considered options

- Unversioned files, and import replaces the copy. Rejected because the importing consultant's additions and rulings are lost, and nobody can see who changed a note or when.
- Full merging in V1: fetch the history and merge on import. Rejected for now because it needs a screen to resolve merge conflicts, a check for decision record ids created in both copies, and a block on indexing files that still hold conflict markers. A single-user proof of concept may never use it.
- Combine two copies by comparing files, without history. Rejected because the importer can see that two files differ but not which side changed them.

## Consequences

- This fixes one input for the ticket that decides how coding agents work on a project memory. The folder agents edit (ADR-0007) has to be a git repository. That ticket still names and defines the folder.
- Merging two copies is on the map under "Not yet specified". The saved branch is what a later merge starts from.
- The saved branch holds text only. What an import does with binary project artifacts that exist only in the local copy is not decided.
