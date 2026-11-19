---
status: accepted
date: 2026-11-19
decided-by: Niko4417
ticket: https://github.com/Niko4417/OPT-cc/issues/20
---

# Split repository files by type, skip files by content, and cite commit-pinned permalinks

This decides what the GitHub glue job (ADR-0018) ingests, how it cuts it into chunks and how answers cite it. It decides nothing about embedding or index layout, which belong to the code retrieval and knowledge organization reviews.

**GitHub content.** The job ingests issues, pull requests, conversation comments, review comments on diff lines and review bodies (`/pulls/{n}/reviews`), all with `state=all`. Commit messages stay out in V1 and come in only if a gold question needs them. Pull requests, issues and comments whose author account is of type `Bot` are skipped, with a setting per project memory to keep them. The test dataset has 451 Dependabot pull requests.

**Records for GitHub items.** Each issue or pull request body, conversation comment, review comment and review body is its own record with its own `html_url`. Each record starts with a short header (repository, number, title, state, labels), so a comment makes sense on its own. A review comment also carries its `path`, line range and `diff_hunk`. A text longer than the chunk size is split with the text splitter, and each piece keeps the same `html_url`.

**Repository files by type.** Each file from `git ls-files` at the pinned SHA goes to the splitter for its type:

- Code goes to chonkie `CodeChunker` on `tree-sitter-language-pack` 1.x (371 grammars, MIT). Byte offsets become line numbers by counting bytes, not characters. Files with no grammar get a line-window fallback.
- Markdown and plain text go to a heading-aware text splitter, so decision records and workshop results keep their sections.
- PDF, Word, slides and spreadsheets go to the Docling path, which the folder and document ingestion review adopts. CSV and TSV go to the document path too.

Chunk size is tuned in the bake-off.

**Skipping by content, not size.** There is no size cap for text files below a 20 MB safety cap. A legacy COBOL program or PL/SQL package can exceed 1 or 2 MB and is exactly what a consultant asks about. The rules:

- Detect the encoding first (BOM, then `charset-normalizer`, MIT) and convert to UTF-8. Skip a file as binary only if it is still binary after that. Skip it as "unknown encoding" if it decodes to nothing, such as EBCDIC, instead of chunking garbage.
- Apply the minified test (average line length over 500 characters) to web assets only (`.js`, `.mjs`, `.css`).
- Skip lockfiles and files with a generated marker: a header such as `// Code generated ... DO NOT EDIT`, GitHub Linguist's content-based generated checks, or `.gitattributes`.
- Index vendored paths and folders named `generated/` with a `vendored` or `generated` tag instead of skipping them, because legacy repositories often hold hand-patched libraries and output from old code generators.

**A rescue-only second check.** Magika (Google, Apache-2.0, a few MB, about 5 ms per file on one CPU, 214 content types including COBOL, Fortran, Pascal, VBA and SQL) checks every file the rules skip and every file with a missing or unknown extension. It can rescue a file it labels as code or text with high confidence, and it picks the chunker for files without an extension. go-enry (Apache-2.0 port of GitHub Linguist) settles ambiguous extensions such as `.cls`, `.inc` and `.h`. The second check never skips a file the rules kept, because a wrong skip loses an answer and a wrong keep only adds noise.

**Visibility and tests.** An ingestion report lists every skipped file with its path, size and reason, and every disagreement between the rules and Magika. A consultant can force-include or force-exclude paths per project memory. Skipped files stay in the pinned clone, so agent arms can still read them. A set of edge-case files (UTF-16 SQL, EBCDIC, a 3 MB COBOL program, one-line JSON, minified JS, a lockfile, a patched vendored library) runs in the no-model CI checks on every pull request.

**Citations.** A chunk from a repository file cites `https://github.com/<owner>/<repo>/blob/<sha>/<path>#L<start>-L<end>`, pinned to the commit it was read at. A chunk from a file that has not changed since keeps its old SHA, because that permalink still shows the same lines. Issues, pull requests and comments cite their own `html_url`. A review comment cites its `html_url`, not a file permalink, because its line numbers refer to the diff.

## Considered options

- One joined text per issue or pull request thread, cut by tokens. Rejected because it breaks links to single comments and mixes speakers in one chunk.
- LlamaIndex `CodeSplitter`. Rejected because it pins `tree-sitter-language-pack` below 1.0, which conflicts with chonkie.
- A per-file size cap of 1 MB (Onyx) or 2 MB (Keiko). Rejected because it drops large legacy programs and keeps small generated or minified files. On the test dataset, all 19 files over 1 MB are binary, generated or minified, so the cap tests nothing there.
- A language model as the second check. Deferred, because the public test dataset has no files to measure it on, so ADR-0016 cannot pick its model.
- Branch links for citations, as Onyx uses. Rejected because the cited lines move after the next push.

## Consequences

- The language-model check and exporters for binary code containers (Oracle Forms `.fmb`, PowerBuilder `.pbl`, binary Delphi `.dfm`) wait with legacy languages in the map's "Not yet specified".
