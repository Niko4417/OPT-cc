# How candidates ingest GitHub code, issues and pull requests

Date: 2026-11-17. Ticket: [#5](https://github.com/Niko4417/OPT-cc/issues/5). Map: [#1](https://github.com/Niko4417/OPT-cc/issues/1).

## Question

How does each candidate stack index a GitHub repository's files, issues and pull requests, and keep each answer's citation pointing at the file, line range, issue or pull request? What are the options for splitting code along its syntax (tree-sitter based chunkers, code-aware splitters, a separate code index such as Sourcebot over MCP), and which languages do they cover?

## Answer

No candidate app ingests all four GitHub source types that a project memory needs: repo files including code, issues, pull requests, and review comments. None of them cites a line range for a repo file either. The best option is a small glue ingestion job owned by this project. The job reads repo files from a local git clone pinned to a commit SHA. It reads issues, issue comments and review comments through the GitHub REST API with `since`. It splits code with tree-sitter through `tree-sitter-language-pack` (MIT, 371 grammars). It stores every chunk with repo, commit SHA, path and start and end line, so each citation becomes a GitHub permalink like `blob/<sha>/<path>#L10-L42`. Issue and pull request chunks keep their `html_url`, and review-comment chunks keep the comment's own `html_url`.

Findings per candidate:

- **Onyx** indexes pull requests, issues and documentation files (`.md`, `.mdx`, `.markdown`, `.rst`, `.txt`, README-like names). It does not index source code, comments, review comments or diffs. Citations point at the whole issue, pull request or file URL, and file URLs use the branch name, not a commit SHA.
- **RAGFlow** 1.0 reads GitHub issues and pull requests (title and body). It reads no GitHub repo files. Its GitLab connector reads code files, but as plain text.
- **Dify**'s GitHub datasource plugin reads repo files with 91 text extensions, issues and pull requests with conversation comments. It does not read review comments, and it has no incremental sync parameter. The Dify license also bans multi-tenant use ([LICENSE](https://github.com/langgenius/dify/blob/main/LICENSE)).
- **AnythingLLM** reads GitHub repo files only, through LangChain JS. It has no GitHub issues or pull requests and cites `github://<path>`.
- **Open WebUI with oikb** syncs repo files only, with a Git blob SHA manifest for incremental sync. It attaches no GitHub URL.
- **WeKnora** has no GitHub connector, and its GitLab connector skips source code extensions.
- **Kotaemon, LightRAG and PrivateGPT** have no GitHub connector ([landscape note](https://github.com/Niko4417/OPT-cc/blob/research/rag-stack-landscape/docs/research/rag-stack-landscape.md)).
- **Keiko** indexes a local working tree with a regex symbol parser for 13 languages and cites character offsets plus a symbol label. It reads no issues or pull requests.
- **Sourcebot** is a code-only search index. It returns line numbers and ctags-based symbol definitions over MCP. It became closed source with v6 on 2026-10-08, so it no longer meets the open-source preference. The landscape note's "MIT" label is wrong for the app.

## Method

I read the connector and chunker source code at these commits (shallow clones on 2026-11-17), plus the official docs pages linked below:

| Repo | Commit | Commit date |
|---|---|---|
| onyx-dot-app/onyx | `cce02672` | 2026-11-17 |
| infiniflow/ragflow | `fe8201db` | 2026-11-18 |
| langgenius/dify-official-plugins | `53cb292f` | 2026-11-17 |
| Mintplex-Labs/anything-llm | `eb498cf2` | 2026-11-16 |
| open-webui/oikb | `29b0e453` | 2026-10-31 |
| Tencent/WeKnora | `7d064bc3` | 2026-11-16 |
| run-llama/llama_index | `99d7433f` | 2026-11-13 |
| langchain-ai/langchain | `ae476b0d` | 2026-11-17 |
| chonkie-inc/chonkie | `664454b6` | 2026-08-26 |
| deepset-ai/haystack-core-integrations | `4d227e02` | 2026-11-17 |
| sourcebot-dev/sourcebot | `2914eb23` | 2026-11-10 |
| Keiko (a local checkout) | `b5fe2a84` | 2026-11-15 |

Links to source files use `blob/<commit>` so the cited lines stay stable.

## Comparison of the candidate apps

| Stack | Repo files | Issues | Pull requests | Comments and review comments | Citation target | Incremental sync |
|---|---|---|---|---|---|---|
| Onyx | Docs only (5 extensions plus README-like names), off by default | Title and body, off by default | Title and body, on by default | None | Whole file, issue or PR URL; files use `blob/<branch>/` | `updated_at` window for issues and PRs; all doc files re-read when `pushed_at` is newer than the last run |
| RAGFlow 1.0 | None from GitHub; GitLab code files as text | Title and body | Title and body | None | Issue or PR URL | `updated_at` window plus checkpoint; separate prune pass |
| Dify GitHub plugin 0.4.12 | 91 text extensions including code | Body plus up to 30 conversation comments | Body plus up to 30 conversation comments | Review comments not read | Issue, PR or file URL in content | None in the plugin code |
| AnythingLLM | All files through LangChain JS loader | None (GitLab loader only) | None | None | `github://<path>` | Experimental per-file "live sync" |
| Open WebUI with oikb | All files | None | None | None | File name in the knowledge base | Git blob SHA manifest diff |
| WeKnora | No GitHub connector; GitLab docs formats only | None | None | None | GitLab blob URL in metadata | Commit diff since recorded head (GitLab) |
| Keiko | Local working tree, 24 code extensions plus docs | None | None | None | Character offsets plus symbol or section label | Git blob id fingerprints |
| Sourcebot | All files (code search only) | None | None | None | Repo, path and line numbers | Re-index every 1 hour by default |

### Onyx

- The GitHub connector has three toggles. `include_prs` defaults to true, and `include_issues` and `include_files` default to false ([config.py L30-32](https://github.com/onyx-dot-app/onyx/blob/cce02672acb42bcefc2d1481de1caa79e45b4fb4/backend/onyx/connectors/github/config.py#L30-L32)).
- File indexing is "intentionally limited to human-readable docs". It covers `.md`, `.mdx`, `.markdown`, `.rst` and `.txt` and excludes source code ([connector.py L85-95](https://github.com/onyx-dot-app/onyx/blob/cce02672acb42bcefc2d1481de1caa79e45b4fb4/backend/onyx/connectors/github/connector.py#L85-L95)). It skips files over 1 MB and paths under `node_modules`, `vendor`, `dist`, `build` and similar folders (L113-123).
- The docs page lists what is not indexed: "Comments on issues and pull requests, review comments, diffs, and commits" and "Source code" ([github.mdx](https://github.com/onyx-dot-app/onyx/blob/cce02672acb42bcefc2d1481de1caa79e45b4fb4/documentation/admins/connectors/official/github.mdx)).
- A pull request becomes one document with one section. The section link is `html_url` and the text is the PR body ([connector.py L436-467](https://github.com/onyx-dot-app/onyx/blob/cce02672acb42bcefc2d1481de1caa79e45b4fb4/backend/onyx/connectors/github/connector.py#L436-L467)). Issues work the same way (L506-533).
- A file document's link is `{repo.html_url}/blob/{branch}/{path}`, and its timestamp is the repo's `pushed_at` for every file (L588-595). A citation therefore points at the current branch head. After a later push, the cited text may no longer be at that URL.
- Incremental sync re-reads PRs and issues whose `updated_at` falls in the poll window. The window starts 3 hours before the last run (L1388-1395). Files are re-listed and re-fetched for the whole repo when `pushed_at` is newer than the window start (L1058-1072).
- Chunking uses chonkie's `SentenceChunker` for all text ([chunker.py L1, L161-176](https://github.com/onyx-dot-app/onyx/blob/cce02672acb42bcefc2d1481de1caa79e45b4fb4/backend/onyx/indexing/chunker.py#L161-L176)). There is no tree-sitter dependency.
- The GitLab connector can index code files as plain text with `include_code_files` ([gitlab/connector.py L204, L234-260](https://github.com/onyx-dot-app/onyx/blob/cce02672acb42bcefc2d1481de1caa79e45b4fb4/backend/onyx/connectors/gitlab/connector.py#L234-L260)). This does not help with GitHub sources.

### RAGFlow 1.0

- The GitHub connector "reads GitHub issues and pull requests" ([github.go L40](https://github.com/infiniflow/ragflow/blob/fe8201db895a4d65f2b9a53284fb9140821e26a9/internal/syncer/connector/github.go#L40)). Each PR or issue becomes one `.md` document whose `SourceID` is `html_url` and whose blob is the body ([L788-841](https://github.com/infiniflow/ragflow/blob/fe8201db895a4d65f2b9a53284fb9140821e26a9/internal/syncer/connector/github.go#L788-L841)). Comments are not fetched.
- Sync skips items whose `updated_at` is at or before the window start and stores a page checkpoint ([L272-330](https://github.com/infiniflow/ragflow/blob/fe8201db895a4d65f2b9a53284fb9140821e26a9/internal/syncer/connector/github.go#L272-L330)). A separate prune session lists all PR and issue URLs to drop deleted ones (L223-230, L336-365).
- The GitLab connector "reads GitLab merge requests, issues, and code files" ([gitlab.go L40](https://github.com/infiniflow/ragflow/blob/fe8201db895a4d65f2b9a53284fb9140821e26a9/internal/syncer/connector/gitlab.go#L40)). The parser docs say "Markdown / Text & Code" files have no extra parsing controls ([configure_parser_component.md L63](https://github.com/infiniflow/ragflow/blob/fe8201db895a4d65f2b9a53284fb9140821e26a9/docs/guides/agent/ingestion_pipeline/configure_parser_component.md)). The repo tree has no tree-sitter files.

### Dify GitHub datasource plugin

- The plugin lists repo files, issues and pull requests. Its README promises "Issues & Pull Requests ... with comments" ([README](https://github.com/langgenius/dify-official-plugins/blob/53cb292fc05094e5f76663b8cd695cf9de8583eb/datasources/github/README.md)).
- It accepts 91 text extensions, including `py`, `java`, `cs`, `go` and `ts`. It has no COBOL or ABAP extensions ([github.py L25-48](https://github.com/langgenius/dify-official-plugins/blob/53cb292fc05094e5f76663b8cd695cf9de8583eb/datasources/github/datasources/github.py#L25-L48)).
- Issue and PR content includes comments from `/issues/{n}/comments` in one request without pagination ([L585-592](https://github.com/langgenius/dify-official-plugins/blob/53cb292fc05094e5f76663b8cd695cf9de8583eb/datasources/github/datasources/github.py#L585-L592)). GitHub returns 30 items per page by default ([GitHub REST docs](https://docs.github.com/en/rest/issues/comments#list-issue-comments)), so long threads are cut at 30 comments. Review comments on diff lines (`/pulls/{n}/comments`) are not read.
- The default inventory lists only the 5 most recently updated issues and PRs per repo (L391-421). The paged preview mode lists more (L300-335).
- The plugin has no `since` parameter or checkpoint, so incremental sync depends on the Dify pipeline, which I did not check.

### AnythingLLM

- The GitHub loader wraps LangChain JS `GithubRepoLoader`. It loads recursively only when a token is set ([RepoLoader/index.js L174-197](https://github.com/Mintplex-Labs/anything-llm/blob/eb498cf2c2111b0c651d4c3781f0cdfb8168935c/collector/utils/extensions/RepoLoader/GithubRepo/RepoLoader/index.js#L174-L197)).
- Each file gets `url: "github://" + path` ([GithubRepo/index.js L56](https://github.com/Mintplex-Labs/anything-llm/blob/eb498cf2c2111b0c651d4c3781f0cdfb8168935c/collector/utils/extensions/RepoLoader/GithubRepo/index.js#L56)). The `chunkSource` stores an encrypted payload, including the token, so one file can be fetched again later (L138-160).
- Only the GitLab loader has `fetchIssues` ([GitlabRepo/RepoLoader/index.js L10, L176](https://github.com/Mintplex-Labs/anything-llm/blob/eb498cf2c2111b0c651d4c3781f0cdfb8168935c/collector/utils/extensions/RepoLoader/GitlabRepo/RepoLoader/index.js#L176)).
- Text is split by LangChain's `RecursiveCharacterTextSplitter` ([TextSplitter/index.js L193-205](https://github.com/Mintplex-Labs/anything-llm/blob/eb498cf2c2111b0c651d4c3781f0cdfb8168935c/server/utils/TextSplitter/index.js#L193-L205)).
- Re-sync is an experimental "live sync" feature with a document sync queue (`server/jobs/sync-watched-documents.js`).

### Open WebUI with oikb

- The oikb GitHub connector syncs files through the Trees API and uses "Git blob SHA" as the checksum, so only changed files upload again ([github.py L1-5, L94](https://github.com/open-webui/oikb/blob/29b0e453812c1b973cfad45420140809db4a98f5/src/oikb/connectors/github.py#L94)).
- It sends file name and directory only, with no GitHub URL ([build_manifest, L53-100](https://github.com/open-webui/oikb/blob/29b0e453812c1b973cfad45420140809db4a98f5/src/oikb/connectors/github.py#L53-L100)). It has no issues or pull requests.
- Open WebUI splits by characters, tokens or Markdown headers ([landscape note, Open WebUI](https://github.com/Niko4417/OPT-cc/blob/research/rag-stack-landscape/docs/research/rag-stack-landscape.md)).

### WeKnora

- The connector folder has Confluence, DingTalk, Feishu, GitLab, IMA, Nextcloud, Notion, RSS, Seafile and Yuque, but no GitHub ([internal/datasource/connector](https://github.com/Tencent/WeKnora/tree/7d064bc3a259f87634840199f565ec4dd644ba1e/internal/datasource/connector)).
- The GitLab connector accepts document formats such as `.pdf`, `.md`, `.docx` and `.json`, and no source code extensions ([gitlab/connector.go L387-396](https://github.com/Tencent/WeKnora/blob/7d064bc3a259f87634840199f565ec4dd644ba1e/internal/datasource/connector/gitlab/connector.go#L387-L396)).

### Keiko

Keiko is Apache-2.0 ([LICENSE](https://github.com/oscharko-dev/Keiko/blob/dev/LICENSE)).

- The repository Knowledge Pod indexes a real working tree, not the GitHub API (`packages/keiko-local-knowledge/src/repository-pod.ts` L1-6).
- The code parser "owns no syntax tree". It finds symbol boundaries with regular expressions and uses the symbol name as the section path (`packages/keiko-local-knowledge/src/parsers/code-parser.ts` L1-7). It maps 24 extensions to 13 languages: C, C++, C#, Go, Java, JavaScript, Kotlin, PHP, Python, Ruby, Rust, Swift and TypeScript (L38-63).
- The chunker cuts code at those symbol boundaries (`chunking/chunker.ts` L310, L446).
- A citation carries `sectionPath`, `characterStart` and `characterEnd`, but no line numbers (`packages/keiko-contracts/src/local-knowledge-records.ts` L165-183; `chunking/citation-mapper.ts` L1-22).
- Refresh fingerprints tracked files by "Git's canonical blob object id" and diffs them against the last baseline (`indexing/repository-fingerprints.ts` L1-6; `fingerprint-diff.ts`).
- No package ingests GitHub issues or pull requests into knowledge. The GitHub API code in `keiko-tools` creates PRs and reads CI failures (`git-pr-gateway.ts`, `git-ci-failure-context.ts`).
- Reusable parts are the blob-id fingerprint diff and the citation record shape.

### Sourcebot

- Sourcebot announced on 2026-10-08 that "Starting with v6, future Sourcebot development will take place in a private repository" ([blog post](https://www.sourcebot.dev/blog/sourcebot-is-now-closed-source)). The "chore: close source" commit landed on 2026-10-10 ([commit 93b7863c](https://github.com/sourcebot-dev/sourcebot/commit/93b7863c)). The public repo now holds only the README, config schemas and a compose file. Its MIT `LICENSE.md` covers only those files.
- The last open release, v5.1.16 (2026-10-09), is under FSL-1.1-ALv2 plus a separate license for `ee` folders ([LICENSE.md at v5.1.16](https://github.com/sourcebot-dev/sourcebot/blob/v5.1.16/LICENSE.md)). FSL allows "internal use and access" and becomes Apache-2.0 two years after each release.
- The v6 image is free for up to 5 users, with code search, MCP server and code navigation included. Permission syncing needs the Pro plan ([pricing](https://www.sourcebot.dev/pricing)).
- It indexes code only. The GitHub connection docs mention repos, branches and tags, but not issues or pull requests ([GitHub connection](https://docs.sourcebot.dev/docs/connections/github)).
- The default branch is always indexed, plus up to 64 extra revisions.
- Re-indexing runs every 1 hour by default (`reindexIntervalMs`), and connections re-sync every 24 hours (`resyncConnectionIntervalMs`) ([schemas/v3/index.json](https://github.com/sourcebot-dev/sourcebot/blob/2914eb23aa6257874f459f93a683eb04366c6cc5/schemas/v3/index.json)). Files over 2 MB are skipped.
- The MCP `grep` tool returns results "grouped by file" that "include line numbers". `read_file` takes a 1-based `offset` and a `limit` of up to 500 lines, and every tool takes a `ref` such as a commit SHA ([MCP code search tools](https://docs.sourcebot.dev/docs/mcp/tools/code-search.md)).
- Code navigation is "search-based". Symbol definitions come from universal ctags at index time ([code navigation](https://docs.sourcebot.dev/docs/features/code-navigation.md)).
- Sourcebot needs Postgres and Redis next to the app container ([docker-compose.yml](https://github.com/sourcebot-dev/sourcebot/blob/2914eb23aa6257874f459f93a683eb04366c6cc5/docker-compose.yml)), so on the Mac it runs on Colima or Podman.

## GitHub readers in the frameworks

| Library | Files | Issues and PRs | Comments | Citation fields | Incremental |
|---|---|---|---|---|---|
| LlamaIndex `GithubRepositoryReader` and `GitHubRepositoryIssuesReader` (readers-github 0.13.0) | Yes, with extension and path filters | Issue title and body; PRs come through the issues API | None | File: `file_path`, `url` (html_url), doc id is blob SHA. Issue: `source` is `html_url` | `file_exists_callback(sha)` skips known blobs; issues have no `since` |
| LangChain community `GitHubIssuesLoader` and `GithubFileLoader` | Yes | Issue body, `include_prs=True`, flag `is_pull_request` | Only the comment count | `url` is `html_url` | `since` parameter |
| Haystack `GitHubIssueViewer` and `GitHubRepoViewer` (github integration 3.0.1) | One path at a time | One issue by URL | One document per comment, with the comment's `html_url` | `url` per document | None; agent tool, not a bulk indexer |

Sources: LlamaIndex [repository/base.py L380-406, L517-521](https://github.com/run-llama/llama_index/blob/99d7433f6f71888951379ccca9294e272442c832/llama-index-integrations/readers/llama-index-readers-github/llama_index/readers/github/repository/base.py#L380-L406) and [issues/base.py L159-176](https://github.com/run-llama/llama_index/blob/99d7433f6f71888951379ccca9294e272442c832/llama-index-integrations/readers/llama-index-readers-github/llama_index/readers/github/issues/base.py#L159-L176). LangChain [community github.py](https://github.com/langchain-ai/langchain-community/blob/main/libs/community/langchain_community/document_loaders/github.py) (L41-146). Haystack [issue_viewer.py L24, L120-160](https://github.com/deepset-ai/haystack-core-integrations/blob/4d227e02d946c5d1e4084b790ea3c56fc2eeb33d/integrations/github/src/haystack_integrations/components/connectors/github/issue_viewer.py#L120-L160).

None of these reads pull request review comments, which are the comments on diff lines.

## GitHub API facts for a glue job

- `GET /repos/{owner}/{repo}/pulls/comments` lists all review comments of a repo. It takes `since` (updated after), `sort` and `per_page` up to 100. Each comment has `path`, `line`, `start_line`, `side`, `commit_id`, `original_commit_id`, `diff_hunk`, `html_url`, `pull_request_url` and `in_reply_to_id` ([GitHub REST: review comments](https://docs.github.com/en/rest/pulls/comments#list-review-comments-in-a-repository)). One call per page returns the comments across all PRs, and each comment carries the file and line range it discusses.
- `GET /repos/{owner}/{repo}/issues/comments` lists all issue and PR conversation comments with `since` ([GitHub REST: issue comments](https://docs.github.com/en/rest/issues/comments#list-issue-comments-for-a-repository)).
- `GET /repos/{owner}/{repo}/issues` takes `since` and `state=all`, and returns pull requests too, marked with a `pull_request` key ([GitHub REST: issues](https://docs.github.com/en/rest/issues/issues#list-repository-issues)).
- A permalink to a line range is `https://github.com/<owner>/<repo>/blob/<commit-sha>/<path>#L<start>-L<end>` ([GitHub docs: permanent links](https://docs.github.com/en/repositories/working-with-files/using-files/getting-permanent-links-to-files)). A link with a SHA keeps pointing at the cited lines after later pushes. A branch link such as Onyx's does not.

## Code splitting options and language coverage

| Splitter | Method | Languages | Position data per chunk | License |
|---|---|---|---|---|
| `tree-sitter-language-pack` 1.25.2 (2026-11-15) | tree-sitter grammars, on-demand download, own chunking and MCP server | 371, including COBOL, Fortran, Ada, Pascal, Smalltalk, VB.NET, SQL, T-SQL, Groovy, Apex | Syntax nodes with byte ranges | MIT; grammars must be permissive, no GPL |
| chonkie `CodeChunker` 1.7.0 | tree-sitter through `tree-sitter-language-pack>=1.8.0`; `language="auto"` tries every grammar | Same 371 | `start_index` and `end_index` from `start_byte` and `end_byte` | MIT |
| LlamaIndex `CodeSplitter` (core 0.14.26) | tree-sitter, default 40 lines with 15 lines overlap, 1500 chars | Needs `tree-sitter-language-pack<1.0` (last 0.x is 0.13.0, 2025-11-26) | `start_char_idx` and `end_char_idx` set by the node parser base | MIT |
| LangChain `RecursiveCharacterTextSplitter.from_language` (text-splitters 1.1.3) | Regex separators per language, no syntax tree | 28, including COBOL and Visual Basic 6 | `start_index` when `add_start_index=True` | MIT |
| Haystack `PythonCodeSplitter` | Python `ast` | Python only | Not checked | Apache-2.0 |
| `code-chunk` (supermemoryai, TypeScript) | tree-sitter, scope chain and imports added to each chunk | 6: TypeScript, JavaScript, Python, Rust, Go, Java | Byte and line ranges | MIT |
| `astchunk` (Python, cAST paper) | tree-sitter, split large nodes, merge small siblings | 4: Python, Java, C#, TypeScript | File path and line numbers | MIT |
| Keiko `code-text` parser | Regex symbol lines | 13 (24 extensions) | Character offsets | Apache-2.0 |
| Sourcebot | Trigram search plus universal ctags symbols, no chunking | Search works on any text; symbols where ctags has a parser | Line numbers in results | Proprietary v6; FSL up to v5.1.16 |

Sources: language pack [README](https://github.com/xberg-io/tree-sitter-language-pack/blob/main/README.md) (371 languages, grammar table, MIT, "Copyleft licenses ... are not accepted") and [PyPI](https://pypi.org/project/tree-sitter-language-pack/). Chonkie [code.py L21-50, L99-133, L185-186](https://github.com/chonkie-inc/chonkie/blob/664454b689d68614bcfc0b1d57fc2a28b5862825/src/chonkie/chunker/code.py#L99-L133) and [pyproject.toml L91](https://github.com/chonkie-inc/chonkie/blob/664454b689d68614bcfc0b1d57fc2a28b5862825/pyproject.toml#L91). LlamaIndex [code.py L13-16, L111-134](https://github.com/run-llama/llama_index/blob/99d7433f6f71888951379ccca9294e272442c832/llama-index-core/llama_index/core/node_parser/text/code.py#L111-L134) and [interface.py L110-115](https://github.com/run-llama/llama_index/blob/99d7433f6f71888951379ccca9294e272442c832/llama-index-core/llama_index/core/node_parser/interface.py#L110-L115). LangChain [base.py L449-479](https://github.com/langchain-ai/langchain/blob/ae476b0d57c6af684ca76d1fc46160b421e52abc/libs/text-splitters/langchain_text_splitters/base.py#L449-L479) and [character.py L659](https://github.com/langchain-ai/langchain/blob/ae476b0d57c6af684ca76d1fc46160b421e52abc/libs/text-splitters/langchain_text_splitters/character.py#L659). Haystack [PythonCodeSplitter docs](https://docs.haystack.deepset.ai/docs/pythoncodesplitter). [code-chunk README](https://github.com/supermemoryai/code-chunk). [astchunk README](https://github.com/yilinjz/astchunk).

Notes on coverage:

- The language pack has no grammar for ABAP, PL/I, RPG, JCL, Natural or PL/SQL (checked against the README grammar table). For these legacy languages the options are LangChain's separator lists (COBOL and VB6 only) or a line-window fallback.
- The COBOL grammar in the pack ships no tag or fold queries (README table), so symbol extraction for COBOL is weaker than for Java or Python.
- LlamaIndex `CodeSplitter` and chonkie need incompatible major versions of the language pack (`<1.0` against `>=1.8.0`). Pick one of them per Python environment.
- Only `code-chunk`, `astchunk` and Sourcebot hand back line numbers directly. All other splitters give character or byte offsets, and the glue job converts them to lines by counting newlines. Chonkie gives byte offsets, so for non-ASCII files the glue must count bytes, not characters.

## Options ranked

1. **Own glue ingestion job writing into the chosen app's store (recommended).** It runs `git clone` and `git fetch`, then `git diff --name-status <last-sha> <head-sha>` for changed files. It calls chonkie `CodeChunker` or `tree-sitter-language-pack` directly for 371 languages, with a line-window fallback for files without a grammar. It calls the GitHub REST endpoints above with `since` for issues, conversation comments and review comments. Each chunk stores repo, commit SHA, path, start line and end line, or the issue, PR or comment `html_url`. This is the only option that covers code, issues, pull requests and review comments with line-range citations and incremental sync. It costs one job of a few hundred lines, and every library it uses is MIT.
2. **Onyx connector for issues, pull requests and Markdown, plus option 1 for code and comments.** This applies if Onyx is the chosen app. Onyx covers decision records and workshop results stored as Markdown, and issue and PR bodies, with no glue. The glue job still adds code, comments and review comments. Onyx file citations use branch URLs, so the glue should also index Markdown if line-stable links matter.
3. **RAGFlow GitHub connector plus option 1.** This applies if RAGFlow is chosen. The connector covers only issue and PR bodies, so the glue job does nearly all the work.
4. **Sourcebot v6 as an extra MCP code-search tool.** It gives grep with line numbers, file reads by SHA and ctags symbols. It is free up to 5 users but closed source, and it needs Postgres, Redis and a container runtime. Use it only if the open-source preference is relaxed, or pin v5.1.16 under FSL for internal use.
5. **Dify GitHub plugin.** It is the only connector that reads code files and issue and PR comments, but it caps threads at 30 comments, skips review comments, has no incremental sync in the plugin, and comes with a restrictive license.
6. **AnythingLLM, Open WebUI with oikb, WeKnora and Keiko as-is.** They read files only, or no GitHub at all, and none cites a line range. Keiko's blob-id fingerprint diff and citation record shape are worth reusing in option 1.

## Open points

- Whether the glue job writes straight into the app's vector store or goes through the app's ingestion API depends on the stack choice in the map.
- Dify's pipeline-level re-sync behavior was not checked.
- Line numbers for review comments refer to the diff side (`side`, `commit_id`). A citation should link the comment's `html_url`, not a file permalink.
