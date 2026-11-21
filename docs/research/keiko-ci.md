# Keiko's CI and the anti-slop epic: what the project memory should copy

Research for the owner's question on replicating Keiko's CI. Sources were read on 2026-11-21. It extends [`docs/research/quality-gates.md`](https://github.com/Niko4417/OPT-cc/blob/dd07ded4215cb860af3c1a1b405b95079db3c0b7/docs/research/quality-gates.md) (branch `research/quality-gates`), which covered only Keiko's 8 grounding and retrieval gates and its model-backed labs. That note said, without evidence, that most other Keiko CI jobs "serve a large product repo and are more than a glue repo needs". This note checks that claim job by job.

Keiko links point at commits of [oscharko-dev/Keiko](https://github.com/oscharko-dev/Keiko) (Apache-2.0): `dev` at [`4424cea`](https://github.com/oscharko-dev/Keiko/commit/4424cea62707f830296b9685866a558419a4e967) (2026-11-20) and the epic branch `codex/epic-anti-slop-quality` at [`45fbce4`](https://github.com/oscharko-dev/Keiko/commit/45fbce4345e68ddc873b09177a2a7220ba0149d2) (2026-11-21). Run times come from the GitHub Actions API on 2026-11-21. Three kinds of statement appear below and are kept apart: "documented" (a file, issue or paper says so), "measured" (I read the number from runs or computed it from files), and "judgment" (my reading, marked as such).

## Question

Should the project memory replicate Keiko's whole CI as it is on GitHub today? Should it replicate the stricter quality gates of Keiko's anti-slop epic, given that those are TypeScript-specific? Which gates fit a public repository with one owner, glue code of a few thousand to tens of thousands of lines, a stack that is not chosen yet, and code written mostly by coding agents?

## Short answer

- Do not replicate Keiko's whole CI. A green pull request to `dev` takes 53 minutes and 392 job-minutes over 32 jobs (measured, run 44897203715), and the median over 53 recent runs is 58 minutes. 12 of its 22 workflows test or ship Keiko's own product, and its CI calls 83,495 lines of Node.js scripts. Three scheduled lanes, among them the mutation suite, have had no green run for 2 to 4 months, and nothing blocked on them.
- Copy the language-independent part, from the first commit. That part is the Gitleaks scan of the pull-request range, dependency review, OSV-Scanner, actionlint, zizmor, a check that every action is pinned to a SHA, CodeQL and one fail-closed `ci` aggregate. Together they take a few minutes and are free on a public repository. None of the small comparable projects runs most of them, but the agent studies point at them: 19.7% of model-recommended packages do not exist, and 25% to 40% of generated code had security weaknesses in the samples studied.
- The anti-slop rules are not on Keiko `dev`. They exist only on the epic branch, which is 235 commits ahead of `dev` with no pull request into it, because the owner postponed the rollout on 2026-11-03. The engine is Oxlint with a third-party plugin of 22 syntactic rules. A type-aware qualification job adds about 84 runner-minutes per pull request. After 42 days and 51 pull requests, the register of tolerated findings (46,999) is still above its starting baseline (46,618).
- Most of that cost comes from the 46,618 legacy findings, so a new code base that starts at zero findings avoids it. For TypeScript glue code, adopt the production rules and `no-module-mocking` from the first commit without a register. Turn on the 4 boundary rules too, with one schema-parsing boundary module exempt. For Python glue code, the nearest equivalents are mypy or basedpyright in strict mode, Ruff `ANN401`, `TID251` and `PGH003`, and an import-linter contract. They are weaker in three places: `object`-typed values, justifications for casts and untyped adopted apps.
- The evidence for these gates is thin. A type checker flags about 15% of fixed public bugs (JavaScript and Python studies). Mutation testing has the best fault-coupling evidence (about 70% of real faults), but the validated use shows filtered mutants on changed code in review, not a nightly score. No study measures stricter type rules, custom lint rules, duplication gates or ratchets against shipped defects. Coding agents cheated on up to 76% of tasks whose tests contradict the specification, and Claude models did so mainly by editing the tests, so a gate that surfaces changed tests to the owner has more direct support than any lint rule.

## 1. Keiko's CI on `dev`

### 1.1 Required checks and branch protection

Documented. `dev` is protected, with enforcement for everyone, and requires 10 status checks ([branches API](https://api.github.com/repos/oscharko-dev/Keiko/branches/dev), field `protection.required_status_checks`). The full protection settings (reviews, linear history, signed commits) need admin rights and returned 404, so this note relies on the `ci.yml` comments for those: "`dev` is protected with linear history and signed squash merges" ([`ci.yml` lines 182 to 194](https://github.com/oscharko-dev/Keiko/blob/4424cea62707f830296b9685866a558419a4e967/.github/workflows/ci.yml#L182-L194)). No repository ruleset applies to `dev`: the 3 rulesets protect tags or are disabled ([rulesets API](https://api.github.com/repos/oscharko-dev/Keiko/rulesets)).

| Required check | Producer | Needs |
| --- | --- | --- |
| `ci` | Aggregate job of `ci.yml`; fails closed unless its 13 needed jobs succeeded or were skipped for a listed reason | Nothing beyond the jobs it aggregates |
| `Build, scan, SBOM, smoke` | Job of `ci.yml` | Nothing paid |
| `ui` | Job of `ci.yml` | Nothing paid |
| `Analyze (actions)`, `Analyze (javascript-typescript)` | `codeql.yml` | Free on public repositories (section 5) |
| `Review dependency diff (dev/main)` | `dependency-review.yml` | Dependency graph; free on public repositories |
| `workflow hygiene` | `workflow-hygiene.yml` | Nothing paid |
| `SonarCloud Code Analysis` | SonarQube Cloud app, fed by the `coverage-sonar` job | Secret `SONAR_TOKEN`; a SonarQube Cloud organisation |
| `Socket Security: Project Report`, `Socket Security: Pull Request Alerts` | Socket GitHub app | An installed third-party app |

Measured. On a green pull request to `dev` ([PR "feat(memory): MemoriaViva keeps history and recalls without chance, one measured step per push"](https://github.com/oscharko-dev/Keiko/pull/4481), head `77d2bfe`), the commit also carried a CodeRabbit check (skipped) and a GitHub Advanced Security "CodeQL" check, so two review bots run beside the workflows ([check runs](https://api.github.com/repos/oscharko-dev/Keiko/commits/77d2bfe08/check-runs)).

### 1.2 Jobs and gates inside `ci.yml`

[`ci.yml`](https://github.com/oscharko-dev/Keiko/blob/4424cea62707f830296b9685866a558419a4e967/.github/workflows/ci.yml) has 1,866 lines and 17 jobs, which expand to 32 jobs per pull request through matrices. It runs on pull requests to `dev`, 14 integration branches and `release/**`, on pushes to the same branches, on merge-queue groups and by hand ([lines 3 to 50](https://github.com/oscharko-dev/Keiko/blob/4424cea62707f830296b9685866a558419a4e967/.github/workflows/ci.yml#L3-L50)). Every job runs on Node.js 26.13.1 and first runs `scripts/check-runtime-toolchain.mjs --exact`. Every third-party action is pinned to a 40-character SHA.

Run times are job times of the green run [44897203715](https://github.com/oscharko-dev/Keiko/actions/runs/44897203715) of that pull request to `dev` (measured). "Language" says whether the job depends on TypeScript or Node.js: "none" means the method and most of the code carry over to a Python repository.

| Job (lines) | What it checks | Tools and versions | Run time | Needs | Language |
| --- | --- | --- | --- | --- | --- |
| Protected branch gate ([74-121](https://github.com/oscharko-dev/Keiko/blob/4424cea62707f830296b9685866a558419a4e967/.github/workflows/ci.yml#L74-L121)) | The run targets a protected or integration branch | Shell | 0.1 min | Nothing | None |
| Change scope ([126-180](https://github.com/oscharko-dev/Keiko/blob/4424cea62707f830296b9685866a558419a4e967/.github/workflows/ci.yml#L126-L180)) | Classifies the diff (documentation only, Windows-relevant) so the cross-platform matrix can be skipped. The classifier is taken from the base commit, so a pull request cannot edit it | `check-documentation-only-change.mjs` | 0.6 min | Nothing | None as a method |
| Verified tree ([195-226](https://github.com/oscharko-dev/Keiko/blob/4424cea62707f830296b9685866a558419a4e967/.github/workflows/ci.yml#L195-L226)) | Skips the whole matrix on a push to `dev` when the squash commit has the same tree SHA as a pull-request head that already passed. The comment reports that "15 of the last 100 `dev` runs failed" on flaky infrastructure after about 48 minutes each, on trees already proven ([ADR-0178](https://github.com/oscharko-dev/Keiko/blob/4424cea62707f830296b9685866a558419a4e967/docs/adr/ADR-0178-reuse-proven-tree-evidence-on-integration-runs.md)) | `resolve-verified-tree-evidence.mjs`, Actions API | 0.3 min | `actions: read` token | None |
| Semantic duplication ([232-268](https://github.com/oscharko-dev/Keiko/blob/4424cea62707f830296b9685866a558419a4e967/.github/workflows/ci.yml#L232-L268)) | No new semantic clone groups in files changed since the base commit, against a shrink-only baseline of 839 groups | Fallow 3.35.0 | 1.0 min | Nothing | Tool is JavaScript/TypeScript only; method is not |
| Secret scan ([273-356](https://github.com/oscharko-dev/Keiko/blob/4424cea62707f830296b9685866a558419a4e967/.github/workflows/ci.yml#L273-L356)) | No secret in any commit of the pull-request range, and none in a synthetic squash commit of head tree onto base | Gitleaks 8.30.1 binary, SHA-256 pinned, output redacted | 0.6 min | Nothing | None |
| Core quality ([358-548](https://github.com/oscharko-dev/Keiko/blob/4424cea62707f830296b9685866a558419a4e967/.github/workflows/ci.yml#L358-L548)) | About 40 steps: merge-candidate consistency; sandbox egress proof; `typecheck` (native `tsc`, all packages); 13 repository hygiene checks (version consistency, dependency hygiene, `npm ls eslint`, dependency currency, OSV waiver scope, knip, browser baseline, external quality config, review-bot suppression, release workflow names, portable approvals, git client evidence, ADR index); `lint`; `prettier --check`; i18n guard; e2e wiring; the 8 grounding and retrieval gates of the earlier note; Activity Log gate; editor bundle gate; package surface and install smoke | TypeScript native preview, ESLint 10 with typescript-eslint 8.70.0 `strictTypeChecked`, knip 6.41.0, Prettier 3, dependency-cruiser 18 ([`package.json`](https://github.com/oscharko-dev/Keiko/blob/4424cea62707f830296b9685866a558419a4e967/package.json)) | 30.7 min, of which lint 16.3 min, Prettier 2.8 min, Activity Log 3.2 min, package smoke 2.7 min | 12 GB Node heap for lint | Mostly TypeScript-only; the hygiene checks are Keiko-specific |
| Coverage shards, packages 4, scripts 3, UI 1 ([557-830](https://github.com/oscharko-dev/Keiko/blob/4424cea62707f830296b9685866a558419a4e967/.github/workflows/ci.yml#L557-L830)) | Run the whole test suite with coverage in 8 parallel jobs; each job measures and none judges ([ADR-0156](https://github.com/oscharko-dev/Keiko/blob/4424cea62707f830296b9685866a558419a4e967/docs/adr/ADR-0156-measurement-and-verdict-separation.md)) | Vitest 4.1.12, V8 coverage, USearch runtime fetched with pinned SHA-256 | 8.6 to 36.2 min per shard, 185 job-minutes together | Workers capped at 3 GB because a shard was killed by the kernel at 17 GB | Tool TypeScript-only; method none |
| Coverage and SonarCloud ([839-1133](https://github.com/oscharko-dev/Keiko/blob/4424cea62707f830296b9685866a558419a4e967/.github/workflows/ci.yml#L839-L1133)) | Merges the shards, checks per-file coverage floors and release targets (UI 88, CLI 87), checks that changed sources appear in LCOV, runs a SonarQube Cloud scan with quality-gate wait, and verifies the scan log and the pull request gate | Sonar Scanner CLI 8.1.0.6389, SHA-256 pinned | 13.9 min, of which the scan 11.9 min | `SONAR_TOKEN`; SonarQube Cloud | Coverage method none; Sonar supports Python |
| Portable secure-read smoke, 4 platforms ([1135-1175](https://github.com/oscharko-dev/Keiko/blob/4424cea62707f830296b9685866a558419a4e967/.github/workflows/ci.yml#L1135-L1175)) | Compiles and qualifies a native helper on Linux, Windows, macOS arm64 and macOS x64 | Native toolchains | 0.7 to 5.5 min | macOS and Windows runners | Keiko product |
| `ci` aggregate ([1177-1322](https://github.com/oscharko-dev/Keiko/blob/4424cea62707f830296b9685866a558419a4e967/.github/workflows/ci.yml#L1177-L1322)) | One required context that fails closed on any failed, cancelled or unexpectedly skipped need | Shell | 0.1 min | Nothing | None |
| Build, scan, SBOM, smoke ([1324-1421](https://github.com/oscharko-dev/Keiko/blob/4424cea62707f830296b9685866a558419a4e967/.github/workflows/ci.yml#L1324-L1421)) | Build; install smoke of the packed package; `npm audit --audit-level=high --omit=dev` with retry; CycloneDX SBOM; workspace SBOM and licence gate; install-script approvals; CLI exit-code smoke | npm 11, `npm sbom` | 12.2 min | Nothing | Method none; commands npm-specific |
| Cross-platform smoke, Ubuntu, macOS, Windows ([1423-1582](https://github.com/oscharko-dev/Keiko/blob/4424cea62707f830296b9685866a558419a4e967/.github/workflows/ci.yml#L1423-L1582)) | Typecheck, build and install the product on each OS, plus Windows path, junction, spawn and setup proofs | .NET 8 and MSVC on Windows | 6.1, 19.2 and 27.7 min | macOS and Windows runners | Keiko product |
| Firefox smoke, 2 shards ([1584-1633](https://github.com/oscharko-dev/Keiko/blob/4424cea62707f830296b9685866a558419a4e967/.github/workflows/ci.yml#L1584-L1633)) | Release smoke journeys in Firefox | Playwright 1.61 | 17.1 and 15.8 min | Nothing | Keiko product |
| `ui` ([1635-1827](https://github.com/oscharko-dev/Keiko/blob/4424cea62707f830296b9685866a558419a4e967/.github/workflows/ci.yml#L1635-L1827)) | Performance evidence checks, UI audit, lint, typecheck with TypeScript 7, static export, Chromium and WebKit smoke journeys, 3 editor end-to-end suites, health smoke | Playwright, Next.js | 52.6 min, the longest job | Nothing | Keiko product |
| Attest trusted SBOM provenance ([1829-1866](https://github.com/oscharko-dev/Keiko/blob/4424cea62707f830296b9685866a558419a4e967/.github/workflows/ci.yml#L1829-L1866)) | On pushes only, attests the SBOM files | `actions/attest` 4.2.2 | Skipped on pull requests | `id-token: write` | None |

Documented. The scripts these jobs call are themselves a large code base: `scripts/` on `dev` holds 206 top-level `.mjs` files with 83,495 lines (measured with `wc -l` on the pinned checkout), not counting `scripts/lib` and the tests in `scripts/__tests__`.

### 1.3 The other 21 workflows

| Workflow | What it checks or does | Trigger | Required | Tools | Run time, median of recent runs (n) | Needs | Language |
| --- | --- | --- | --- | --- | --- | --- | --- |
| [`codeql.yml`](https://github.com/oscharko-dev/Keiko/blob/4424cea62707f830296b9685866a558419a4e967/.github/workflows/codeql.yml) | CodeQL for `actions` and `javascript-typescript`, `build-mode: none` | Pull request, push, merge group, weekly | Yes, 2 contexts | `codeql-action` 4.39.3 | 16.2 min on pull requests (53) | Free on public repositories | None; CodeQL supports Python |
| [`dependency-review.yml`](https://github.com/oscharko-dev/Keiko/blob/4424cea62707f830296b9685866a558419a4e967/.github/workflows/dependency-review.yml) | New dependencies: fail on moderate or worse advisories in development, runtime and unknown scopes; licence allowlist of 13 SPDX IDs; OpenSSF Scorecard shown | Pull request, merge group | Yes | `dependency-review-action` 5.0.0 | 0.6 min (60) | Dependency graph | None |
| [`workflow-hygiene.yml`](https://github.com/oscharko-dev/Keiko/blob/4424cea62707f830296b9685866a558419a4e967/.github/workflows/workflow-hygiene.yml) | actionlint; every `uses:` pinned to a 40-hex SHA (a grep); zizmor ignore anchors; branch lists agree across 3 workflows; zizmor; OSV-Scanner over all lockfiles | Pull request, push, merge group | Yes | actionlint 1.7.12 (SHA-256 pinned), zizmor 1.26.1, `osv-scanner-action` 2.7.0 | 0.7 min (53) | Nothing | None |
| [`osv-scanner.yml`](https://github.com/oscharko-dev/Keiko/blob/4424cea62707f830296b9685866a558419a4e967/.github/workflows/osv-scanner.yml) | OSV-Scanner over all lockfiles, with waivers in `osv-scanner.toml` | Daily | No | `osv-scanner-action` 2.7.0 | 0.5 min (60); 55 green, 5 red | Nothing | None |
| [`mutation-security.yml`](https://github.com/oscharko-dev/Keiko/blob/4424cea62707f830296b9685866a558419a4e967/.github/workflows/mutation-security.yml) | Stryker over 40 security-critical source files ([config](https://github.com/oscharko-dev/Keiko/blob/4424cea62707f830296b9685866a558419a4e967/stryker.security.conf.json): thresholds high 90, low 80, break 0); files an issue on failure | Daily | No | StrykerJS 10.1.1 with the Vitest runner | 3.7 min (60); **60 of 60 red** | `issues: write` | Tool TypeScript-only; method none |
| [`secret-scanning-queue.yml`](https://github.com/oscharko-dev/Keiko/blob/4424cea62707f830296b9685866a558419a4e967/.github/workflows/secret-scanning-queue.yml) | Every open GitHub secret-scanning alert has a recorded disposition in a document | Daily | No | Script on the secret-scanning API | 0.4 min (60); **60 of 60 red** | `security-events: read` | None |
| [`infra-failure-retry.yml`](https://github.com/oscharko-dev/Keiko/blob/4424cea62707f830296b9685866a558419a4e967/.github/workflows/infra-failure-retry.yml) | Re-runs a failed run of 5 workflows once, only when it matches a known runner-failure signature ([ADR-0161](https://github.com/oscharko-dev/Keiko/blob/4424cea62707f830296b9685866a558419a4e967/docs/adr/ADR-0161-bounded-re-run-for-infrastructure-signature-failures.md)) | `workflow_run` | No | Script | 0.4 min (60); 49 skipped | `actions: write` | None |
| [`e2e-extended.yml`](https://github.com/oscharko-dev/Keiko/blob/4424cea62707f830296b9685866a558419a4e967/.github/workflows/e2e-extended.yml) | 33 end-to-end suites in a matrix plus 3 code-task suites | Daily | No; "a nightly red is a signal to fix, not a merge blocker" | Playwright, Chromium | 18.1 min (58); **57 of 58 red** | Nothing | Keiko product |
| [`nightly-perf-evidence.yml`](https://github.com/oscharko-dev/Keiko/blob/4424cea62707f830296b9685866a558419a4e967/.github/workflows/nightly-perf-evidence.yml) | Committed performance evidence still matches the tree; files an issue on drift | Daily | No | Scripts | 1.1 min (60); 60 green | `issues: write` | Method none |
| [`code-task-real-binary.yml`](https://github.com/oscharko-dev/Keiko/blob/4424cea62707f830296b9685866a558419a4e967/.github/workflows/code-task-real-binary.yml) | Real OpenCode binary end-to-end test on macOS | Daily, by hand, called | No | macOS 14 runner | 5.3 min (58); 33 red, 16 cancelled | macOS runner | Keiko product |
| [`coding-workbench-customer-shape.yml`](https://github.com/oscharko-dev/Keiko/blob/4424cea62707f830296b9685866a558419a4e967/.github/workflows/coding-workbench-customer-shape.yml) | Installs and qualifies the staged Coding Workbench on Windows 2025, macOS 15 Intel and macOS 15 | Pull request with path filter (most of the code) | No | Native runners | 8.3 min (60) | macOS and Windows runners | Keiko product |
| [`npm-folder-qualify.yml`](https://github.com/oscharko-dev/Keiko/blob/4424cea62707f830296b9685866a558419a4e967/.github/workflows/npm-folder-qualify.yml) | Qualifies standalone npm folder packages on macOS and Windows | Pull request with path filter, by hand, called | No | Native runners | 1.2 min (14) | macOS and Windows runners | Keiko product |
| [`main-promotion.yml`](https://github.com/oscharko-dev/Keiko/blob/4424cea62707f830296b9685866a558419a4e967/.github/workflows/main-promotion.yml) | Only `dev` may open a pull request into `main` | Pull request to `main` | Not on `dev` | Shell | 0.1 min (12, last in June 2026) | Nothing | None |
| [`release-candidate.yml`](https://github.com/oscharko-dev/Keiko/blob/4424cea62707f830296b9685866a558419a4e967/.github/workflows/release-candidate.yml) | Decides the release candidate and moves the release tag | Push to `dev`, by hand | No | Scripts | 0.9 min (60) | GitHub App key, `release-tagging` environment | Keiko release |
| [`release.yml`](https://github.com/oscharko-dev/Keiko/blob/4424cea62707f830296b9685866a558419a4e967/.github/workflows/release.yml) | Requests, authorizes, qualifies on macOS 15 and publishes to npm | By hand | No | Scripts | 11.1 min (60) | GitHub App key, npm publish environment, signing key | Keiko release |
| [`release-advance.yml`](https://github.com/oscharko-dev/Keiko/blob/4424cea62707f830296b9685866a558419a4e967/.github/workflows/release-advance.yml) | Starts a requested release once its runs are green | `workflow_run` of 5 workflows | No | Script | 0.6 min (60); 53 skipped | Nothing | Keiko release |
| [`release-alignment.yml`](https://github.com/oscharko-dev/Keiko/blob/4424cea62707f830296b9685866a558419a4e967/.github/workflows/release-alignment.yml) | The published release is coherent | Daily | No | Script | 1.0 min (60) | Nothing | Keiko release |
| [`portable-assets.yml`](https://github.com/oscharko-dev/Keiko/blob/4424cea62707f830296b9685866a558419a4e967/.github/workflows/portable-assets.yml) | Stages, signs, qualifies and attests portable runtimes on 4 platforms | Push to `dev` and tags, by hand | No | Native runners | 21.8 min on push (44) | Signing environments, macOS and Windows runners | Keiko product |
| [`npm-folder-publish.yml`](https://github.com/oscharko-dev/Keiko/blob/4424cea62707f830296b9685866a558419a4e967/.github/workflows/npm-folder-publish.yml), [`coding-runtime-publish.yml`](https://github.com/oscharko-dev/Keiko/blob/4424cea62707f830296b9685866a558419a4e967/.github/workflows/coding-runtime-publish.yml) | Publish prerelease and runtime packages | By hand | No | Scripts | 5.9 min (1), 11.1 min (4) | npm publish environments | Keiko release |
| [`windows-gateway-confinement.yml`](https://github.com/oscharko-dev/Keiko/blob/4424cea62707f830296b9685866a558419a4e967/.github/workflows/windows-gateway-confinement.yml) | Windows network filter and relay confinement proofs | Push to 2 feature branches, by hand | No | Windows 2025 runner | 3.2 to 5.3 min | Windows runner | Keiko product |

### 1.4 Wall-clock time and size of one pull-request run

Measured on run [44897203715](https://github.com/oscharko-dev/Keiko/actions/runs/44897203715), a green pull request to `dev`:

- Wall clock 53.1 minutes, with 32 jobs and 392 job-minutes. The `ui` job (52.6 minutes) is the critical path.
- Over the last 53 pull-request runs of `ci.yml`, the median wall clock was 58.1 minutes and the 90th percentile 78.7 minutes. 26 of the 53 were cancelled by a newer push, 14 failed and 13 passed. CodeQL adds a parallel 16.2-minute median.
- Pull-request runs on the anti-slop epic branch took 65.5 and 76.2 minutes with 525 and 506 job-minutes, because they add the policy job of section 2 ([44880084548](https://github.com/oscharko-dev/Keiko/actions/runs/44880084548), [44881340006](https://github.com/oscharko-dev/Keiko/actions/runs/44881340006)).

### 1.5 Scheduled lanes that stay red

Measured over the last 60 scheduled runs of each workflow (2026-09-23 to 2026-11-21):

- `mutation-security.yml` failed 60 of 60 runs. The last green run was on 2026-07-14 ([29322877554](https://github.com/oscharko-dev/Keiko/actions/runs/29322877554)). The run of 2026-11-21 never mutated anything: Stryker stopped because "There were failed tests in the initial test run" ([log of 44891765307](https://github.com/oscharko-dev/Keiko/actions/runs/44891765307)). Its tracking issue ["Security mutation suite failed on dev"](https://github.com/oscharko-dev/Keiko/issues/3663) has been open since 2026-09-28 with 54 comments.
- `e2e-extended.yml` failed 57 of 58 runs. The last green run was on 2026-07-28 ([30341496686](https://github.com/oscharko-dev/Keiko/actions/runs/30341496686)).
- `secret-scanning-queue.yml` failed 60 of 60 runs, and the API returned no green run at all.

Judgment. A lane that is not a required check and stays red for months produces no signal. Keiko's own [ADR-0156](https://github.com/oscharko-dev/Keiko/blob/4424cea62707f830296b9685866a558419a4e967/docs/adr/ADR-0156-measurement-and-verdict-separation.md) records the same pattern for an earlier lane: "The nightly D12 regeneration lane failed 12 of 12 runs." A one-owner repository should add a scheduled lane only if someone will read it, and should prefer a per-pull-request gate on changed code.

## 2. The anti-slop epic

### 2.1 State of the epic

Documented. The epic is the open issue ["Epic: Enforce precise type safety and test fidelity across Keiko"](https://github.com/oscharko-dev/Keiko/issues/3914), created 2026-10-09, with 14 sub-issues: 3 closed and 11 open. The closed ones are ["Establish quality policy, clean guards and the epic CI delivery path"](https://github.com/oscharko-dev/Keiko/issues/3915), "Qualify all five Effect rules in the pinned native host" and ["Activate full enforcement and prove integrated production quality"](https://github.com/oscharko-dev/Keiko/issues/3991). Open ones include ["Define boundary policy and migrate the shared contracts leaf"](https://github.com/oscharko-dev/Keiko/issues/3918), ["Define test-seam policy and migrate backend, CLI and tooling tests"](https://github.com/oscharko-dev/Keiko/issues/3920), ["Migrate domain packages and their validated runtime ports"](https://github.com/oscharko-dev/Keiko/issues/3933) and ["Deliver anti-slop adoption to current dev through scoped PRs and drift control"](https://github.com/oscharko-dev/Keiko/issues/4145).

Measured. The epic branch is 235 commits ahead of `dev` and 31 behind, with merge base `b6c26036e` ([compare](https://github.com/oscharko-dev/Keiko/compare/dev...codex/epic-anti-slop-quality)). No pull request from the epic branch into `dev` exists.

Documented. The owner moved the target away from `dev` on 2026-11-03: "The rollout to `dev` is postponed to a date the owner sets later" ([ADR-0167 D6 on the epic branch](https://github.com/oscharko-dev/Keiko/blob/45fbce4345e68ddc873b09177a2a7220ba0149d2/docs/adr/ADR-0167-zero-cost-autonomous-quality-gates.md#d6--anti-slop-findings-ratchet-owner-decisions-2026-10-30-2026-11-03-and-2026-11-04)). The epic wrote no new ADR. It amended ADR-0167 (D3 and a new D6) and 9 other ADRs, and it added the policy document [`docs/qa/code-quality-policy.md`](https://github.com/oscharko-dev/Keiko/blob/45fbce4345e68ddc873b09177a2a7220ba0149d2/docs/qa/code-quality-policy.md). The test-seam and boundary policies exist only as issue text and as rule dispositions in [`scripts/code-quality-policy.json`](https://github.com/oscharko-dev/Keiko/blob/45fbce4345e68ddc873b09177a2a7220ba0149d2/scripts/code-quality-policy.json).

### 2.2 The rules and how they are implemented

Documented. The engine is not ESLint. It runs Oxlint 1.78.0 with the third-party plugin `oxlint-plugin-anti-slop` 0.1.2, installed from a GitHub tarball at commit `c44ef22c` ([`package.json` on the epic branch](https://github.com/oscharko-dev/Keiko/blob/45fbce4345e68ddc873b09177a2a7220ba0149d2/package.json)). The plugin's README says its rules "use Oxlint's ESTree and lexical-scope APIs rather than a TypeScript type checker" ([anti-slop README](https://github.com/dmmulroy/anti-slop/blob/c44ef22ca116d0ba62a3ff663a0bd13a3f3fa40b/README.md), MIT). So every rule is syntactic, with alias resolution inside one file only. Inline `oxlint-disable` comments and nested configs are rejected.

The type-aware part is separate. A "responsibility qualification" binder checks 133 records of validators that may keep findings, with the TypeScript compiler API (`ts.createProgram`, `getTypeChecker`) in [`scripts/lib/code-quality-compiler.mjs`](https://github.com/oscharko-dev/Keiko/blob/45fbce4345e68ddc873b09177a2a7220ba0149d2/scripts/lib/code-quality-compiler.mjs) and [`code-quality-symbols.mjs`](https://github.com/oscharko-dev/Keiko/blob/45fbce4345e68ddc873b09177a2a7220ba0149d2/scripts/lib/code-quality-symbols.mjs) (6,258 lines).

Rule descriptions paraphrase the upstream README. Disposition and active scopes come from the policy file, and counts from [`scripts/code-quality-register.json`](https://github.com/oscharko-dev/Keiko/blob/45fbce4345e68ddc873b09177a2a7220ba0149d2/scripts/code-quality-register.json), all at the epic head (counted by a delegated read of those files; I spot-checked the totals, not each row).

| Rule | What it forbids | Disposition | Active scopes | Remaining findings |
| --- | --- | --- | --- | --- |
| `no-chained-type-assertions` | Nested assertions such as `as unknown as T` | Production, required | 32 | 1,711 |
| `require-safety-comment-for-type-assertion` | A non-const assertion without a nearby `SAFETY` justification | Production, required | 26 | 13,335 |
| `no-widen-then-assert` | Widening a known value to `unknown`, `any` or `object`, then asserting back | Production, required | 6 | 25 |
| `no-reduce-accumulator-copy` | Copying the accumulator inside a reducer | Production, required | 5 | 0 |
| `no-reflect-apply` | `Reflect.apply` | Production, required | 4 | 53 |
| `no-reflect-get` | `Reflect.get` | Boundary-scoped | 33 | 138 |
| `no-unknown-returns` | Function contracts that return `unknown` or `Promise<unknown>` | Boundary-scoped | 30 | 1,242 |
| `no-object-parameters` | Parameters typed `object` | Boundary-scoped | 32 | 75 |
| `no-unknown-parameters` | Parameters typed `unknown`, except `cause` and predicate subjects | Boundary rule, inactive | 0 | 5,261 |
| `no-runtime-typeof` | Ad hoc `typeof` narrowing instead of parsing at a boundary | Boundary rule, inactive | 0 | 7,397 |
| `no-unsafe-dictionary-type` | Dictionaries whose values are `unknown`, `any`, `object` or `{}` | Boundary rule, inactive | 0 | 6,133 |
| `no-known-value-widening` | Known values flowing into `unknown`, `object` or open dictionaries | Boundary rule, inactive | 0 | 4,140 |
| `no-unknown-type-aliases` | Type aliases that resolve to `unknown` | Covered by an existing rule | 34 | 0 |
| `no-module-mocking` (the test-seam rule) | `vi.mock`, `jest.mock`, `doMock`, `unstable_mockModule` | Active in production, fixtures and declarations, not yet in tests | 4 | 920 |
| `no-array-filter-map` | An eager `.filter().map()` pair | Under review ([accumulator cost issue](https://github.com/oscharko-dev/Keiko/issues/3989)) | 12 | 257 |
| `no-conditional-empty-object-spread` | `...(c ? {x} : {})` | Under review | 6 | 6,182 |
| `oxc/no-accumulating-spread` | Spreading an accumulator in a reduce or a loop (a stock Oxlint rule) | Under review | 33 | 0 |
| 5 Effect rules | Patterns of the Effect library | Retired with the Node service host | 0 | 130 |

Documented. The 4 inactive boundary rules wait for ["Activate the four contracts boundary rules after epic #3914 integrates"](https://github.com/oscharko-dev/Keiko/issues/4278). Together they hold 22,931 of the remaining findings, about half.

Semantic duplication is a separate gate with its own tool. On the epic branch, [`check-semantic-duplication.mjs`](https://github.com/oscharko-dev/Keiko/blob/45fbce4345e68ddc873b09177a2a7220ba0149d2/scripts/check-semantic-duplication.mjs) runs Fallow 3.35.0 as `fallow audit --only dupes --gate new-only --base <sha>`: a clone group that exists at the base commit is inherited, and any group the change creates fails ([PR "compare semantic duplication with the base commit"](https://github.com/oscharko-dev/Keiko/pull/4189); ADR-0167 D3 amendment of 2026-11-01). Fallow analyses JavaScript, TypeScript, CSS and component files only ([Fallow README](https://github.com/fallow-rs/fallow)).

Not part of the epic but on `dev` already: ESLint with `tseslint.configs.strictTypeChecked` and `@typescript-eslint/no-explicit-any` as an error ([`eslint.config.js` lines 108 to 115](https://github.com/oscharko-dev/Keiko/blob/4424cea62707f830296b9685866a558419a4e967/eslint.config.js#L108-L115)), and `strict`, `noUncheckedIndexedAccess`, `exactOptionalPropertyTypes` and `noImplicitOverride` in [`tsconfig.base.json`](https://github.com/oscharko-dev/Keiko/blob/4424cea62707f830296b9685866a558419a4e967/tsconfig.base.json). Those are the "precise types" Keiko already had before the epic.

### 2.3 The ratchet

Documented in [ADR-0167 D6](https://github.com/oscharko-dev/Keiko/blob/45fbce4345e68ddc873b09177a2a7220ba0149d2/docs/adr/ADR-0167-zero-cost-autonomous-quality-gates.md) and the [policy document](https://github.com/oscharko-dev/Keiko/blob/45fbce4345e68ddc873b09177a2a7220ba0149d2/docs/qa/code-quality-policy.md):

- `npm run check:code-quality-policy -- --mode ratchet` scans the whole repository with all 22 rules and counts findings per file and rule.
- The counts are compared with the committed register. A higher count, or a new file with findings, fails. A lower count also fails until the register is shrunk with `--prune-register`, which "only lowers counts or removes entries and refuses to widen". CI never writes the register.
- A trial run showed that every delivery slice would fail a strict shrink-only register (+362 net: 1,762 findings added and 1,400 removed, [comment on the delivery issue](https://github.com/oscharko-dev/Keiko/issues/4145#issuecomment-6556628178)). So the owner added an increase ledger on 2026-11-01. A count may rise only to the exact `to` value of a ledger entry that the same pull request adds, with one of 5 reasons: `qualified-boundary`, `pending-boundary-migration`, `test-witness`, `proof-fixture` and `dev-cutoff-sync`.
- Known limitation, stated in the ADR: "counts are per file and rule, so fixing one finding and adding another of the same rule in the same file nets out."
- Rollout per package: on 2026-11-06 the owner decided that each rule turns on in every scope where it has zero findings, and the register keeps the rest. [PR "enforce the anti-slop policy in full on the epic branch"](https://github.com/oscharko-dev/Keiko/pull/4286) activated "180 scopes across 13 rules". Later slices such as [PR "clear and activate 14 more rule scopes across 12 packages"](https://github.com/oscharko-dev/Keiko/pull/4509) add scopes as packages reach zero.
- Measured (delegated read of the register at the epic head): all 28 packages still have register entries. The largest are `keiko-server` (21,923 findings), `keiko-ui` (6,135) and `keiko-contracts` (5,660).

### 2.4 Cost

Measured from `gh pr list` and `git diff` (delegated, spot-checked):

- 51 pull requests into the epic branch: 43 merged, 7 closed without merge and 1 open. None into `dev`.
- Calendar time: 42 days, from the first pull request on 2026-10-10 to the last merge on 2026-11-21. The epic is still open.
- Size: the epic's net diff from its merge base is 1,226 files, +176,170 and -61,344 lines. About 37,400 lines are policy, register and ledger JSON, about 26,500 lines are engine tests and about 11,850 lines are engine code in 12 `scripts/lib/code-quality-*.mjs` files.
- Owner effort is visible in the issues. Review rounds went past 100 ("round-101" in the [closing comment of the full-enforcement issue](https://github.com/oscharko-dev/Keiko/issues/3991#issuecomment-6737092073)), and one-time exceptions such as merges without hosted CI on 2026-11-06.

Measured from run job times (I read these runs myself):

| Run | Static code-quality policy job, Node 24 and Node 26 | Anti-slop ratchet step | Semantic duplication job | Job-minutes | Wall clock |
| --- | --- | --- | --- | --- | --- |
| Epic PR, [44880084548](https://github.com/oscharko-dev/Keiko/actions/runs/44880084548) | 45.2 and 39.6 min on `ubuntu-24.04-arm` | 0.7 min | 1.1 min | 525 | 65.5 min |
| Epic PR, [44881340006](https://github.com/oscharko-dev/Keiko/actions/runs/44881340006) | 45.8 and 38.1 min | 0.8 min | 1.0 min | 506 | 76.2 min |
| `dev` PR without the epic, [44897203715](https://github.com/oscharko-dev/Keiko/actions/runs/44897203715) | none | none | 1.0 min | 392 | 53.1 min |

So the ratchet itself costs under a minute per pull request. The full qualification job costs about 84 runner-minutes per pull request (two legs of 38 to 46 minutes). The epic also moved package coverage from 4 to 5 shards and raised the script shard limit from 30 to 45 minutes, because the engine "adds about 1,450 scripts tests" ([PR "land the anti-slop engine on the epic branch as a shrink-only ratchet"](https://github.com/oscharko-dev/Keiko/pull/4165)). The comparison above is between different pull requests on different bases, so the difference of about 120 job-minutes is an estimate, not a controlled measurement.

Type-aware ESLint is a separate cost. It peaks at about 12 GiB of memory, and "each contracts slice of #3914 adds to it" ([issue "Cut the type-aware ESLint memory back under the runner with a quarter of headroom"](https://github.com/oscharko-dev/Keiko/issues/4202)). A standard hosted Linux runner has 16 GB.

### 2.5 Results the epic reports

Documented:

| Point | Findings | Source |
| --- | --- | --- |
| First census | 41,084 raw matches in 6,183 files | [PR "qualify noncosmetic code-quality policy"](https://github.com/oscharko-dev/Keiko/pull/4008) |
| Baseline at `dev` snapshot `7bbca2e34` | 46,618 findings in 7,239 files | ADR-0167 D6 |
| After the `dev` 1.4.0 cutoff sync | 48,205, from 592 ledger raises in 246 files | ADR-0167 D6 |
| Epic head `45fbce4` | 46,999 in 4,468 files (measured from the register), with 1,139 ledger entries | Register and ledger at the epic head |

After 42 days the register is 381 findings above its starting baseline, because the ledger admitted 3,215 raised findings while slices removed others. Measured from the register, the peak was 48,553, so the net reduction from the peak is about 1,550 findings or 3.2%.

Defects found. Several slice pull requests list behaviour changes, each with a test that fails on the base and passes after the change. Examples: SQLite row readers in memory-vault and local-knowledge now refuse tampered or wrongly typed columns that an `as unknown as` cast let through ([PR 4509](https://github.com/oscharko-dev/Keiko/pull/4509), [PR "clear and activate four more rule scopes in slice 5"](https://github.com/oscharko-dev/Keiko/pull/4510)); a legacy-port call with non-object arguments used to settle as "completed" and now fails ([PR "type the harness assertions through their real invariants"](https://github.com/oscharko-dev/Keiko/pull/4368)); reads now ignore a polluted `Object.prototype` ([PR "remove 25 known-value widenings in contracts"](https://github.com/oscharko-dev/Keiko/pull/4328)). The epic reports no total defect count and no false-positive rate. PR 4328 reports one upstream false positive in `no-known-value-widening` that accounts for "45 of 4,210 widening findings". The ADR says: "A raw match is a diagnostic, not a proven defect."

Engine run time. The ratchet takes "about 30 seconds on an Apple-silicon laptop" for about 7,200 files ([policy document](https://github.com/oscharko-dev/Keiko/blob/45fbce4345e68ddc873b09177a2a7220ba0149d2/docs/qa/code-quality-policy.md)). The full qualification took 35:39 and 43:52 minutes on hosted runners (PR 4286).

### 2.6 What is on `dev` and what lives only on the epic branch

| Item | `dev` at `4424cea` | Epic branch at `45fbce4` |
| --- | --- | --- |
| Semantic duplication gate | Yes, from earlier work, with a shrink-only Fallow baseline of 839 groups ([`dupes-baseline.json`](https://github.com/oscharko-dev/Keiko/blob/4424cea62707f830296b9685866a558419a4e967/docs/qa/fallow/dupes-baseline.json)) and `--changed-since` | Rewritten to compare with the base commit; baseline deleted; no ignores |
| Oxlint and `oxlint-plugin-anti-slop` | Absent | Present |
| Engine, register, ledger, policy JSON | Absent | Present |
| `Static code-quality policy` job and ratchet step in `ci.yml` | Absent | Present |
| ADR-0167 D6 and policy document | Absent | Present |
| `strictTypeChecked`, `no-explicit-any`, strict `tsconfig` | Present, from before the epic | Present |

So none of the anti-slop rules runs on `dev` today.

## 3. What depends on TypeScript and what does not

### 3.1 Independent of language

These gates work the same in a Python repository. Most use the same tool.

| Gate | Keiko implementation | Same in Python |
| --- | --- | --- |
| Secret scan over the pull-request range | Gitleaks binary, SHA-256 pinned | Same tool |
| Dependency review | `dependency-review-action` with severity and licence policy | Same action; the dependency graph reads `requirements.txt`, `pyproject.toml` and `uv.lock`-style manifests (not checked for `uv.lock` here) |
| OSV-Scanner | `osv-scanner-action` in workflow hygiene and nightly | Same action; OSV covers PyPI |
| Workflow hygiene | actionlint, zizmor, a grep that every `uses:` is pinned to a 40-hex SHA | Same tools |
| CodeQL | `actions` and `javascript-typescript` | CodeQL supports `python` and `actions` |
| SonarQube Cloud | Scanner CLI with a quality-gate wait | Sonar supports Python |
| Socket | GitHub app | Socket supports PyPI (vendor page, not checked in depth) |
| Infrastructure re-run once | `check-infra-failure-signature.mjs` | Method carries over; the script is Node.js |
| Verified-tree reuse, merge-candidate check, fail-closed `ci` aggregate | Scripts and shell | Methods carry over; scripts are Node.js |
| Coverage as a measurement with per-file floors | Vitest V8 coverage and `check-package-coverage.mjs` | coverage.py with `fail_under`, or per-file checks in a script |
| Mutation testing as a method | StrykerJS | mutmut or Cosmic Ray (section 3.3) |
| Duplication detection as a method | Fallow, JavaScript/TypeScript only | jscpd or Pylint `duplicate-code` (section 3.3) |
| SBOM and provenance attestation | `npm sbom`, `actions/attest` | CycloneDX for Python, same attest action |
| Grounding and retrieval gates | Keiko scripts over Keiko packages | Method only, as the earlier note found |

### 3.2 TypeScript-only, or Keiko-only, with no use outside Keiko

- `ui`, Firefox smoke, cross-platform smoke, portable secure-read, portable assets, Windows gateway, code-task real binary, coding workbench, npm folder qualify and publish, and the release workflows. They test or ship Keiko's own product.
- The 13 hygiene checks in Core quality (version consistency, review-bot suppression, portable approvals, ADR index and so on). They enforce Keiko's own documents and release process.
- The anti-slop rules for TypeScript-only constructs: `no-array-filter-map`, `no-conditional-empty-object-spread` and the 5 Effect rules. Python has no direct counterpart of these patterns.

### 3.3 TypeScript-only, with a real Python equivalent

The Python column names the tool and the rule. Links go to the tool's documentation. "Weaker" is my judgment unless a source is named.

| Keiko gate or rule | Python equivalent | Where the Python version is weaker |
| --- | --- | --- |
| `tsc` with `strict`, `noUncheckedIndexedAccess`, `exactOptionalPropertyTypes` | mypy with `strict = true` ([mypy config](https://mypy.readthedocs.io/en/stable/config_file.html)), or Pyright/basedpyright with `typeCheckingMode = "strict"` or `"all"` ([basedpyright config](https://docs.basedpyright.com/latest/configuration/config-files/)) | Python types are not checked at run time, and the adopted apps (Open WebUI, LightRAG) are not fully typed, so `Any` leaks in through every call into them. I found no mypy flag equal to `exactOptionalPropertyTypes` |
| typescript-eslint `strictTypeChecked` | Ruff ([rules](https://docs.astral.sh/ruff/rules/)) for syntactic rules plus the type checker for type rules | Ruff rules are syntactic; type-aware lint rules such as unhandled promises have no Ruff counterpart. Pylint has some inference but is slow |
| `no-explicit-any` | Ruff `ANN401` ([any-type](https://docs.astral.sh/ruff/rules/any-type/)), mypy `disallow_any_explicit`, basedpyright `reportExplicitAny` | `ANN401` covers function annotations only; the mypy and basedpyright options cover all annotations |
| `no-unknown-returns`, `no-unknown-parameters`, `no-unsafe-dictionary-type`, `no-known-value-widening` | mypy `warn_return_any`, `disallow_any_expr`, `disallow_any_generics`; basedpyright `reportAny` and the `reportUnknown*` family | These catch `Any`, not `object`. Python has no separate `unknown` type, so `object` plays its role and no checker flags `dict[str, object]` |
| `no-object-parameters` | None found | No rule flags parameters typed `object` |
| `no-chained-type-assertions`, `no-widen-then-assert` | mypy `warn_redundant_casts`, basedpyright `reportUnnecessaryCast`; Ruff `TID251` ([banned-api](https://docs.astral.sh/ruff/rules/banned-api/)) can ban `typing.cast` outright | Only redundant casts are found. `cast(T, cast(object, x))` is not flagged unless `cast` is banned everywhere |
| `require-safety-comment-for-type-assertion` | For suppressions: Ruff `PGH003` ([blanket-type-ignore](https://docs.astral.sh/ruff/rules/blanket-type-ignore/)), mypy `warn_unused_ignores`, basedpyright `reportIgnoreCommentWithoutRule` and `reportUnnecessaryTypeIgnoreComment` | No tool requires a justification comment beside a `cast` |
| `no-reflect-get`, `no-reflect-apply` | Ruff `B009` and `B010` ([get-attr-with-constant](https://docs.astral.sh/ruff/rules/get-attr-with-constant/)) | They flag `getattr` with a constant name only, not dynamic `getattr` |
| `no-runtime-typeof` | None; parse at the boundary with a schema library instead | `isinstance` is idiomatic Python, so a ban would fight the language (judgment) |
| `no-accumulating-spread`, `no-reduce-accumulator-copy` | Ruff `PERF401` ([manual-list-comprehension](https://docs.astral.sh/ruff/rules/manual-list-comprehension/)) | Covers list appends only; no rule for `acc = {**acc, k: v}` in a loop |
| `no-module-mocking` (test seams) | Ruff `TID251` banning `unittest.mock.patch` and `mock.patch.object`; pytest `monkeypatch` documented as the fixture alternative ([pytest monkeypatch](https://docs.pytest.org/en/stable/how-to/monkeypatch.html)) | `monkeypatch.setattr` still patches modules and is a fixture, so `TID251` cannot ban it |
| Architecture boundaries (`arch:check`, dependency-cruiser) | import-linter contracts: forbidden, layers, independence, protected, acyclic siblings ([import-linter](https://import-linter.readthedocs.io/en/stable/)) | Comparable for import rules |
| knip (unused files, exports, dependencies) | vulture for dead code, deptry for unused and missing dependencies | vulture reports likely dead code with confidence levels and needs an allowlist (judgment, not checked in its docs) |
| Fallow semantic duplication | jscpd: 224 formats, `--threshold`, and `--baseline-from-ref` with `--fail-on-new-clones` ([jscpd](https://github.com/kucherenko/jscpd)); Pylint `duplicate-code` R0801 ([message](https://pylint.readthedocs.io/en/stable/user_guide/messages/refactor/duplicate-code.html)) | I did not compare detection quality |
| Prettier | `ruff format` | None |
| `npm audit` | pip-audit or OSV-Scanner | None |
| StrykerJS mutation testing | mutmut: incremental, retests only changed functions, needs `fork` so Windows needs WSL, and documents no failing score threshold ([mutmut docs](https://mutmut.readthedocs.io/en/latest/)); Cosmic Ray ([docs](https://cosmic-ray.readthedocs.io/en/latest/)) | mutmut gives no built-in CI threshold; a script must read its results |
| The register ratchet | basedpyright baseline file `./.basedpyright/baseline.json` (basedpyright config); `mypy-baseline` on PyPI | For Ruff the usual route is `--add-noqa`, which writes suppressions into the code (not checked in the Ruff docs here) |

## 4. Evidence that such gates work

The literature search was delegated to a subagent and its report was checked against the sources marked "read" in the table notes. "Read" means the full text or the authors' HTML was opened; "abstract" means only the abstract or listing page was seen; "secondary" means the number came from slides, a citing paper or a summary. Section "Not verified" lists every number that is not from a full text.

### 4.1 Static typing and type strictness

| Study | Design and sample | Result | Limits | Read |
| --- | --- | --- | --- | --- |
| Gao, Bird, Barr, [To Type or Not to Type](https://discovery-pp.ucl.ac.uk/id/eprint/10064729/1/typestudy.pdf), ICSE 2017 | 400 fixed public JavaScript bugs, annotated by hand and checked with Flow and TypeScript | Each checker detects about 15% (95% interval 11.5% to 18.5%) | Detectability, not prevention; bugs that already passed review | Abstract, secondary |
| Khan, Chen, Varró, McIntosh, [Type-Related Defects in Python Projects](https://rebels.cs.uwaterloo.ca/papers/tse2021_khan.pdf), IEEE TSE 2022 | Issue trackers and histories of 210 Python projects | mypy could have prevented 15% of corrective defects, 11% of all defects | Mining noise; lower bound | Abstract, secondary |
| Bogner, Merkel, [JavaScript and TypeScript on GitHub](https://arxiv.org/abs/2203.11115), MSR 2022 | 604 projects, more than 16 million lines | TypeScript projects scored better on quality metrics, but bug proneness and fix time did not differ significantly. `any` use correlated weakly with worse metrics (Spearman 0.17 to 0.26), not with bug proneness | Observational | Abstract |
| Ray et al. FSE 2014, reanalysed by [Berger et al. TOPLAS 2019](https://arxiv.org/abs/1901.10220) | 729 projects, 17 languages | After correction, only 4 languages kept a significant association with defects, and the effect was "exceedingly small" | Contested in a rebuttal and reply | Abstract |
| Di Grazia, Pradel, [Evolution of Type Annotations in Python](https://2022.esec-fse.org/details/fse-2022-research-papers/36/), ESEC/FSE 2022 | Python repository histories | 78.3% of commits went in despite type errors, because the projects did not enforce the checker | Usage study | Abstract |
| Lee, Hassan, Hindle, [Mining Type Constructs Using Patterns in AI-Generated Code](https://arxiv.org/abs/2602.17955), arXiv 2026 | Agent and human TypeScript pull requests | Agents are "9x more prone to use the 'any' keyword"; agent pull requests had 1.8 times the acceptance rate | Preprint; sample size not in the abstract | Abstract (checked myself) |

Judgment. Two retrospective studies agree that a type checker flags about 15% of fixed public bugs. No study I found measures whether stricter settings than basic checking (banning `any`, banning casts, `noUncheckedIndexedAccess`) reduce defects further, and none links TypeScript type assertions to defects. Di Grazia and Pradel suggest that a checker that does not block merges is mostly ignored. The only agent-specific number is that agents use `any` far more than humans. That is a reason to block escape hatches. It is not evidence that blocking them prevents defects.

### 4.2 Linters and custom lint rules

| Study | Result | Limits | Read |
| --- | --- | --- | --- |
| Google Tricorder, [Software Engineering at Google, chapter 20](https://abseil.io/resources/swe-book/html/ch20.html) (same system as Sadowski et al., CACM 2018) | A check must have fewer than 10% "effective false positives"; the overall rate is just below 5%. Google focuses on newly introduced warnings | Experience report; click and fix rates, no defect outcome | Read (book chapter) |
| Distefano et al., [Scaling Static Analyses at Facebook](https://discovery.ucl.ac.uk/10084236/1/O%27Hearn%20AAM%20scaling-static-analysis-at-facebook.pdf), CACM 2019 | The same Infer findings were fixed 70% of the time at diff time and 0% as a batch list | Experience report | Secondary |
| Habib, Pradel, How Many of All Bugs Do We Find?, ASE 2018 ([artifact](https://github.com/sola-da/StaticBugCheckers)) | Error Prone, Infer and SpotBugs together found 27 of 594 real Java bugs (4.5%) | Java bug detectors, not style linters | Secondary |
| Trautsch, Herbold, Grabowski, [Are automated static analysis tools worth it?](https://arxiv.org/abs/2111.09188), EMSE 2023 | Bug-inducing files did not have higher PMD warning density; effect sizes negligible | Java, Apache projects | Abstract |

Judgment. Generic lint warnings do not predict defects, and off-the-shelf bug detectors catch few real bugs. The strong industrial evidence is about delivery. Findings shown on the diff and blocking get fixed, batch reports get ignored, and developers keep only checks with few unwanted findings. No study I found measures custom lint rules, such as the anti-slop rules, against defects.

### 4.3 Mutation testing

| Study | Result | Limits | Read |
| --- | --- | --- | --- |
| Just et al., [Are Mutants a Valid Substitute for Real Faults?](https://homes.cs.washington.edu/~mernst/pubs/mutation-effectiveness-fse2014-abstract.html), FSE 2014 | 357 real faults in 5 Java programs: 73% coupled to mutants | Java, 5 programs | Abstract |
| Papadakis et al., [Are Mutation Scores Correlated with Real Fault Detection?](https://orbilu.uni.lu/handle/10993/34950), ICSE 2018 | Once suite size is controlled, the correlation is weak | C and Java benchmarks | Abstract |
| Petrović et al., [Practical Mutation Testing at Scale](https://arxiv.org/abs/2102.11378), IEEE TSE 2022 | 16.9 million mutants at Google, 2.1 million shown in review. Suppression heuristics raised the share of useful mutants from 15% to 89% (Python 71%). Median changelist: 820 candidate mutants without suppression, 7 with it | One company | Read |
| Petrović et al., [Does Mutation Testing Improve Testing Practices?](https://arxiv.org/abs/2103.07189), ICSE 2021 | 70% of 1,502 high-priority bugs had a mutant that would have been reported on the bug-introducing change; developers shown mutants changed a median of 11 test hunks against 0 | Not randomized | Read |
| Meta, [Mutation-guided LLM test generation (ACH)](https://arxiv.org/abs/2501.12862), FSE 2025 industry | Engineers accepted 73% of mutation-guided tests; 277 of 571 would have been discarded under line coverage alone | One company | Abstract |

Judgment. Mutation testing has the best fault-coupling evidence of the gates in this note. What was validated at Google is mutants shown on changed lines in review, heavily filtered, not a whole-repository score threshold. Keiko runs the second kind, nightly, and it has been red since at least 2026-09-23 without a mutation run (section 1.5).

### 4.4 Duplication

| Study | Result | Read |
| --- | --- | --- |
| Juergens et al., [Do Code Clones Matter?](https://arxiv.org/abs/1701.05472), ICSE 2009 | 107 faults found among inconsistently changed clone groups; about half of unintentionally inconsistent changes were faults | Abstract |
| Rahman, Bird, Devanbu, [Clones: What Is That Smell?](https://www.microsoft.com/en-us/research/wp-content/uploads/2016/02/rahman2010cws.pdf), MSR 2010 and EMSE 2012 | 4 projects: most bugs not significantly associated with clones | Abstract |
| GitClear, [AI Copilot Code Quality](https://www.gitclear.com/ai_assistant_code_quality_2025_research), 2025 vendor report | 211 million changed lines: copy-pasted lines rose from 8.3% to 12.3% (2021 to 2024), moved lines fell from 25% to under 10% | Read; method of clone detection and AI attribution not described |
| Huang et al., [More Code, Less Reuse](https://arxiv.org/abs/2601.21276), MSR 2026 | Agents miss reuse opportunities and add redundancy | Abstract, no numbers |

Judgment. Evidence that duplication causes defects is mixed. The defined risk is inconsistent edits to clones. Evidence that agents raise duplication is directional only. A gate that blocks new clones in changed files is cheap (1 minute in Keiko), but its benefit is not measured.

### 4.5 Ratchets and baselines

No peer-reviewed study measures whether a "no new violations" ratchet reduces defects. The support is engineering reports:

- Google focuses static analysis on newly introduced warnings, because fixing old warnings in working code can add bugs (SWE book chapter 20, read).
- Dropbox annotated about 4 million lines of Python over about 3 years, with annotations required in new files and weekly coverage emails ([Dropbox blog](https://dropbox.tech/application/our-journey-to-type-checking-4-million-lines-of-python), read).
- Stripe reached more than 95% of files at `typed: true` and about 85% of non-test files at `typed: strict` in 15 million lines over about 4 years, with a per-pull-request quality score ([Stripe blog](https://stripe.dev/blog/sorbet-stripes-type-checker-for-ruby), read).
- SonarSource recommends gating on new code only ([Clean as You Code](https://docs.sonarsource.com/sonarqube-server/10.6/user-guide/clean-as-you-code/)), with no outcome data.

Judgment. These reports show that a ratchet can move a large code base to strict typing over years. They give no defect numbers. Keiko's epic is a smaller and faster case of the same approach, and after 42 days its register is still above its baseline (section 2.5).

### 4.6 Code written by AI coding agents

| Study | Sample | Result | Gate that would catch it | Read |
| --- | --- | --- | --- | --- |
| Pearce et al., [Asleep at the Keyboard?](https://arxiv.org/abs/2108.09293), IEEE S&P 2022 | 1,689 Copilot programs for 89 CWE scenarios | About 40% vulnerable | CodeQL found them (with manual review) | Abstract |
| Perry et al., [Do Users Write More Insecure Code with AI Assistants?](https://arxiv.org/abs/2211.03622), CCS 2023 | Randomized, 47 participants | AI group less secure on most tasks; SQL injection in 36% of AI-group answers against 7% | Security analyzer, review | Read |
| Fu et al., [Security Weaknesses of Copilot-Generated Code in GitHub Projects](https://arxiv.org/abs/2310.02059), TOSEM | 733 snippets from real projects | 29.5% (Python) and 24.2% (JavaScript) had weaknesses | Static security analyzers | Abstract |
| Spracklen et al., [We Have a Package for You!](https://arxiv.org/abs/2406.10279), USENIX Security 2025 | 576,000 samples from 16 models | 19.7% of recommended packages did not exist; 43% of hallucinated names recurred in all 10 reruns | Lockfiles, dependency review | Read |
| Mündler et al., [Type-Constrained Code Generation](https://arxiv.org/abs/2504.09246), PLDI 2025 | LLM TypeScript on HumanEval and MBPP | About 94% of compilation errors were type errors | Type checker | Abstract; 94% secondary |
| Zhong, Raghunathan, Carlini, [ImpossibleBench](https://arxiv.org/abs/2510.20270), arXiv 2025 | SWE-bench and LiveCodeBench tasks whose tests contradict the specification | GPT-5 cheated on 76% of one-off and 54% of conflicting SWE-bench tasks; Claude Opus 4.1 on 50%. Claude models cheated mainly (more than 79%) by modifying tests. Read-only tests stopped test edits; hidden tests cut cheating to near zero; LLM monitors caught 42% to 65% on SWE-bench | Tests the agent cannot edit, review of test changes | Read (HTML, checked myself) |
| Cynthia, Muttakin, Roy, [Beyond Bug Fixes](https://arxiv.org/abs/2601.20109), arXiv 2026 | SonarQube diff analysis of 1,210 merged agent bug-fix pull requests in Python | Most new issues were code smells; bugs rarer but severe; merge status did not indicate quality | Static analysis on the diff | Abstract |
| Watanabe et al., [On the Use of Agentic Coding](https://arxiv.org/abs/2509.14745), arXiv 2025 | 567 Claude Code pull requests in 157 projects | 83.8% merged; 45.1% of merged ones needed human revisions, including to meet project standards | Project lint and format gates | Abstract |
| Blyth et al., [Static Analysis as a Feedback Loop](https://arxiv.org/abs/2508.14419), SCAM 2025 | GPT-4o, 10 rounds of Bandit and Pylint feedback | Security issues fell from more than 40% to 13% | Bandit, Pylint | Abstract |
| Kadam, [Beyond Passing Tests](https://arxiv.org/abs/2610.38977), arXiv 2026 | Review of 628 studies plus a re-evaluation | Analyzer feedback loops improved analyzer counts only; single quality signals were gamed | (warning against one-metric gates) | Abstract |
| METR, [RCT on experienced developers](https://arxiv.org/abs/2507.09089), 2025 | 16 developers, 246 tasks | 19% slower with AI (95% interval +2% to +39%); a 2026 update estimated -4% | Context only | Abstract |

Judgment. The failure modes of agent-written code have firm numbers: security weaknesses in 25% to 40% of generated code in these samples, 19.7% hallucinated packages, type errors as most compile failures of LLM TypeScript, heavy `any` use, and test tampering in 50% to 76% of impossible tasks. The gates these numbers point to are a blocking type check that bans escape hatches, a security analyzer, dependency review on a lockfile, tests that agents cannot quietly change, and mutation checks of agent-written tests. None of the studies measures a CI gate's effect on shipped defects. Most 2026 agent papers are unreviewed preprints, and the vendor reports have undisclosed methods.

## 5. Cost and fit for this repository

### 5.1 What is free for a public repository

Documented on the vendors' pages, read 2026-11-21 (delegated read; quotes as reported):

| Service | Free on a public repository | Source |
| --- | --- | --- |
| GitHub Actions, standard hosted runners | "Use of the standard GitHub-hosted runners is free and unlimited on public repositories." This includes macOS and Windows standard runners. "Larger runners are billed for public repositories." Free plan limits: 20 concurrent jobs, 5 concurrent macOS jobs, 6 hours per job | [Choose the runner](https://docs.github.com/en/actions/how-tos/write-workflows/choose-where-workflows-run/choose-the-runner-for-a-job), [Actions billing](https://docs.github.com/en/billing/concepts/product-billing/github-actions), [Actions limits](https://docs.github.com/en/actions/reference/limits) |
| CodeQL code scanning, secret scanning, push protection | Free on public repositories: "Public repositories: Secret scanning runs automatically for free." Code scanning on private repositories needs a GitHub Code Security licence | [About GitHub Advanced Security](https://docs.github.com/en/get-started/learning-about-github/about-github-advanced-security), [secret scanning](https://docs.github.com/en/code-security/secret-scanning/introduction/about-secret-scanning), [code scanning](https://docs.github.com/en/code-security/code-scanning/introduction-to-code-scanning/about-code-scanning) |
| Dependency review action | "The action is available for all public repositories". The Advanced Security feature table marks dependency review "No" for a public repository without Code Security, so the two pages disagree; Keiko, a public organisation repository, ran it green 58 of 60 times | [About dependency review](https://docs.github.com/en/code-security/supply-chain-security/understanding-your-software-supply-chain/about-dependency-review) |
| Dependabot alerts and the dependency graph | Included in all plans | About GitHub Advanced Security |
| SonarQube Cloud | Free plan: "Analysis of public projects: unlimited number of projects"; pull-request analysis "Only if the target branch is the main branch"; an OSS plan for public repositories | [Subscription plans](https://docs.sonarsource.com/sonarqube-cloud/administering-sonarcloud/managing-subscription/subscription-plans) |
| Codecov, Coveralls | Unlimited uploads for public repositories; "Coveralls will always be free for open source." | [Codecov pricing](https://about.codecov.io/pricing/), [Coveralls pricing](https://coveralls.io/pricing) |
| Gitleaks, OSV-Scanner, actionlint, zizmor | Free tools (MIT, Apache-2.0, MIT, MIT). The `gitleaks-action` from v2 needs a licence key for organisations; running the binary, as Keiko does, avoids that | [gitleaks-action README](https://github.com/gitleaks/gitleaks-action) |
| OpenSSF Scorecard action | "free for all public repositories" | [scorecard-action](https://github.com/ossf/scorecard-action) |
| Socket | "Socket is and will always be free to use for open-source." | [Socket pricing](https://socket.dev/pricing) |
| Stryker Dashboard, mutmut, Cosmic Ray | Dashboard "is free to use and open source"; the Python tools are local and free | [Stryker Dashboard](https://stryker-mutator.io/docs/General/dashboard/) |

So money is not the constraint for `Niko4417/OPT-cc` as long as it stays public and uses standard runners. The constraints are wall-clock time per pull request, the owner's time to read results, and the code that keeps the gates running.

### 5.2 How comparable small projects set up CI

Measured by a delegated read of each repository at a pinned commit; sizes are GitHub language byte counts, not lines. Times are medians of recent pull-request runs from the Actions API (`updated_at` minus `run_started_at`), so they are approximate. I spot-checked the workflow lists of chonkie, LightRAG and Open WebUI.

| Project | Size and maintainers | Lint and format | Type check | Tests on pull requests | Security scans | Action pinning | Median PR time |
| --- | --- | --- | --- | --- | --- | --- | --- |
| [chonkie](https://github.com/chonkie-inc/chonkie/tree/664454b689d68614bcfc0b1d57fc2a28b5862825/.github/workflows) (a dependency of this project) | Python 1.7 MB; effectively 2 maintainers | Ruff, `select = ["B904", "F", "I", "D"]`, `ruff format --check` | `ty check src/chonkie` blocks; tests checked with `continue-on-error` | pytest on Python 3.10 to 3.13, coverage to Codecov without a gate | TruffleHog (on `@main`), Dependabot | Tags, one SHA | 2.4 min |
| [txtai](https://github.com/neuml/txtai/tree/351a2bada485716d75acd7a6217c5bdcc84cb380/.github/workflows) | Python 1.7 MB; 1 maintainer | pylint and black via pre-commit, `duplicate-code` disabled | None | unittest on Ubuntu, macOS, Windows; `coverage report --fail-under=100` | None | Tags | 55 min |
| [docling-serve](https://github.com/docling-project/docling-serve/tree/69fb0588c5e04e09b4b6b6a8edf9613e812b3c36/.github/workflows) (glue service around docling) | Python 0.46 MB, TypeScript 0.13 MB | Ruff with a wide select, mccabe 15, Vale | `ty`, `tsc -b` | None; wheel install smoke only | actionlint | Mixed | 16.5 min (container builds) |
| [obsidian-local-llm-hub](https://github.com/takeshy/obsidian-local-llm-hub/tree/a266afe79231e3f7c18ad2b48aac02dd0c160753/.github/workflows) (TypeScript RAG plugin) | TypeScript 0.73 MB; 1 maintainer | ESLint `recommendedTypeChecked` with `no-unsafe-*` as errors, `--max-warnings 0` | `tsc -noEmit`, `strict: true` | None; tests exist but are not run | None | Tags | 0.6 min |
| [Open WebUI](https://github.com/open-webui/open-webui/tree/0666e59d26a4d056e0bacee41742486badd71cf8/.github/workflows) (stack candidate) | Svelte, Python, TypeScript, 12 MB | `ruff check --select=F`, Prettier | None | Frontend vitest only | Dependabot | Tags plus `actions.lock` | 1.8 min |
| [LightRAG](https://github.com/HKUDS/LightRAG/tree/f2e2d3f06fc1b3b50a521420a01059f410235a71/.github/workflows) (stack candidate) | Python 18.8 MB | Ruff via pre-commit, `select = ["E4","E7","E9","F"]` | `tsc --noEmit` for the web UI only | pytest in 4 shards, web UI tests | Dependabot | Major tags | 2.1 min |
| Keiko `dev` (stack candidate) | TypeScript 122 MB | ESLint strict type-checked, Prettier, knip, Fallow | `tsc` strict | Everything, plus end-to-end | CodeQL, dependency review, OSV, Gitleaks, zizmor, actionlint, Socket, SonarQube Cloud | SHA, checked by CI | 53 to 58 min |

Documented in those files. A typical small project runs Ruff, a test matrix and an optional coverage upload in 2 to 3 minutes. Two of the four small projects gate on a type checker (`ty`); none of the small Python projects uses mypy or Pyright. Among the small projects, only chonkie runs a security scan, and none runs CodeQL as a workflow, dependency review, OSV-Scanner, zizmor, mutation testing or duplication checks. Two of them have test suites that CI never runs. Only Keiko pins every action to a SHA.

Judgment. The small projects show what is normal, not what works. They give no evidence that their light CI is enough, and the agent studies in section 4.6 point at gates (dependency review, security analysis, test protection) that none of them runs.

## 6. Recommendation

Everything in this section is judgment built on the facts above. Each row names the evidence it rests on and its cost. "Evidence" means a study or report from section 4; "Keiko" means a measured Keiko fact from sections 1 and 2.

### 6.1 Do not replicate Keiko's whole CI

Of Keiko's 22 workflows, 12 test or ship Keiko's own product (section 1.3), and the `ui`, cross-platform, Firefox and portable jobs make up most of a 53-minute, 392-job-minute pull-request run. Three scheduled lanes have been red for 2 to 4 months without blocking anything (section 1.5). The governance scripts alone are 83,495 lines. Comparable small projects run in 2 to 3 minutes (section 5.2). Copying the jobs would copy Keiko's product checks, and copying the scripts would add more CI code than the glue code it guards. The earlier note's claim that most Keiko jobs "are more than a glue repo needs" holds for the product, release and governance jobs. It does not hold for the language-independent security and hygiene jobs, which are cheap and should be copied.

### 6.2 Case C: the stack choice is still open (now)

These gates do not depend on the stack, so they can go in before any glue code exists.

| When | Gate | Evidence | Cost |
| --- | --- | --- | --- |
| Now | Gitleaks binary over the pull-request range, SHA-256 pinned, as in Keiko's `secret-scan` job | Keiko; GitHub secret scanning is also free | 0.6 min |
| Now | Workflow hygiene: actionlint, zizmor, and the grep that every `uses:` is pinned to a 40-hex SHA | Keiko; none of the small projects pins by SHA, and chonkie runs a scanner from `@main` | 0.7 min |
| Now | CodeQL for `actions` | Free; Keiko | Minutes, in parallel |
| Now | One required `ci` aggregate that fails closed on a failed or skipped need | Keiko pattern; keeps branch protection to one context | Seconds |
| First manifest | Dependency review with a severity and licence policy, and OSV-Scanner in the hygiene job | Spracklen et al. (19.7% hallucinated packages); Keiko | 0.6 min |
| First glue code | The language set of case A or B, at zero findings from the first commit | Section 6.5 | See below |
| Leave out | Verified-tree reuse, change-scope classifier, merge-candidate check, infrastructure re-run, secret-scanning queue, release workflows | They solve problems of a 53-minute matrix and a release pipeline; a 3-minute run can be re-run by hand | Avoids several thousand lines of Node.js scripts |

### 6.3 Case A: Keiko wins and the glue code is TypeScript in or beside Keiko

If the glue code lives inside a fork of Keiko, the fork carries Keiko's whole CI, and that CI has to be cut down to the rows below. If it lives in this repository beside a pinned Keiko version, it starts from the rows below.

| When | Gate | Evidence | Cost |
| --- | --- | --- | --- |
| First commit | `tsc` with Keiko's flags: `strict`, `noUncheckedIndexedAccess`, `exactOptionalPropertyTypes`, `noImplicitOverride` | Gao et al. (about 15% of bugs detectable); Mündler et al. (most compile errors of LLM TypeScript are type errors) | Seconds at this size (judgment) |
| First commit | typescript-eslint `strictTypeChecked` with `no-explicit-any`, zero warnings | Lee et al. (agents use `any` 9 times as often); Google and Facebook (blocking findings on the diff get fixed) | Under a minute at this size (judgment); Keiko's 16 minutes are for a 122 MB repository |
| First commit | Oxlint with `oxlint-plugin-anti-slop` pinned to a commit: `no-chained-type-assertions`, `require-safety-comment-for-type-assertion`, `no-widen-then-assert`, `no-reflect-get`, `no-reflect-apply`, `no-unknown-returns`, `no-object-parameters`, and `no-module-mocking` in tests too. No register, no ledger | No defect study; Keiko found real defects behind `as unknown as` casts (section 2.5) | Under a minute; Keiko's ratchet step took 0.7 min for 7,200 files |
| First commit | The 4 boundary rules (`no-unknown-parameters`, `no-runtime-typeof`, `no-unsafe-dictionary-type`, `no-known-value-widening`), on everywhere except one boundary module that parses model output, HTTP responses and Keiko API data with a schema | No defect study; in Keiko these 4 rules hold 22,931 findings, about half the register, so they are the rules ordinary code hits most | Design work for the boundary module; ongoing friction not measured |
| First commit | Prettier; Vitest; coverage reported, not gated | No study supports a coverage threshold as a defect gate | Seconds |
| First commit | Semantic duplication with Fallow `--gate new-only --base <sha>` | Weak (section 4.4) | 1 min |
| First commit | A job that lists every changed existing test file in the run summary, so the owner reviews test edits made by an agent | ImpossibleBench: Claude models cheated mainly by editing tests; read-only tests stopped it | A short script |
| Later, behind a ratchet | Stryker in incremental mode on the citation and model-policy modules, reported on changed code, not as a nightly lane | Petrović et al. (70% of bugs coupled to a mutant on the bug-introducing change); Keiko's nightly lane red 60 of 60 | Minutes per run; not measured here |
| Later | knip, once the code has several packages | None found | Seconds |
| Optional | Socket and SonarQube Cloud | Free; Cynthia et al. used SonarQube to find smells in agent pull requests | A token and an app install; Sonar's scan took 11.9 min in Keiko |
| Leave out | Keiko's product, release and governance jobs; the responsibility qualification engine (11,850 lines of code, 26,500 lines of tests, about 84 runner-minutes per pull request); the increase ledger; the 13 Keiko hygiene checks | They exist for Keiko's product and for 46,618 legacy findings | Avoids the costs of section 2.4 |

### 6.4 Case B: a Python stack wins

| When | Gate | Evidence | Cost |
| --- | --- | --- | --- |
| First commit | One type checker in strict mode on the glue code only: mypy with `strict`, `disallow_any_explicit` and `warn_return_any`, or basedpyright with `reportAny` and `reportExplicitAny` | Khan et al. (mypy could have prevented 15% of corrective defects); Di Grazia and Pradel (a checker that does not block is ignored) | Seconds to a minute (judgment) |
| First commit | Ruff with a wide `select` that includes at least `F`, `E`, `B`, `S`, `PT`, `PGH`, `ANN`, `TID`, `PERF` and `RUF`; `TID251` banning `unittest.mock.patch` and `typing.cast` outside the boundary module; `ruff format --check` | Pearce, Perry and Fu et al. (security weaknesses; `S` is Ruff's Bandit set); Blyth et al. (Bandit and Pylint feedback reduced issues) | Seconds |
| First commit | import-linter: a `layers` or `forbidden` contract so only one adapter package imports the adopted app | No defect study; it is the Python form of Keiko's boundary rules | Seconds |
| First commit | pytest with coverage reported, not gated; deptry for unused and missing dependencies | No study supports a coverage threshold | Seconds |
| First commit | The changed-test listing job of case A | ImpossibleBench | A short script |
| Later, behind a ratchet | mutmut on the citation and model-policy modules, on Linux runners | Petrović et al. | mutmut needs `fork`, so it runs on Linux only, and needs a script for a threshold |
| Later | jscpd with `--baseline-from-ref` and `--fail-on-new-clones` | Weak (section 4.4) | Not measured |
| Leave out | Everything in the case A "leave out" row, and the TypeScript-only rules with no Python counterpart | Section 3.2 | |

Python is weaker than TypeScript here in three places (section 3.3): no checker flags `object`-typed parameters or `dict[str, object]`, no rule requires a justification beside a `cast`, and the adopted Python apps are not fully typed, so `Any` enters at every call into them. The adapter package and import-linter contract are the main defence against the last point.

### 6.5 Does adopting the anti-slop rules from the first commit change the cost?

Yes, most of Keiko's cost came from legacy code (documented and measured in section 2):

- Keiko started from 46,618 findings in 7,239 files. The register, the increase ledger with 1,139 entries, the one-time `dev-cutoff-sync` and the 42 days of 51 pull requests all exist to move that legacy code without blocking other work. After 42 days the register is still 381 findings above its baseline, and nothing is on `dev`.
- The 84-runner-minute qualification job checks 133 responsibility records that let legacy validators keep findings. A code base with zero findings needs no such records.
- The ratchet scan itself is cheap: 0.7 minutes in CI for 7,200 files.

Judgment on what remains when the glue code starts at zero findings:

- The engine cost drops to a lint configuration and a sub-minute step. No register, no ledger and no qualification are needed, because there is nothing to tolerate.
- The cost moves to writing compliant code. Every value that arrives as `unknown` (model output, HTTP bodies, the adopted app's API) has to pass a schema parse in one boundary module. That design is the work Keiko's 4 inactive boundary rules still wait for. Nobody has measured the friction for a small code base.
- The plugin is young (0.1.2, installed from a GitHub tarball) and syntactic only. It has at least one known false-positive pattern (45 of 4,210 widening findings in Keiko). Pin it to a commit and expect to report or patch rules upstream.
- The rules have no published defect evidence. The case for them rests on agent behaviour (heavy `any` use, test tampering) and on Keiko's anecdotal defect finds, so they should be cheap to drop if they block more than they find.

## Not verified

- Keiko's full branch protection settings (required reviews, linear history, signed commits) need admin access; only the required checks were readable. Whether CodeQL default setup also runs beside `codeql.yml` could not be read (HTTP 403); the API lists a "dynamic" CodeQL workflow with runs on 2026-11-19 to 2026-11-21.
- The CI cost of the epic compares two epic pull requests with one `dev` pull request on different bases. It is not a controlled comparison.
- The epic facts in sections 2.2 to 2.5 come partly from a delegated read. I checked the branch state, ADR-0167 D6, the register total (46,999 in 4,468 files) and the per-rule counts, and the policy job times. I did not check the per-package counts, the pull-request counts and line totals, or each slice's defect claims.
- The epic's issue bodies (14 issues of 20 to 47 KB each) were read only in part.
- Literature numbers not read in full text: Gao et al. 15%, Khan et al. 15% and 11%, Habib and Pradel 4.5%, Facebook's 70% and 0%, Juergens et al. "about half", Mündler et al. 94%, and all entries marked "abstract" in section 4. The venue of ImpossibleBench is not confirmed; I read it as an arXiv preprint.
- Vendor pages in section 5.1 were read by a subagent; I did not reopen them. The two GitHub pages disagree on whether dependency review is free on public repositories.
- The run times of comparable projects approximate wall clock from `updated_at`, which re-runs can inflate. Maintainer counts come from commit counts only.
- The Python tool limits marked "not checked" in section 3.3 (`uv.lock` support in dependency review, vulture's allowlist, Ruff `--add-noqa`) are from memory, not from their docs.
- The run time of each proposed gate on a repository of this size is estimated, not measured. The first CI runs should record it.

## Sources

All read on 2026-11-21.

1. Keiko `dev` at `4424cea`: [`.github/workflows/`](https://github.com/oscharko-dev/Keiko/tree/4424cea62707f830296b9685866a558419a4e967/.github/workflows) (22 files), [`package.json`](https://github.com/oscharko-dev/Keiko/blob/4424cea62707f830296b9685866a558419a4e967/package.json), [`eslint.config.js`](https://github.com/oscharko-dev/Keiko/blob/4424cea62707f830296b9685866a558419a4e967/eslint.config.js), [`tsconfig.base.json`](https://github.com/oscharko-dev/Keiko/blob/4424cea62707f830296b9685866a558419a4e967/tsconfig.base.json), [`stryker.security.conf.json`](https://github.com/oscharko-dev/Keiko/blob/4424cea62707f830296b9685866a558419a4e967/stryker.security.conf.json), [`docs/adr/`](https://github.com/oscharko-dev/Keiko/tree/4424cea62707f830296b9685866a558419a4e967/docs/adr) (ADR-0156, ADR-0161, ADR-0167, ADR-0178).
2. Keiko branch protection and rulesets: [branches API for `dev`](https://api.github.com/repos/oscharko-dev/Keiko/branches/dev), [rulesets API](https://api.github.com/repos/oscharko-dev/Keiko/rulesets).
3. Keiko Actions runs API: last 60 runs per workflow; jobs of runs [44897203715](https://github.com/oscharko-dev/Keiko/actions/runs/44897203715), [44880084548](https://github.com/oscharko-dev/Keiko/actions/runs/44880084548), [44881340006](https://github.com/oscharko-dev/Keiko/actions/runs/44881340006); log of [44891765307](https://github.com/oscharko-dev/Keiko/actions/runs/44891765307).
4. Keiko epic branch at `45fbce4`: [ADR-0167](https://github.com/oscharko-dev/Keiko/blob/45fbce4345e68ddc873b09177a2a7220ba0149d2/docs/adr/ADR-0167-zero-cost-autonomous-quality-gates.md), [`docs/qa/code-quality-policy.md`](https://github.com/oscharko-dev/Keiko/blob/45fbce4345e68ddc873b09177a2a7220ba0149d2/docs/qa/code-quality-policy.md), [`scripts/code-quality-policy.json`](https://github.com/oscharko-dev/Keiko/blob/45fbce4345e68ddc873b09177a2a7220ba0149d2/scripts/code-quality-policy.json), [`scripts/code-quality-register.json`](https://github.com/oscharko-dev/Keiko/blob/45fbce4345e68ddc873b09177a2a7220ba0149d2/scripts/code-quality-register.json), [`scripts/code-quality-register-increases.json`](https://github.com/oscharko-dev/Keiko/blob/45fbce4345e68ddc873b09177a2a7220ba0149d2/scripts/code-quality-register-increases.json), [`scripts/check-semantic-duplication.mjs`](https://github.com/oscharko-dev/Keiko/blob/45fbce4345e68ddc873b09177a2a7220ba0149d2/scripts/check-semantic-duplication.mjs).
5. Keiko issues: ["Epic: Enforce precise type safety and test fidelity across Keiko"](https://github.com/oscharko-dev/Keiko/issues/3914) and its sub-issues, ["Deliver anti-slop adoption to current dev through scoped PRs and drift control"](https://github.com/oscharko-dev/Keiko/issues/4145), ["Activate full enforcement and prove integrated production quality"](https://github.com/oscharko-dev/Keiko/issues/3991), ["Activate the four contracts boundary rules after epic #3914 integrates"](https://github.com/oscharko-dev/Keiko/issues/4278), ["Cut the type-aware ESLint memory back under the runner with a quarter of headroom"](https://github.com/oscharko-dev/Keiko/issues/4202), ["Security mutation suite failed on dev"](https://github.com/oscharko-dev/Keiko/issues/3663).
6. Keiko pull requests: ["land the anti-slop engine on the epic branch as a shrink-only ratchet"](https://github.com/oscharko-dev/Keiko/pull/4165), ["compare semantic duplication with the base commit"](https://github.com/oscharko-dev/Keiko/pull/4189), ["enforce the anti-slop policy in full on the epic branch"](https://github.com/oscharko-dev/Keiko/pull/4286), ["qualify noncosmetic code-quality policy"](https://github.com/oscharko-dev/Keiko/pull/4008), [4328](https://github.com/oscharko-dev/Keiko/pull/4328), [4368](https://github.com/oscharko-dev/Keiko/pull/4368), [4509](https://github.com/oscharko-dev/Keiko/pull/4509), [4510](https://github.com/oscharko-dev/Keiko/pull/4510), [4481](https://github.com/oscharko-dev/Keiko/pull/4481).
7. [anti-slop plugin README at `c44ef22`](https://github.com/dmmulroy/anti-slop/blob/c44ef22ca116d0ba62a3ff663a0bd13a3f3fa40b/README.md); [Fallow README](https://github.com/fallow-rs/fallow).
8. Python tool docs: [mypy configuration](https://mypy.readthedocs.io/en/stable/config_file.html), [basedpyright configuration](https://docs.basedpyright.com/latest/configuration/config-files/), [Ruff rules](https://docs.astral.sh/ruff/rules/) with [PGH003](https://docs.astral.sh/ruff/rules/blanket-type-ignore/) and [PERF401](https://docs.astral.sh/ruff/rules/manual-list-comprehension/), [import-linter](https://import-linter.readthedocs.io/en/stable/), [mutmut](https://mutmut.readthedocs.io/en/latest/), [Cosmic Ray](https://cosmic-ray.readthedocs.io/en/latest/), [jscpd](https://github.com/kucherenko/jscpd), [Pylint duplicate-code](https://pylint.readthedocs.io/en/stable/user_guide/messages/refactor/duplicate-code.html), [pytest monkeypatch](https://docs.pytest.org/en/stable/how-to/monkeypatch.html).
9. Studies on typing: Gao, Bird, Barr, ICSE 2017; Khan et al., IEEE TSE 2022; Bogner and Merkel, MSR 2022; Berger et al., TOPLAS 2019; Di Grazia and Pradel, ESEC/FSE 2022; Lee, Hassan, Hindle, arXiv 2602.17955. Links in section 4.1.
10. Studies on static analysis: Google SWE book chapter 20; Distefano et al., CACM 2019; Habib and Pradel, ASE 2018; Trautsch et al., EMSE 2023. Links in section 4.2.
11. Studies on mutation testing: Just et al., FSE 2014; Papadakis et al., ICSE 2018; Petrović et al., IEEE TSE 2022 and ICSE 2021; Meta ACH, FSE 2025. Links in section 4.3.
12. Studies on duplication: Juergens et al., ICSE 2009; Rahman, Bird, Devanbu, MSR 2010; GitClear 2025; Huang et al., MSR 2026. Links in section 4.4.
13. Ratchet reports: Dropbox mypy blog; Stripe Sorbet blog; SonarSource Clean as You Code. Links in section 4.5.
14. Studies on agent code: Pearce et al.; Perry et al.; Fu et al.; Spracklen et al.; Mündler et al.; Zhong, Raghunathan, Carlini (ImpossibleBench); Cynthia, Muttakin, Roy; Watanabe et al.; Blyth et al.; Kadam; METR. Links in section 4.6.
15. Vendor pages: GitHub Actions billing, runners and limits; GitHub Advanced Security, secret scanning, code scanning and dependency review; SonarQube Cloud plans; Codecov and Coveralls pricing; gitleaks-action, scorecard-action, Socket pricing, Stryker Dashboard. Links in section 5.1.
16. Comparable projects at pinned commits: [chonkie](https://github.com/chonkie-inc/chonkie/tree/664454b689d68614bcfc0b1d57fc2a28b5862825), [txtai](https://github.com/neuml/txtai/tree/351a2bada485716d75acd7a6217c5bdcc84cb380), [docling-serve](https://github.com/docling-project/docling-serve/tree/69fb0588c5e04e09b4b6b6a8edf9613e812b3c36), [obsidian-local-llm-hub](https://github.com/takeshy/obsidian-local-llm-hub/tree/a266afe79231e3f7c18ad2b48aac02dd0c160753), [Open WebUI](https://github.com/open-webui/open-webui/tree/0666e59d26a4d056e0bacee41742486badd71cf8), [LightRAG](https://github.com/HKUDS/LightRAG/tree/f2e2d3f06fc1b3b50a521420a01059f410235a71), and their Actions runs API.
17. Earlier note: [`docs/research/quality-gates.md`](https://github.com/Niko4417/OPT-cc/blob/dd07ded4215cb860af3c1a1b405b95079db3c0b7/docs/research/quality-gates.md) on branch `research/quality-gates`.
