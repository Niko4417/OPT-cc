# How coding agents read and edit a project memory

Date: 2026-11-18. Ticket: [Find how coding agents can read and edit a project memory](https://github.com/Niko4417/OPT-cc/issues/30), part of the map [Project memory reference installation](https://github.com/Niko4417/OPT-cc/issues/1). These are research findings, not owner decisions. The owner reviews them in [Decide how coding agents work on a project memory](https://github.com/Niko4417/OPT-cc/issues/31).

## Question

How can Codex, Claude Code, OpenCode and GitHub Copilot read a project memory, edit its project artifacts and records, and get their edits indexed back into the app? Compare MCP support and its limits in each agent, a file-based round trip through `AGENTS.md` and skills over a plain folder of records in the formats of ADR-0001 and ADR-0002, and how existing tools solve this. Check for each agent whether its model calls can be routed through the local LiteLLM proxy, so the project memory's model policy can be enforced (ADR-0007). Licenses must allow free commercial use.

## Answer

1. **Use both paths, with files as the source of truth.** Agents edit decision records and workshop results as plain Markdown files in one folder that the app owns. They search and read the indexed project memory through a small local MCP server with read-only tools. No agent needs an MCP write tool, because every agent already edits files with its own diff, approval and undo.
2. **The app re-indexes from a folder watcher plus a periodic hash rescan.** Agent hooks can make re-indexing faster, but they are optional. Hooks fire only for the agent's own edit tools, and shell commands or editors bypass them.
3. **One `AGENTS.md` and one skill reach all four agents,** if the skill is copied into three folders: `.agents/skills/` (Codex, OpenCode, Copilot CLI), `.claude/skills/` (Claude Code, OpenCode, Copilot CLI) and `.github/skills/` (Copilot in VS Code). Claude Code reads `AGENTS.md` only when no `CLAUDE.md` exists, so the folder must not contain one.
4. **The MCP server exposes tools only and speaks the older protocol revisions.** Resources, prompts, roots and list-change notifications are missing in at least one agent. Codex announces protocol 2025-06-18, and OpenCode defaults to revisions up to 2025-11-25, while the current spec is 2026-07-28. The server is a stdio process that forwards to the app's loopback API. Tool results stay under 8 KB, the smallest limit found (Codex's 10,000-byte fallback for unknown models).
5. **All four agents can send their model calls to LiteLLM on 127.0.0.1:4000, but with different strength.** Codex, OpenCode and Copilot CLI can be held to on-device only by configuration that a repository cannot override. Claude Code works through LiteLLM's Anthropic route, but Anthropic does not support non-Claude models behind a gateway. Copilot in VS Code stays on-device only while the user is signed out of GitHub and several settings are set. Nothing in any agent lets the app prove which model an agent used, so the app enforces the model policy by generating each agent's configuration and checking it, not by observing traffic.
6. **License check.** Codex (Apache-2.0), OpenCode (MIT) and VS Code with the Copilot extension (MIT) pass. Copilot CLI has a proprietary license that grants free use, including commercial use. Claude Code is proprietary, and its use falls under Anthropic's Commercial Terms. The owner has to decide whether Claude Code meets the "free commercial use" rule.
7. **A coding agent on the local models is a new load on the 64 GB Mac.** Coding agents need large prompts, and Claude Code will not auto-compact below 100,000 tokens. [Check that the lab models fit in 64 GB with embedding and reranking](https://github.com/Niko4417/OPT-cc/issues/4) capped prompts at 32k tokens. Running agents on-device therefore needs its own measurement.

## 1. Two integration paths

ADR-0007 asks for a plain folder of project artifacts and records that agents edit, separate from the signed project memory bundle. The two paths answer different needs.

| Need | MCP server | Plain folder with `AGENTS.md` and a skill |
| --- | --- | --- |
| Search the whole project memory, including code, issues and PDFs | Yes. Only the app has the index and the citations | No. `grep` finds words, not answers with citations |
| Read one decision record or the decision register with mapped status | Yes, with the status mapping of ADR-0001 applied | Partly. The agent reads the raw file and must map status itself |
| Create or change a record | Possible, but each agent then shows an opaque tool call instead of a diff | Yes. Every agent edits files with diffs, approval and undo |
| Works without the app running | No | Yes |
| Review and history | Only what the app logs | Git history of the folder |

Existing tools land on the same split (section 6): the files stay the truth, an embedded index sits next to them, a read-only MCP server answers searches, and a watcher or rescan keeps the index current.

### Proposed folder

ADR-0007 leaves the folder's name and definition to this topic. A candidate for the owner to name in [Decide how coding agents work on a project memory](https://github.com/Niko4417/OPT-cc/issues/31):

```
<memory-folder>/
  AGENTS.md                        rules for agents: formats, ids, where things go
  .agents/skills/project-memory/   SKILL.md plus templates (copied to .claude/skills and .github/skills)
  .mcp.json                        project MCP config for Claude Code and Copilot CLI
  decisions/                       MADR 4.0 records with id, type, supersedes (ADR-0001)
  workshops/                       workshop results with front matter (ADR-0002)
  artifacts/                       project artifacts added by hand (PDF, slides, audio)
```

The folder is a git repository of its own, because client repositories are read-only (ADR-0001). Git gives the review trail and lets the app detect changes after a restart. Agents edit only Markdown. For binary project artifacts they read the extracted text through the MCP server, because the app already runs Docling on them.

## 2. Per-agent comparison

| | Codex CLI | Claude Code | OpenCode | Copilot CLI | Copilot in VS Code |
| --- | --- | --- | --- | --- | --- |
| Version on 2026-11-18 | 0.170.0 [C1] | 2.1.327 [A1] | 2.3.2 (1.19.6 still maintained) [O2] | 1.0.114 [G4] | VS Code 1.147.0, Copilot extension 0.75.0 [G1][G2] |
| License | Apache-2.0 [C2] | Proprietary, Commercial Terms [A2][A3] | MIT [O1] | Proprietary, free use allowed [G5] | MIT [G1][G2] |
| MCP transports | stdio, Streamable HTTP; no SSE [C4] | stdio, HTTP; SSE deprecated [A7] | stdio, Streamable HTTP [O5] | stdio, HTTP, SSE [G14] | stdio, HTTP, SSE [G10] |
| MCP features beyond tools | Resources through built-in tools, server instructions, elicitation; no prompts, roots or sampling [C4] | Resources, prompts, roots, elicitation, list_changed [A7] | Resources through built-in tools, prompts, server instructions, elicitation [O5] | Not documented in detail | Copilot harness: elicitation, sampling; no prompts, resources only for MCP Apps, no roots [G11] |
| MCP output limit | 10,000 tokens for OpenAI models; 10,000 bytes for unknown models such as a local one [C4] | 25,000 tokens; text over 50,000 characters goes to a file [A7][A8] | Not documented | 20,480 bytes (`COPILOT_LARGE_OUTPUT_THRESHOLD_BYTES`) [G16] | Not verified; 128-tool limit on the Copilot API path [G12] |
| `AGENTS.md` | Native, 32 KiB combined limit, project file skipped when untrusted [C6] | Native since 2.1.277, only when no `CLAUDE.md` exists [A9] | Native, `CLAUDE.md` fallback [O10] | Native, also `CLAUDE.md` and `.github/copilot-instructions.md` [G17] | Native [G18] |
| Skill folders | `.agents/skills`, `.codex/skills` [C7] | `.claude/skills` [A10] | `.opencode/skills`, `.claude/skills`, `.agents/skills` [O11] | `.github/skills`, `.claude/skills`, `.agents/skills` [G20] | Copilot harness: `.github/skills`; Local harness also `.claude/skills` and `.agents/skills` [G21] |
| Hook after an edit | `PostToolUse`, matcher `apply_patch` [C8] | `PostToolUse`, matcher `Edit\|Write`, `http` hook type [A11] | Plugin subscribed to `filesystem.changed` [O13] | `postToolUse` in `.github/hooks/*.json` [G22] | Same hook files, preview [G23] |
| Model calls through LiteLLM | Yes, needs `/v1/responses` [C4][L2] | Yes, needs `/v1/messages`; non-Claude models unsupported by Anthropic [A5][A6] | Yes, OpenAI-compatible provider [O14] | Yes, `providers.json` plus `COPILOT_OFFLINE=true` [G15][G26] | Yes, Custom Endpoint; semantic search, completions and embeddings stay on GitHub [G8] |
| On-device only can be held | Yes, by user config | Yes, by managed settings, unsupported model path | Yes, by `policies` | Yes, with offline mode | Only while signed out of GitHub |

### Codex CLI

**MCP.** Servers go under `[mcp_servers.<name>]` in `~/.codex/config.toml` or a project `.codex/config.toml`. Codex ignores project layers in an untrusted project, and trust is stored as `projects."<path>".trust_level` [C3][C4 `config/src/loader/mod.rs:133-135`]. The stdio keys are `command`, `args`, `env`, `env_vars` and `cwd`. Every server also takes `enabled`, `startup_timeout_sec`, `tool_timeout_sec`, `enabled_tools`, `disabled_tools`, `default_tools_approval_mode` and per-tool `tools.<tool>.output_token_limit` [C4 `config/src/mcp_types.rs:240-312`]. The docs give 10 s startup and 60 s per call. The 0.170.0 source falls back to 30 s and 300 s, so set both [C3][C4 `codex-mcp/src/rmcp_client.rs:122-123`]. Codex accepts at most 2,048 catalog items per server [C4 `codex-mcp/src/pagination.rs:10`].

Codex exposes MCP resources to the model through three built-in tools: `list_mcp_resources`, `list_mcp_resource_templates` and `read_mcp_resource`. The client declares only the `elicitation` capability and protocol 2025-06-18. It sends no roots and no sampling. It ignores `resources/list_changed`, and it re-lists tools on `tools/list_changed` only behind an off-by-default feature flag [C4 `codex-mcp/src/rmcp_client.rs:1364-1394`, `rmcp-client/src/logging_client_handler.rs:82-92`].

In the default `auto` approval mode, a tool with `readOnlyHint: true` runs without asking. Any other tool asks unless it sets both `destructiveHint: false` and `openWorldHint: false` [C4 `core/src/mcp_tool_call.rs:2793-2820`]. Read-only search tools with honest annotations therefore run without prompts. `codex mcp add pm -- <cmd>` adds a server [C3].

**Instructions and skills.** Codex walks from the project root (marker `.git`) down to the working directory and takes one file per directory: `AGENTS.override.md`, then `AGENTS.md`, then names in `project_doc_fallback_filenames`. The combined limit is `project_doc_max_bytes`, 32,768 bytes by default [C4 `core/src/agents_md.rs:7-18, 162-171, 279-299`]. Skills come from `.agents/skills` in every directory from the project root to the working directory, `.codex/skills`, `~/.agents/skills` and `/etc/codex/skills`. At start Codex loads only names and descriptions, capped at 2% of the context window [C4 `ext/skills/src/host_roots.rs:126-187`][C7].

**Hooks.** Hooks are on by default. `PostToolUse` with matcher `apply_patch` (aliases `Edit` and `Write`) fires after file edits. The hook gets the patch text in `tool_input.command`, which names the files, but no separate path field. Each user approves a hook once with `/hooks`, and editing the hook file asks again [C8][C4 `core/src/tools/hook_names.rs:28-38`].

**Model routing.** Only `wire_api = "responses"` is accepted. `"chat"` fails with "`wire_api = \"chat\"` is no longer supported" [C4 `model-provider-info/src/lib.rs:112, 142`]. A project `.codex/config.toml` can never set `model_provider`, `model_providers`, `openai_base_url` or `otel`, so a repository cannot redirect model calls [C4 `config/src/loader/mod.rs:90-103`]. Codex sends telemetry to OpenAI by default even with a custom provider, and it offers a hosted web search tool to custom providers. The settings below turn both off [C4 `otel/src/config.rs:11-35`, `core/src/tools/spec_plan.rs:736-751`].

```toml
# ~/.codex/config.toml in a dedicated CODEX_HOME without auth.json
model = "local-coder"               # a LiteLLM model_name
model_provider = "litellm"
model_context_window = 131072
web_search = "disabled"
check_for_update_on_startup = false

[model_providers.litellm]
name = "LiteLLM (local)"
base_url = "http://127.0.0.1:4000/v1"
env_key = "LITELLM_API_KEY"
wire_api = "responses"

[analytics]
enabled = false
[feedback]
enabled = false
[otel]
exporter = "none"
metrics_exporter = "none"
[features]
apps = false
```

LiteLLM serves `/v1/responses` and documents this Codex setup [L1][L2]. For an `openai/` route with a custom `api_base`, such as `mlx_lm.server`, LiteLLM forwards `/responses` unchanged. The route must opt in to the Responses-to-chat bridge with `use_chat_completions_api: true` or the `openai/chat_completions/<model>` prefix, because `mlx_lm.server` has no `/v1/responses` [L3]. Not verified: tool-call and streaming fidelity through that bridge to `mlx_lm.server`.

### Claude Code

**MCP.** Scopes are local and user (`~/.claude.json`) and project (`.mcp.json` at the repo root). `.mcp.json` expands `${VAR}` and `${VAR:-default}`. In interactive mode Claude Code asks before it uses a server from `.mcp.json`. `-p` mode loads them without asking [A7]. It supports tools, resources (as `@` mentions plus list and read tools), prompts (as slash commands), roots, elicitation and `list_changed`. `MAX_MCP_OUTPUT_TOKENS` defaults to 25,000, and a text result over 50,000 characters is saved to a file. `MCP_TIMEOUT` (startup) defaults to 30,000 ms. Tool search defers MCP tools by default [A7][A8].

**Instructions and skills.** Claude Code reads `AGENTS.md` natively since 2.1.277 through the built-in plugin `cc-plugin-agents-md@builtin`, but by default only when there is no `CLAUDE.md`, `.claude/CLAUDE.md` or `CLAUDE.local.md` in the working directory or above. With a `CLAUDE.md`, put `@AGENTS.md` at its top [A9]. Skills live in `.claude/skills/<name>/SKILL.md` and `~/.claude/skills/`. The docs do not mention `.agents/skills` [A10].

**Hooks.** A `PostToolUse` hook with matcher `Edit|Write` gets `tool_input.file_path` as an absolute path. The `http` hook type posts the hook JSON to a URL, so the app can receive edits directly [A11]:

```json
{ "hooks": { "PostToolUse": [ { "matcher": "Edit|Write",
  "hooks": [ { "type": "http", "url": "http://127.0.0.1:<app port>/reindex", "timeout": 10 } ] } ] } }
```

**Model routing.** The gateway must serve the Anthropic Messages API at `/v1/messages` and forward the `anthropic-beta` and `anthropic-version` headers. `/v1/messages/count_tokens` is optional [A12]. LiteLLM documents this setup, including non-Anthropic models: every model id Claude Code sends must exist as a LiteLLM `model_name` [L4][L5]. Anthropic's own docs say it "doesn't support routing Claude Code to non-Claude models through any gateway" [A6]. Documented caveats for a local model [A5][A12][A15]:

- An unknown model id is assumed to have a 1M-token window. Set `CLAUDE_CODE_MAX_CONTEXT_TOKENS` to the real window.
- `CLAUDE_CODE_AUTO_COMPACT_WINDOW` does not go below 100,000 tokens, so the local model needs about 100k context.
- Pre-release request fields can fail on a translated upstream. `CLAUDE_CODE_DISABLE_EXPERIMENTAL_BETAS=1` strips most of them.

```json
{ "env": {
  "ANTHROPIC_BASE_URL": "http://127.0.0.1:4000",
  "ANTHROPIC_AUTH_TOKEN": "<LiteLLM key>",
  "ANTHROPIC_MODEL": "local-coder",
  "ANTHROPIC_DEFAULT_OPUS_MODEL": "local-coder",
  "ANTHROPIC_DEFAULT_SONNET_MODEL": "local-coder",
  "ANTHROPIC_DEFAULT_HAIKU_MODEL": "local-coder",
  "CLAUDE_CODE_MAX_CONTEXT_TOKENS": "131072",
  "CLAUDE_CODE_DISABLE_EXPERIMENTAL_BETAS": "1",
  "CLAUDE_CODE_DISABLE_NONESSENTIAL_TRAFFIC": "1",
  "CLAUDE_CODE_DISABLE_OFFICIAL_MARKETPLACE_AUTOINSTALL": "1" },
  "skipWebFetchPreflight": true }
```

With `ANTHROPIC_AUTH_TOKEN` set in the user settings or the shell, Claude Code skips the Anthropic login [A5]. `CLAUDE_CODE_DISABLE_NONESSENTIAL_TRAFFIC` turns off telemetry, error reports, auto-update and the fast-mode check. The WebFetch domain check (it sends the hostname to `api.anthropic.com`) needs `skipWebFetchPreflight` [A16][A18]. Managed settings at `/Library/Application Support/ClaudeCode/managed-settings.json` with `"allowedProviders": ["customEndpoint"]` make Claude Code refuse any other endpoint [A17][A18]. Not verified: whether these settings cover every outbound call.

### OpenCode

OpenCode moved from `sst/opencode` to `anomalyco/opencode`. Version 2 renamed many config keys and still accepts the V1 names [O1][O4].

**MCP.** V2 puts servers under `mcp.servers.<name>` in `opencode.json(c)` (global, project, or `.opencode/`). Local servers take `type: "local"`, `command` (array), `environment` and `disabled`. The `protocol` key defaults to `legacy`, which speaks revisions up to 2025-11-25. Timeouts are `startup` 30 s, `catalog` 30 s and `execution` 12 h [O5]. OpenCode exposes resources through built-in tools, turns prompts into `/<server>:<prompt>` commands, and supports elicitation. By default agents reach MCP tools through Code Mode, a JavaScript `execute` tool. `codemode: false` lists tools directly [O5][O7]. Permission rules on `<server>_<tool>` allow or deny single tools [O8].

**Instructions and skills.** OpenCode loads every `AGENTS.md` from the workspace up to `~` and falls back to `CLAUDE.md` per directory [O10]. Skills come from `.opencode/skills`, `.claude/skills` and `.agents/skills` in the project and the matching global folders [O11].

**Events.** A V2 plugin (`@opencode/plugin`) can subscribe to `filesystem.changed` and call the app with `fetch`. The app can also subscribe to the same event stream through `opencode serve` and `@opencode/client` without a plugin [O13][O19]. Not verified by running.

**Model routing.** A custom provider with the OpenAI-compatible package points at LiteLLM, and a `provider.use` policy blocks every other provider, including OpenCode's built-in hosted models [O9][O14][O15]:

```jsonc
{
  "model": "litellm/local-coder",
  "agents": { "title": { "model": "litellm/local-coder" } },
  "providers": { "litellm": {
    "package": "@opencode/ai/providers/openai-compatible",
    "settings": { "baseURL": "http://127.0.0.1:4000/v1", "apiKey": "{env:LITELLM_API_KEY}" },
    "models": { "local-coder": { "modelID": "local-coder",
      "capabilities": { "tools": true }, "limit": { "context": 131072, "output": 8192 } } } } },
  "policies": [
    { "action": "provider.use", "resource": "*", "effect": "deny" },
    { "action": "provider.use", "resource": "litellm", "effect": "allow" } ],
  "websearch": false
}
```

Session titles go to `agents.title.model` or to a model of the session's provider, so they stay on LiteLLM [O1 `packages/core/src/session/context.ts:99-116`]. Non-model traffic: the model catalog from `models.opencode.ai` (`OPENCODE_DISABLE_MODELS_FETCH=1`) and the update check (`"update": "disable"` in the global config). Both env vars are in the source only, not the docs [O1 `packages/cli/src/server-process.ts:117-121`][O3]. No analytics code was found in `core`, `cli` or `server` [O1].

### GitHub Copilot

The Copilot extension now lives in `microsoft/vscode` under `extensions/copilot`, MIT. VS Code agent mode has several harnesses. The Copilot harness runs on the Copilot SDK. The older Local harness "will be removed by February 2027" [G2][G6]. Answers differ by harness.

**MCP.** Copilot CLI reads `~/.copilot/mcp-config.json`, `.mcp.json` from the working directory up to the repo root, and `.github/mcp.json`. It does not read `.vscode/mcp.json` [G14]. VS Code's Copilot harness reads `~/.copilot/mcp-config.json` and `.mcp.json` with the key `mcpServers`. The Local harness reads `.vscode/mcp.json` with the key `servers` [G10]. The Copilot harness supports tools, elicitation and sampling, but no prompts and no roots, and uses resources only for MCP Apps [G11]. The "MCP servers in Copilot" org policy can switch MCP off for Business and Enterprise users [G9]. Copilot CLI does not pass tool output above 20,480 bytes to the model directly [G16].

**Instructions and skills.** Copilot CLI reads `AGENTS.md`, `CLAUDE.md`, `.github/copilot-instructions.md` and `.github/instructions/**/*.instructions.md` [G17]. Copilot CLI finds skills in `.github/skills/`, `.claude/skills/` and `.agents/skills/`. VS Code's Copilot harness uses `.github/skills/` and `~/.copilot/skills/` [G20][G21].

**Hooks.** Copilot CLI and VS Code (preview) read `.github/hooks/*.json`. A `postToolUse` entry runs a command with `toolName` and `toolArgs` on stdin [G22][G23].

**Model routing in Copilot CLI.** `~/.copilot/providers.json` (or `COPILOT_PROVIDER_BASE_URL` and the other `COPILOT_PROVIDER_*` variables) points the CLI at LiteLLM. With `COPILOT_OFFLINE=true`, the docs say "No GitHub authentication is attempted", "The CLI only makes network requests to your configured BYOK provider" and "Telemetry is fully disabled" [G15][G26]. BYOK needs no Copilot subscription [G7].

```json
{ "providers": [ { "name": "litellm", "type": "openai",
    "baseUrl": "http://127.0.0.1:4000/v1", "apiKey": "<LiteLLM key>", "wireApi": "completions" } ],
  "models": [ { "id": "local-coder", "provider": "litellm",
    "maxPromptTokens": 100000, "maxOutputTokens": 8192 } ] }
```

**Model routing in VS Code.** "Chat: Manage Language Models" adds a Custom Endpoint, stored in `chatLanguageModels.json` with `vendor: "customendpoint"` and `apiType: "chat-completions"`. The older `github.copilot.chat.customOAIModels` setting is deprecated [G8]. With a Copilot token present, utility tasks (titles, commit messages, intent detection, summaries) use GitHub's own utility model unless `chat.utilityModel` and `chat.utilitySmallModel` point at the custom model. Semantic search, inline completions and embeddings never use BYOK models [G8]. BYOK works without GitHub sign-in in the Local harness and in the Agents window [G8][G25]. VS Code has no equivalent of `COPILOT_OFFLINE`, so signing in later sends utility and embedding calls to GitHub again.

## 3. Holding agents to the model policy

ADR-0007 lets an agent touch a project memory only if its model location is within the model policy. The app cannot observe an agent's model traffic, and an MCP client's self-reported name proves nothing. What the sources support:

1. **The app generates the agent configuration.** For each agent it writes a dedicated configuration (a `CODEX_HOME`, OpenCode global config with `policies`, Copilot `providers.json` plus `COPILOT_OFFLINE=true`, Claude Code managed settings with `allowedProviders: ["customEndpoint"]`) that sends model calls only to LiteLLM, and a launcher starts the agent with it.
2. **The app checks the routes.** It reads the configured LiteLLM model name and that route's `model_info.model_location` from `/model/info`, as [Pick the local model server and LiteLLM setup](https://github.com/Niko4417/OPT-cc/issues/3) proposed, and refuses to start the agent on a project memory whose model policy the route exceeds.
3. **A repository cannot override the routing in Codex,** because project config cannot set providers [C4]. OpenCode's project config can set providers, so the `provider.use` deny policy belongs in the global or system config at `/Library/Application Support/opencode/` [O3][O9]. Claude Code ignores project settings for the gateway login and honors managed settings first [A5][A17].
4. **The plain folder cannot enforce anything itself.** Any agent the consultant starts outside the launcher can read it. This is policy by configuration. A stricter control would encrypt the folder at rest, which conflicts with agents reading it as plain files.

Two policy cases follow. Under the default on-device-only policy, the agent must run on a local model behind LiteLLM. Under a policy that allows third-party cloud, the agent's own vendor model is a third-party cloud route, and the configuration only needs to name it as one.

## 4. MCP server design for all four agents

- **Transport.** stdio, because every agent supports it and it needs no OAuth or Origin checks. The stdio process is a thin shim that forwards to the app's API on 127.0.0.1. One long-lived Streamable HTTP server would also work, but OpenCode turns OAuth on by default for remote servers, and the spec requires Origin validation [O5][M3].
- **Protocol revision.** The server must answer 2025-06-18 (Codex) and the revisions up to 2025-11-25 (OpenCode `legacy`). The Python SDK `mcp` 2.3.0 (MIT) speaks both 2026-07-28 and the legacy protocol [M7]. Keiko's `packages/keiko-mcp` negotiates only 2026-07-28 (Keiko `docs/adr/ADR-0181-shared-managed-mcp-host.md`), so it would not talk to Codex 0.170.0 as is. Not verified by connecting.
- **Tools only.** Copilot's harness has no prompts and no general resources, and Codex ignores `list_changed`, so every capability is a tool with a fixed list. Suggested tools: `search` (hybrid retrieval, returns cited snippets), `read` (one source by citation or record id, paged), `decision_register` (records with mapped status, supersession and open decision conflicts) and `index_status`. All carry `readOnlyHint: true`, so Codex runs them without prompting [C4].
- **Output size.** Under 8 KB per result with paging, below Codex's 10,000-byte fallback for local models and Copilot CLI's 20,480-byte threshold [C4][G16].
- **Writes.** No write tool in V1. Agents edit files in the folder. A later `propose_record` tool could validate front matter before writing, if agents produce invalid records too often.

## 5. Getting edits indexed back

| Trigger | Catches | Misses |
| --- | --- | --- |
| Agent hook (Codex `PostToolUse` on `apply_patch`, Claude Code `PostToolUse` on `Edit\|Write`, OpenCode `filesystem.changed`, Copilot `postToolUse`) | Edits by the agent's own edit tool, within seconds | Shell commands, other editors, edits while the app is down |
| Folder watcher | Every write while the app runs | Coalesced or dropped events, editors that write by rename; edits while the app is down |
| Hash rescan on start and on a timer | Everything, including edits while the app was down | Nothing, but slower |

macOS FSEvents may coalesce or drop events. On `kFSEventStreamEventFlagMustScanSubDirs` the app must rescan, and Apple calls the event list "advisory" [M27]. The watcher libraries document the same: `watchdog` 7.1.0 (Apache-2.0) misses on-modified events from Vim's write-by-rename [M24], `chokidar` 6 (MIT) merges an unlink plus add within 100 ms into one change [M25], and `@parcel/watcher` 2.6.0 (MIT) reports a rename as delete plus create and can replay events since a snapshot after a restart [M26]. So the app runs a watcher with a short debounce plus a content-hash rescan at start and every few minutes, as claude-context and ChunkHound do (section 6). Because the folder is a git repo, `git status` and commit SHAs give a cheap change list at start. Hooks are an optional fast path, not the mechanism.

On re-index, a changed decision record goes through the same reader as ADR-0001 describes. A new `supersedes` link or a status change updates the decision register, and a new decision conflict waits for a human ruling.

## 6. How existing tools expose a knowledge base

| Server, version | License | Plain Markdown is the truth | Re-index after a file edit | Search | Fully local |
| --- | --- | --- | --- | --- | --- |
| `memory` reference server, 2.5.0 [M9] | MIT, moving to Apache-2.0 | No, one JSONL graph | No index | Substring | Yes |
| `filesystem` reference server, 2.5.0 [M10] | same | Yes | No index | File names only | Yes |
| basic-memory 1.0.0 [M11] | AGPL-3.0, fails the license rule | Yes, two-way | File watcher | Full text plus vectors | Yes |
| qmd 2.9.0 [M12] | MIT | Yes; MCP side read-only | Manual `qmd update` and `qmd embed` | BM25, vectors, rank fusion, rerank | Yes, embeddings via an OpenAI-compatible URL |
| docs-mcp-server 3.4.1 [M13] | MIT | Indexes folders into its own store | `refresh` skips unchanged pages | Full text, optional vectors | Yes; telemetry on by default |
| claude-context [M14] | MIT | Code files | Merkle sync every 5 minutes plus a trigger file | BM25 plus dense | Needs a Milvus server |
| ChunkHound 6.1.0 [M19] | MIT | Code files | `watchdog` watcher, 0.5 s debounce | Regex plus semantic | Yes |
| Serena 2.1.0 [M20] | GPL-3.0, fails the license rule | Yes, memory notes | No index | LSP symbols | Yes |
| Cognee MCP 1.6.9, Graphiti MCP 0.31.2 [M15][M17] | Apache-2.0 | No, content goes in through tools | None | Graph plus vectors | Needs an LLM to build the graph; Graphiti needs FalkorDB or Neo4j |
| Context7 [M18] | MIT client, cloud backend | No | Not applicable | Not applicable | No |

Most of them document setup for several of our agents: docs-mcp-server for Claude Code, OpenCode and Copilot CLI; ChunkHound for Claude Code, Codex, OpenCode and VS Code [M13][M19]. All expose tools, not resources. Only the `memory` server documents a resource [M9]. None of the candidate stacks offers its own index to outside agents. Keiko's MCP package relays tools into Keiko's own coding runs and serves no Knowledge Pods. LightRAG has no MCP server. Open WebUI is an MCP client only [M21][M22]. qmd is the closest match to the recommended design, with Markdown as the truth, a read-only MCP server and local embeddings, and it is MIT. Its manual `update` step is what the project memory replaces with a watcher.

## 7. Standards the folder relies on

- **Agent Skills** (agentskills.io, Apache-2.0). A skill is `skill-name/SKILL.md` with required `name` (1 to 64 characters, lowercase letters, digits and hyphens, matching the folder) and `description` (1 to 1,024 characters). The cross-client paths are `.agents/skills/` and `~/.agents/skills/`. The spec lists Claude Code, Codex, OpenCode, GitHub Copilot and VS Code as clients [M29][M30][M31]. Claude Code and VS Code's Copilot harness do not read `.agents/skills/`, hence the three copies.
- **`AGENTS.md`** (agents.md, MIT, stewarded by the Agentic AI Foundation). The nearest file in the directory tree wins. The site lists Codex, OpenCode, VS Code and the Copilot coding agent. It does not list Claude Code, which reads the file anyway since 2.1.277 [M32][A9].
- **MCP** 2026-07-28 made the protocol stateless and deprecated roots, sampling and logging. Servers must validate the `Origin` header and should bind to 127.0.0.1 when running locally [M1][M3][M4].

## Open points

- **Context size for on-device agents.** [Check that the lab models fit in 64 GB with embedding and reranking](https://github.com/Niko4417/OPT-cc/issues/4) capped prompts at 32k tokens and output at 8,192 tokens for the lab pair. Coding agents send large system prompts and tool lists, and Claude Code needs about 100k context. A prototype should measure one agent session on `mlx_lm.server` with a larger cap and check that it still fits next to the app's own models.
- **LiteLLM bridges.** Not tested: Codex's Responses calls through LiteLLM's chat bridge to `mlx_lm.server`, and Claude Code's Messages calls through LiteLLM's translation, including tool calls and streaming.
- **Claude Code license.** Proprietary, under Anthropic's Commercial Terms. The owner decides whether this meets the license rule.
- **Copilot in VS Code.** The documented no-sign-in path is the Local harness, which is removed by February 2027. Whether the Copilot harness honors an offline mode is not verified.
- **Skill copies.** Whether symlinks instead of copies work in each agent's skill folder is not verified.
- **Folder name and location.** The owner names the folder and decides whether one folder serves one project memory.

## Sources

Codex CLI (`openai/codex` at tag `rust-v0.170.0`; paths relative to `codex-rs/`):

- C1: https://github.com/openai/codex/releases/tag/rust-v0.170.0
- C2: https://github.com/openai/codex/blob/rust-v0.170.0/LICENSE
- C3: https://learn.chatgpt.com/docs/extend/mcp?surface=cli (developers.openai.com/codex redirects here)
- C4: https://github.com/openai/codex/tree/rust-v0.170.0/codex-rs, files and lines as cited
- C5: https://learn.chatgpt.com/docs/config-file/config-reference
- C6: https://learn.chatgpt.com/docs/agent-configuration/agents-md
- C7: https://learn.chatgpt.com/docs/build-skills
- C8: https://learn.chatgpt.com/docs/hooks

Claude Code:

- A1: https://github.com/anthropics/claude-code/releases/tag/v2.1.327
- A2: https://github.com/anthropics/claude-code/blob/main/LICENSE.md
- A3: https://code.claude.com/docs/en/legal-and-compliance and https://www.anthropic.com/legal/commercial-terms
- A5: https://code.claude.com/docs/en/llm-gateway-connect
- A6: https://code.claude.com/docs/en/llm-gateway
- A7: https://code.claude.com/docs/en/mcp
- A8: https://code.claude.com/docs/en/env-vars
- A9: https://code.claude.com/docs/en/memory
- A10: https://code.claude.com/docs/en/skills
- A11: https://code.claude.com/docs/en/hooks
- A12: https://code.claude.com/docs/en/llm-gateway-protocol
- A15: https://code.claude.com/docs/en/model-config
- A16: https://code.claude.com/docs/en/data-usage
- A17: https://code.claude.com/docs/en/managed-settings
- A18: https://code.claude.com/docs/en/settings-reference

OpenCode (`anomalyco/opencode` at tag `v2.3.2`, commit `c0ad18b4`):

- O1: https://github.com/anomalyco/opencode/tree/v2.3.2, files and lines as cited, and its `LICENSE`
- O2: https://github.com/anomalyco/opencode/releases
- O3: https://opencode.ai/docs/config/
- O4: https://opencode.ai/docs/migrate-v1/
- O5: https://opencode.ai/docs/mcp-servers/
- O7: https://opencode.ai/docs/tools/
- O8: https://opencode.ai/docs/agents/
- O9: https://opencode.ai/docs/policies/
- O10: https://opencode.ai/docs/instructions/
- O11: https://opencode.ai/docs/skills/
- O13: https://opencode.ai/docs/build/plugins/server/
- O14: https://opencode.ai/docs/providers/
- O15: https://opencode.ai/docs/models/
- O19: https://opencode.ai/docs/build/client/

GitHub Copilot:

- G1: https://github.com/microsoft/vscode/releases/tag/1.147.0 and its `LICENSE.txt`
- G2: https://github.com/microsoft/vscode/blob/1.147.0/extensions/copilot/package.json and `extensions/copilot/LICENSE.txt`
- G4: https://github.com/github/copilot-cli/releases/tag/v1.0.114
- G5: https://github.com/github/copilot-cli/blob/main/LICENSE.md
- G6: https://code.visualstudio.com/docs/agents/run/agent-harnesses
- G7: https://docs.github.com/en/copilot/concepts/models/bring-your-own-key
- G8: https://code.visualstudio.com/docs/copilot/customization/language-models (source: `microsoft/vscode-docs` at `c70b98c6`, `docs/agent-customization/language-models.md`)
- G9: https://docs.github.com/en/copilot/concepts/mcp-management
- G10: https://code.visualstudio.com/docs/agents/reference/mcp-configuration
- G11: https://code.visualstudio.com/api/extension-guides/ai/mcp
- G12: https://github.com/microsoft/vscode/blob/1.147.0/extensions/copilot/src/platform/configuration/common/configurationService.ts (lines 640, 1153)
- G14: https://docs.github.com/en/copilot/how-tos/copilot-cli/customize-copilot/add-mcp-servers
- G15: https://docs.github.com/en/copilot/how-tos/copilot-cli/set-up-copilot-cli/authenticate-copilot-cli
- G16: https://docs.github.com/en/copilot/reference/copilot-cli-reference/cli-command-reference
- G17: https://docs.github.com/en/copilot/how-tos/copilot-cli/customize-copilot/add-custom-instructions
- G18: https://code.visualstudio.com/docs/copilot/customization/custom-instructions
- G20: https://docs.github.com/en/copilot/how-tos/copilot-cli/customize-copilot/add-skills
- G21: https://code.visualstudio.com/docs/copilot/customization/agent-skills
- G22: https://docs.github.com/en/copilot/reference/hooks-configuration
- G23: https://code.visualstudio.com/docs/copilot/customization/hooks
- G25: https://github.com/microsoft/vscode-docs/blob/c70b98c6db768eceb7c2a233f0d281ce5f6973ed/docs/agents/run/agents-window-configuration.md
- G26: https://docs.github.com/en/copilot/reference/copilot-cli-reference/cli-config-dir-reference

LiteLLM:

- L1: https://docs.litellm.ai/docs/proxy/client_setup/codex_cli
- L2: https://docs.litellm.ai/docs/tutorials/openai_codex
- L3: https://docs.litellm.ai/docs/response_api
- L4: https://docs.litellm.ai/docs/proxy/client_setup/claude_code
- L5: https://docs.litellm.ai/docs/tutorials/claude_non_anthropic_models

MCP, existing servers and file watching:

- M1: https://modelcontextprotocol.io/specification/versioning
- M3: https://modelcontextprotocol.io/specification/2026-07-28/basic/transports/streamable-http
- M4: https://modelcontextprotocol.io/specification/2026-07-28/deprecated
- M7: https://github.com/modelcontextprotocol/python-sdk/releases/tag/v2.3.0 and its `LICENSE`
- M9: https://github.com/modelcontextprotocol/servers/blob/c799c8552b/src/memory/README.md
- M10: https://github.com/modelcontextprotocol/servers/blob/c799c8552b/src/filesystem/README.md
- M11: https://github.com/basicmachines-co/basic-memory (tag v1.0.0, `src/basic_memory/index/watch_service.py`)
- M12: https://github.com/tobi/qmd/blob/v2.9.0/README.md
- M13: https://github.com/arabold/docs-mcp-server (v3.4.1, `docs/guides/`)
- M14: https://github.com/zilliztech/claude-context/blob/64a87f9b31/packages/mcp/README.md and `packages/mcp/src/sync.ts`
- M15: https://github.com/topoteretes/cognee/tree/v1.6.9/cognee-mcp
- M17: https://github.com/getzep/graphiti/blob/v0.31.2/mcp_server/README.md
- M18: https://github.com/upstash/context7/blob/master/README.md
- M19: https://github.com/chunkhound/chunkhound (v6.1.0, `chunkhound/services/realtime/service.py`)
- M20: https://github.com/oraios/serena (v2.1.0, `LICENSE`)
- M21: https://github.com/HKUDS/LightRAG/blob/bb08ff8504/docs/FileProcessingPipeline.md
- M22: https://docs.openwebui.com/features/extensibility/mcp
- M24: https://github.com/gorakhargosh/watchdog/blob/v7.1.0/README.rst
- M25: https://github.com/paulmillr/chokidar/blob/6.0.0/README.md
- M26: https://github.com/parcel-bundler/watcher/blob/v2.6.0/README.md
- M27: https://developer.apple.com/library/archive/documentation/Darwin/Conceptual/FSEvents_ProgGuide/UsingtheFSEventsFramework/UsingtheFSEventsFramework.html
- M29: https://agentskills.io/specification
- M30: https://github.com/agentskills/agentskills/blob/83021e2be4/docs/client-implementation/adding-skills-support.mdx
- M31: https://github.com/agentskills/agentskills/blob/83021e2be4/docs/snippets/clients.jsx
- M32: https://agents.md/

Keiko (a local checkout at `b5fe2a84f`, GitHub `dev` at `8701b820ff`): `packages/keiko-mcp/src/relay-endpoint.ts`, `packages/keiko-server/src/coding-sidecar-mcp-relay.ts`, `docs/adr/ADR-0181-shared-managed-mcp-host.md`.
