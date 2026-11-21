# Where, when and how the model-backed evaluation runs on AWS

Date: 2026-11-21. Review: [Review and adopt: quality gates and CI](https://github.com/Niko4417/OPT-cc/issues/24). Research it revises: [Pick quality gates, CI and evaluation checks to adopt](https://github.com/Niko4417/OPT-cc/issues/14), note [`quality-gates.md`](https://github.com/Niko4417/OPT-cc/blob/research/quality-gates/docs/research/quality-gates.md) on branch `research/quality-gates` (written 2026-11-17, called "the old note" below). Context: [ADR-0013](https://github.com/Niko4417/OPT-cc/blob/main/docs/adr/0013-run-v1-on-aws-as-a-proof-of-concept-and-leave-on-device-and-client-hosted-for-later.md), [ADR-0014](https://github.com/Niko4417/OPT-cc/blob/main/docs/adr/0014-require-eu-processing-for-every-third-party-cloud-route.md), [ADR-0016](https://github.com/Niko4417/OPT-cc/blob/main/docs/adr/0016-pick-the-cheapest-model-per-task-whose-quality-is-good-enough.md), [ADR-0017](https://github.com/Niko4417/OPT-cc/blob/main/docs/adr/0017-feed-the-managed-retrieval-arm-our-own-chunks-and-score-it-per-source-type.md), [ADR-0028](https://github.com/Niko4417/OPT-cc/blob/main/docs/adr/0028-use-thunderbird-as-the-small-test-project-and-shopware-as-the-large-one-measured-in-three-runs.md), [ADR-0030](https://github.com/Niko4417/OPT-cc/blob/main/docs/adr/0030-keep-the-test-dataset-in-a-public-folder-of-this-repository-and-ingest-it-as-a-folder.md), [ADR-0032](https://github.com/Niko4417/OPT-cc/blob/main/docs/adr/0032-freeze-issues-and-pull-requests-in-a-private-snapshot-and-keep-sync-off-for-scored-project-memories.md), the [AWS installation research](https://github.com/Niko4417/OPT-cc/blob/research/aws-installation/docs/research/aws-installation.md) (P1) and the [AWS models research](https://github.com/Niko4417/OPT-cc/blob/research/aws-models/docs/research/aws-models.md) (P2).

Labels used in this note: **documented** is a fact stated by the owner of the system (GitHub, AWS, Anthropic, a project's own files). **Measured** is a number someone measured and published. **Estimate** is my arithmetic from named inputs. **Judgment** is my own reasoning. Where a paragraph mixes them, the label sits next to the claim.

## Question

The old note ran the model-backed checks nightly on a 64 GB Mac with two local models. ADR-0013 moved V1 to AWS in Frankfurt (eu-central-1), with per-token Bedrock models behind LiteLLM (P2) and one EC2 instance per installation that is normally stopped (P1). Where, when and how should the model-backed evaluation run now, and which of the old note's threshold numbers have evidence behind them?

## Short answer

To be filled in when all parts are done.

## 1. GitHub Actions on a public repository that needs AWS credentials

### 1.1 Which events get secrets and an OIDC token

`Niko4417/OPT-cc` is public, was created on 2026-11-17 and has `main` as default branch [15]. Two GitHub rules decide what a workflow run can reach. Secrets other than `GITHUB_TOKEN` "are not passed to the runner when a workflow is triggered from a forked repository" [1, 2]. And an OIDC token needs the permission `id-token: write` [3, 5], while for a pull request event from a fork "any write permissions" are changed "to read only" unless an admin turns on the setting that sends write tokens to fork pull requests [3]. From these two rules I conclude that a fork pull request gets neither secrets nor an OIDC token (judgment from two documented rules; GitHub does not name `id-token` in the fork rule itself).

| Event | Code that runs | Repository secrets | OIDC token | `sub` claim (default format) | Source |
| --- | --- | --- | --- | --- | --- |
| `pull_request` from a fork | The pull request's merge commit | No | No (write permissions become read) | Not issued | [1, 2, 3] |
| `pull_request` from a branch of this repository | The pull request's merge commit, so the branch's version of the workflow file | Yes | Yes | `repo:OWNER/REPO:pull_request`, but only "if the job doesn't reference an environment" | [1, 4] |
| `pull_request_target` | The default branch of the base repository | Yes, also for forks | Yes | As for the default branch | [1, 3] |
| `push` to `main` | The pushed commit | Yes | Yes | `repo:OWNER/REPO:ref:refs/heads/main` | [1, 4] |
| `schedule` | "Last commit on default branch" | Yes | Yes | `repo:OWNER/REPO:ref:refs/heads/main` | [1, 4] |
| `workflow_dispatch` | The chosen branch or tag. After one run, a dispatch "against any branch or tag" is possible through the API or CLI | Yes | Yes | `repo:OWNER/REPO:ref:refs/heads/BRANCH` | [1, 4] |
| Any job that references an environment | Depends on the event | Plus the environment's secrets, after its protection rules pass | Yes | `repo:OWNER/REPO:environment:NAME` | [4, 6] |

Three more documented rules matter for this repository:

- `pull_request_target` runs with secrets and with a read/write `GITHUB_TOKEN` "even when it is triggered from a public fork" [3]. GitHub says to avoid it "if you need to build or run code from the pull request" [1], and its secure-use reference says such workflows "must not explicitly check out untrusted code" [7]. GitHub Security Lab calls the combination of `pull_request_target` and a checkout of the pull request "a dangerous practice" [14].
- A first-time contributor's fork pull request "may require manual approval from a maintainer with write access" before workflows run [8]. This gates when the workflow runs. It does not give the run any secret.
- Self-hosted runners "should almost never be used for public repositories" [7]. The old note already ruled them out. This still holds for the EC2 instance.

### 1.2 What the IAM role trust policy should condition on

**Documented.** IAM checks every role that trusts GitHub's OIDC provider. It rejects a trust policy unless the key `token.actions.githubusercontent.com:sub` is present and "its value is not solely a wildcard character" [13]. AWS also warns that without a `sub` limited to an organization or repository, "GitHub Actions from organizations or repositories outside of your control are able to assume roles" [13]. The audience for the official `aws-actions/configure-aws-credentials` action is `sts.amazonaws.com` [5].

**Documented.** Repositories created after 2026-07-15 use an immutable `sub` format that includes the owner ID and repository ID: `repo:OWNER@OWNER-ID/REPO@REPO-ID:...` [4, 5]. `Niko4417/OPT-cc` was created on 2026-11-17, with repository ID 1461301307 and owner ID 159039192 [15]. Its `sub` therefore looks like `repo:Niko4417@159039192/OPT-cc@1461301307:environment:NAME` (judgment from [4] and [15]; I did not request a token to confirm it). This format protects against a recycled repository name.

**Judgment.** The role that lets GitHub start an evaluation run should trust one environment and nothing else:

```json
"Condition": {
  "StringEquals": {
    "token.actions.githubusercontent.com:aud": "sts.amazonaws.com",
    "token.actions.githubusercontent.com:sub": "repo:Niko4417@159039192/OPT-cc@1461301307:environment:evaluation"
  }
}
```

The reason is that every other form has a hole in this repository:

- `...:pull_request` is issued to every same-repository pull request, and that pull request's branch decides what the workflow file does [1, 4]. Any branch pushed by the owner or by a coding agent with push access could then assume the role.
- `...:ref:refs/heads/main` is also issued to a `workflow_dispatch` on `main`. That is safe. But a wildcard such as `ref:refs/heads/*` would admit every branch, and a dispatch can target any branch [1].
- An `environment:` subject is issued only to jobs that reference the environment, and the environment's deployment branch rule is matched against `GITHUB_REF` [6]. With the rule "Selected branches: `main`", a job on a pull request merge ref (`refs/pull/N/merge`) or on another branch cannot use the environment [6].

GitHub also allows `job_workflow_ref` in a customized `sub` [4]. That pins the role to one workflow file. It needs a REST API call per repository and adds little when one workflow uses the environment (judgment).

### 1.3 What environment protection rules add

All rules below are available for public repositories on GitHub Free [6].

| Rule | What it does | Source |
| --- | --- | --- |
| Deployment branches and tags | "Only branches and tags that match your specified name patterns can deploy to the environment." Fork branches named like a protected branch "cannot deploy" | [6] |
| Required reviewers | Up to six people or teams, one of whom must approve before the job runs. A job "cannot access environment secrets until one of the required reviewers approves it". Self-review can be prevented | [6] |
| Wait timer | Delays the job by 1 to 43,200 minutes | [6] |
| Admin bypass | On by default: "administrators can bypass the protection rules" | [6] |

**Judgment.** For a single owner, the deployment branch rule does the work. A required reviewer would make every scheduled run wait for a click, and self-review prevention would block the only person who can approve. AWS recommends the same: "use deployment branches and tags to restrict which branches and tags can deploy" [13].

### 1.4 Documented rules for scheduled workflows

All from the `schedule` section of [1]:

- A scheduled workflow runs "on the latest commit on the default branch", and only if the workflow file exists there.
- The `schedule` event "can be delayed during periods of high loads". High load "includes the start of every hour", and "some queued jobs may be dropped". GitHub's advice is to pick another minute of the hour.
- "In a public repository, scheduled workflows are automatically disabled when no repository activity has occurred in 60 days."
- The shortest interval is every 5 minutes. Cron runs in UTC unless a time zone is set.

Other limits: a job on a GitHub-hosted runner may run for 6 hours [10]. Standard GitHub-hosted runners are free in public repositories [9]. Logs and artifacts are kept 90 days by default and need only read access [12], which for a public repository is everyone (judgment). GitHub-hosted Linux runners run on AWS or Azure in a "multi-cloud compute pool" [11]. GitHub names no region for them.

### 1.5 What a per-pull-request check can and cannot do with AWS here

**Can do, safely (judgment from the rules above):**

- Run every check of the old note's section 4.1 without credentials: unit tests, scorer self-tests, gold-set integrity, scripted-model smoke tests, policy and secret scans. These need no AWS.
- Read public files only. The test dataset is public by ADR-0030, so a pull request job can check it without a token.

**Cannot do safely:**

- Call Bedrock or any AWS API from a fork pull request. It gets no secret and no OIDC token [1, 2, 3].
- Call AWS from a same-repository pull request through a role that trusts `:pull_request` or a branch wildcard. The pull request's branch controls the workflow file [1], so the trust would cover any code pushed to any branch. Coding agents push branches in this project.
- Use `pull_request_target` to get credentials for pull request code [1, 7, 14].
- Read the private snapshot of ADR-0032 or write anything with personal data into logs or artifacts, because both are readable by anyone [12].

**What remains for model-backed checks on pull requests:** none by default. If the owner wants one, the documented safe shape is a `workflow_dispatch` run on `main` or a run after merge, both through the protected environment. Section 5 shows that several real projects do run paid model checks on same-repository pull requests. They accept the branch risk, often with a path filter and a check that the pull request does not come from a fork.
