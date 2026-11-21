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

## 2. Running a job on an instance that is usually stopped

The installation from P1 is one EC2 t4g.large with Docker Compose, a 30 GB gp3 volume and no inbound rule. It is reached through Session Manager port forwarding and is normally stopped. It does have a public IPv4 address for outbound traffic while it runs (P1, "Reaching the web app"). The project memories, the LiteLLM database and the scored snapshot of ADR-0032 live on that instance and in its S3 bucket. An evaluation run therefore needs the instance running, the harness started on it, and the instance stopped again.

### 2.1 Prices that every option shares

| Item | Price in eu-central-1 | Source |
| --- | --- | --- |
| EC2 t4g.large while running, plus its public IPv4 address | $0.0768 + $0.005 = $0.0818 an hour. Each start bills at least one minute, then per second | P1 [20, 22, 23], [27a] |
| EventBridge Scheduler | First 14 million invocations a month free, then $1.15 per million | [28] |
| Systems Manager Automation | $0.002 per step; `aws:executeScript` $0.00003 per second | [27] |
| Systems Manager Run Command on EC2 | "No additional charges" | [26, 27] |
| SNS email | First 1,000 email notifications a month free, then $2.00 per 100,000 | [29] |
| GitHub-hosted standard runner, public repository | Free | [9] |

The Systems Manager Price List file for Frankfurt (published 2026-11-18) has no Automation row [30], so the Automation price comes from the pricing page alone.

### 2.2 Options

**A. EventBridge Scheduler starts a Systems Manager Automation runbook.** A schedule calls `ssm:StartAutomationExecution` through a universal target (`arn:aws:scheduler:::aws-sdk:ssm:startAutomationExecution`) [17]. The runbook records the instance state, starts the instance with `aws:changeInstanceState` (default timeout one hour) [20], runs the harness with `aws:runCommand` [21], and stops the instance again. A step's `onFailure` property can send a failed run to the stop step [22].

**B. A timer inside the instance.** A systemd timer fires only while the instance runs. With `Persistent=true` it fires at the next boot if it missed a run "when the system was powered down" [31]. On an instance that is usually stopped, that means the evaluation would start whenever the consultant starts the instance to work (judgment). A variant lets EventBridge Scheduler call `ec2:startInstances` and lets a boot unit run the harness and shut the instance down. An operating system shutdown stops the instance by default (P1 [23]).

**C. A GitHub Actions workflow on `main` starts the same runbook.** A `schedule`, `push` or `workflow_dispatch` workflow assumes a role through OIDC (section 1.2), calls `StartAutomationExecution`, waits, and reads the scorecard from S3.

**D. A GitHub-hosted runner runs the stack itself.** The workflow starts the app, LiteLLM and PostgreSQL on the runner with Docker Compose, loads or rebuilds the project memory, and calls Bedrock directly with OIDC credentials.

| | A. Scheduler and Automation | B. Timer in the instance | C. GitHub workflow starts the runbook | D. Stack on a GitHub runner |
| --- | --- | --- | --- | --- |
| AWS cost of the trigger per run (estimate) | About $0.01: 5 Automation steps at $0.002 [27]; Scheduler free [28] | $0 for the variant with Scheduler starting the instance [28] | As A, about $0.01 | $0 |
| Compute per run of 1 hour (estimate) | $0.08 on EC2 [27a] | $0.08 | $0.08 on EC2; the runner that waits is free [9] | $0 on the runner [9]. Rebuilding a small project memory adds about $3.60 of Cohere Embed v4 and at least 100 minutes (P1, P2) unless it is restored from S3 |
| Credentials, and where they live | Only AWS roles: the Scheduler role (start one runbook), the Automation role (start, stop, describe the instance, send one command), the instance role (Bedrock, S3). Nothing outside AWS | Only the instance role, plus `ec2:StartInstances` for Scheduler | As A, plus one role that trusts the `evaluation` environment and may start one runbook and read one S3 prefix. GitHub holds only the role ARN, which is not secret | A role that GitHub can assume with Bedrock invoke rights, `bedrock-mantle:CreateInference` and read access to the snapshot bucket |
| Result and exit status | Scorecard written to S3 by the harness. The Automation status ("Success", "Failed", "TimedOut") is an EventBridge event [32] that a rule can send to SNS email [29]. Run Command returns at most the first 24,000 characters of stdout [25], so the scorecard must not travel through stdout | The harness must upload to S3 and publish to SNS itself. A hung run is caught only by a timeout set on the unit (judgment) | As A, plus a check run or commit status on the `main` commit, a job summary and a failed workflow, all with `GITHUB_TOKEN` | Job summary and artifacts on GitHub. These are public [12], so only body-free numbers may go there |
| Fits the CloudFormation template | Yes: `AWS::Scheduler::Schedule` [18], `AWS::SSM::Document` with `DocumentType: Automation` [19], IAM roles, an SNS topic and an EventBridge rule | Yes, but the unit file lives in the instance's user data or image, not as a resource | Yes: as A plus `AWS::IAM::OIDCProvider` and one role. The workflow file lives in this repository | The role only. The stack definition must also run outside AWS |
| Main weakness | The Run Command execution timeout must be raised for a run of more than an hour (the default of `AWS-RunShellScript` is not verified here). Scheduler delivers "at least once" [16], so the harness must refuse a second run for the same date | Cannot tell a scheduled boot from a consultant's start without extra state | GitHub may delay or drop scheduled runs and disables them after 60 days without activity [1]. Job limit 6 hours [10] | Personal data of the ADR-0032 snapshot would be processed on a runner in a region GitHub does not name [11]. The managed retrieval arm still runs in AWS (ADR-0017). Runner hardware varies, so latency is not comparable across runs |

Notes on the table:

- Automation runs started with a role may run longer than 12 hours. In a user's context the limit is 12 hours [23]. `StartAutomationExecution` allows 1 call per second [23], which does not matter here.
- `aws:runCommand` delivery waits until `TimeoutSeconds` (30 to 2,592,000) if the agent is not online yet [21, 24]. This covers the minute or so after a start until the SSM Agent registers (judgment; I measured nothing).
- If the instance was already running when the schedule fired, because a consultant was using it, the runbook should leave it running at the end. Option A can branch on the recorded start state (judgment).

### 2.3 Judgment

Option A fits best for scheduled runs. It keeps every credential in AWS, costs about one cent per run beyond the instance time, survives a repository without activity, and fits the template P1 recommends. Option C is a good second trigger for runs a person wants now, for example after a merge, because it reuses A's runbook and reports the verdict on the commit. Option B alone does not work on a stopped instance. Option D breaks the rule of ADR-0032 that the scored snapshot stays in the installation's AWS account, and it rebuilds or downloads the project memory every time.
