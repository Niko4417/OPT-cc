# Keiko's CI and the anti-slop epic: what the project memory should copy

Research for the owner's question on replicating Keiko's CI. Sources were read on 2026-11-21. It extends [`docs/research/quality-gates.md`](https://github.com/Niko4417/OPT-cc/blob/dd07ded4215cb860af3c1a1b405b95079db3c0b7/docs/research/quality-gates.md) (branch `research/quality-gates`), which covered only Keiko's 8 grounding and retrieval gates and its model-backed labs. That note said, without evidence, that most other Keiko CI jobs "serve a large product repo and are more than a glue repo needs". This note checks that claim job by job.

Keiko links point at commits of [oscharko-dev/Keiko](https://github.com/oscharko-dev/Keiko) (Apache-2.0): `dev` at [`4424cea`](https://github.com/oscharko-dev/Keiko/commit/4424cea62707f830296b9685866a558419a4e967) (2026-11-20) and the epic branch `codex/epic-anti-slop-quality` at [`45fbce4`](https://github.com/oscharko-dev/Keiko/commit/45fbce4345e68ddc873b09177a2a7220ba0149d2) (2026-11-21). Run times come from the GitHub Actions API on 2026-11-21. Three kinds of statement appear below and are kept apart: "documented" (a file, issue or paper says so), "measured" (I read the number from runs or computed it from files), and "judgment" (my reading, marked as such).

## Question

Should the project memory replicate Keiko's whole CI as it is on GitHub today? Should it replicate the stricter quality gates of Keiko's anti-slop epic, given that those are TypeScript-specific? Which gates fit a public repository with one owner, glue code of a few thousand to tens of thousands of lines, a stack that is not chosen yet, and code written mostly by coding agents?

## Short answer

To be completed after all parts.

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
- Size: the epic's net diff from its merge base is 1,226 files, +176,170 and −61,344 lines. About 37,400 lines are policy, register and ledger JSON, about 26,500 lines are engine tests and about 11,850 lines are engine code in 12 `scripts/lib/code-quality-*.mjs` files.
- Owner effort is visible: review rounds past 100 ("round-101" in the [closing comment of the full-enforcement issue](https://github.com/oscharko-dev/Keiko/issues/3991#issuecomment-6737092073)), and one-time exceptions such as merges without hosted CI on 2026-11-06.

Measured from run job times (I read these runs myself):

| Run | Static code-quality policy job, Node 24 and Node 26 | Anti-slop ratchet step | Semantic duplication job | Job-minutes | Wall clock |
| --- | --- | --- | --- | --- | --- |
| Epic PR, [44880084548](https://github.com/oscharko-dev/Keiko/actions/runs/44880084548) | 45.2 and 39.6 min on `ubuntu-24.04-arm` | 0.7 min | 1.1 min | 525 | 65.5 min |
| Epic PR, [44881340006](https://github.com/oscharko-dev/Keiko/actions/runs/44881340006) | 45.8 and 38.1 min | 0.8 min | 1.0 min | 506 | 76.2 min |
| `dev` PR without the epic, [44897203715](https://github.com/oscharko-dev/Keiko/actions/runs/44897203715) | none | none | 1.0 min | 392 | 53.1 min |

So the ratchet itself costs under a minute per pull request. The full qualification job costs about 84 runner-minutes per pull request (two legs of 38 to 46 minutes). The epic also moved package coverage from 4 to 5 shards and raised the script shard limit from 30 to 45 minutes, because the engine "adds about 1,450 scripts tests" ([PR "land the anti-slop engine on the epic branch as a shrink-only ratchet"](https://github.com/oscharko-dev/Keiko/pull/4165)). The comparison above is between different pull requests on different bases, so the difference of about 120 job-minutes is an estimate, not a controlled measurement.

A separate cost: type-aware ESLint peaks at about 12 GiB of memory, and "each contracts slice of #3914 adds to it" ([issue "Cut the type-aware ESLint memory back under the runner with a quarter of headroom"](https://github.com/oscharko-dev/Keiko/issues/4202)). A standard hosted Linux runner has 16 GB.

### 2.5 Results the epic reports

Documented:

| Point | Findings | Source |
| --- | --- | --- |
| First census | 41,084 raw matches in 6,183 files | [PR "qualify noncosmetic code-quality policy"](https://github.com/oscharko-dev/Keiko/pull/4008) |
| Baseline at `dev` snapshot `7bbca2e34` | 46,618 findings in 7,239 files | ADR-0167 D6 |
| After the `dev` 1.4.0 cutoff sync | 48,205, from 592 ledger raises in 246 files | ADR-0167 D6 |
| Epic head `45fbce4` | 46,999 in 4,468 files (measured from the register), with 1,139 ledger entries | Register and ledger at the epic head |

After 42 days the register is 381 findings above its starting baseline, because the ledger admitted 3,215 raised findings while slices removed others. Measured from the register: the peak was 48,553, so the net reduction from the peak is about 1,550 findings or 3.2%.

Defects found: several slice pull requests list behaviour changes, each with a test that fails on the base and passes after the change. Examples: SQLite row readers in memory-vault and local-knowledge now refuse tampered or wrongly typed columns that an `as unknown as` cast let through ([PR 4509](https://github.com/oscharko-dev/Keiko/pull/4509), [PR "clear and activate four more rule scopes in slice 5"](https://github.com/oscharko-dev/Keiko/pull/4510)); a legacy-port call with non-object arguments used to settle as "completed" and now fails ([PR "type the harness assertions through their real invariants"](https://github.com/oscharko-dev/Keiko/pull/4368)); reads now ignore a polluted `Object.prototype` ([PR "remove 25 known-value widenings in contracts"](https://github.com/oscharko-dev/Keiko/pull/4328)). The epic reports no total defect count and no false-positive rate. PR 4328 reports one upstream false positive in `no-known-value-widening` that accounts for "45 of 4,210 widening findings". The ADR says: "A raw match is a diagnostic, not a proven defect."

Engine run time: the ratchet takes "about 30 seconds on an Apple-silicon laptop" for about 7,200 files ([policy document](https://github.com/oscharko-dev/Keiko/blob/45fbce4345e68ddc873b09177a2a7220ba0149d2/docs/qa/code-quality-policy.md)). The full qualification took 35:39 and 43:52 minutes on hosted runners (PR 4286).

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
