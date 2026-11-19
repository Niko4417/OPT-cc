# How to build, start and tear down an installation on AWS

Date: 2026-11-20. Ticket: [#46](https://github.com/Niko4417/OPT-cc/issues/46). Map: [#1](https://github.com/Niko4417/OPT-cc/issues/1). Context: [ADR-0010](https://github.com/Niko4417/OPT-cc/blob/main/docs/adr/0010-count-a-consultancy-tenant-as-third-party-cloud.md), [ADR-0013](https://github.com/Niko4417/OPT-cc/blob/main/docs/adr/0013-run-v1-on-aws-as-a-proof-of-concept-and-leave-on-device-and-client-hosted-for-later.md), [ADR-0014](https://github.com/Niko4417/OPT-cc/blob/main/docs/adr/0014-require-eu-processing-for-every-third-party-cloud-route.md), [ADR-0015](https://github.com/Niko4417/OPT-cc/blob/main/docs/adr/0015-keep-v1-single-user-with-no-shared-team-server.md), [ADR-0018](https://github.com/Niko4417/OPT-cc/blob/main/docs/adr/0018-ingest-github-repositories-with-our-own-glue-job-into-our-own-chunk-store.md), [ADR-0023](https://github.com/Niko4417/OPT-cc/blob/adr/project-memory-bundle/docs/adr/0023-sign-and-encrypt-every-bundle-with-one-key-per-installation.md) and [ADR-0024](https://github.com/Niko4417/OPT-cc/blob/adr/project-memory-bundle/docs/adr/0024-move-a-bundle-between-installations-as-a-file-or-an-expiring-s3-link.md) (both on the branch `adr/project-memory-bundle`), [cloud provider research](https://github.com/Niko4417/OPT-cc/blob/research/cloud-provider/docs/research/cloud-provider.md) (P1), [AWS models research](https://github.com/Niko4417/OPT-cc/blob/research/aws-models/docs/research/aws-models.md) (P2), [stack scoring research](https://github.com/Niko4417/OPT-cc/blob/research/stack-scoring/docs/research/stack-scoring.md) (P3).

## Question

How does a consultant get a working installation of the app on AWS in Frankfurt (eu-central-1), and how is it removed again? The owner's scenario is that the app works in the owner's installation, and a second consultant gets their own by taking the repository and running one install command, then opening a web app. The ticket asks for one recommendation on each of six points: which AWS account, how the installation is created, what runs the app, how the single user reaches the web app and logs in, the one-time steps per account for Bedrock, and cost limit and teardown. The base stack is still open ([#10](https://github.com/Niko4417/OPT-cc/issues/10)), so each recommendation states what depends on it.

## Answer

| Point | Recommendation | Main reason |
| --- | --- | --- |
| AWS account | One AWS account per installation. During the proof of concept the owner creates these as member accounts of one AWS organization and pays one bill | The account is the isolation, quota and billing boundary that AWS provides without any policy work, and a member account can later move whole into a consultancy's organization |
| How the installation is created | One CloudFormation template in the repository and a short wrapper script with `install`, `open`, `stop`, `start` and `destroy` | No tool beyond the AWS CLI, no state file on the consultant's laptop, no license question. OpenTofu (MPL-2.0) is the alternative if the install code should later cover a client's Azure tenant |
| What runs the app | One EC2 instance (t4g.large, 2 vCPU, 8 GiB) with Docker Compose and one 30 GB gp3 volume | The git clone, the plain files and the stores need a disk that outlives a restart. A stopped instance costs only its disk |
| Reaching the web app | No public address. The consultant opens a Session Manager port forwarding session and browses to `http://localhost:<port>`. The app's own login stays on | AWS checks the consultant's IAM identity before any packet reaches the app. No domain, certificate, load balancer or Cognito is needed |
| One-time steps per account | A scripted checklist: first call to each AWS Marketplace model with admin rights, a quota check, a cost allocation tag, a budget | Bedrock grants model access on first use, but per account, and quotas are per account and Region |
| Cost limit and teardown | Two limits. A LiteLLM budget stops model calls at once, and an AWS budget with two actions stops the instance and denies Bedrock when the bill arrives. `destroy` empties the bucket, deletes the knowledge bases and the CloudFormation stack, then lists leftovers by tag | AWS billing data can be a day old, so an AWS budget alone is an alarm, not a cap |

Cost per installation, infrastructure only, list prices in USD without VAT for Frankfurt, read on 2026-11-20 [20, 22]:

| State | Per month |
| --- | --- |
| Stopped (idle) | $4.08 |
| Running 168 hours (8 hours on 21 days) | $17.82 |
| Running all month (730 hours) | $63.79 |

Model and knowledge base usage come on top. For the small project memory of the cloud provider research this adds about $7.94 a month, of which $2.50 is Managed Knowledge Base storage that is also charged while the instance is stopped (P1). The cloud provider research assumed a 1 vCPU, 2 GB Fargate task at $41.45 a month. The EC2 instance replaces that line.

The AWS fee difference between one account and several is $0. AWS charges nothing for an account, for AWS Organizations or for consolidated billing [4], and every resource of an installation costs the same in either layout. Two small effects remain. A budget with actions costs $0.10 a day after the first two [22], and customers who are new to AWS get $100 to $200 of sign-up credits [7]. I could not verify whether those two allowances count per member account or per organization.

## Which AWS account

The ticket names two options. I split the first one, because it matters who pays.

- **Option A, standalone account.** Each consultant signs up for their own AWS account and applies the install there.
- **Option B, member account.** The owner turns the existing account into the management account of an AWS organization and creates one member account per installation.
- **Option C, shared account.** Every installation is a separate CloudFormation stack inside the owner's one account.

| | A, standalone account | B, member account | C, shared account |
| --- | --- | --- | --- |
| Isolation of bucket, installation key and IAM | The account boundary. Nothing is shared | The account boundary. The management account can assume `OrganizationAccountAccessRole` in every member account [2] | IAM policies that the install template writes. Every administrator of the account can read every installation key |
| Quotas | Own quotas | Own quotas | Shared by all installations |
| Who pays | The consultant, with their own payment method | The owner, on one bill with a cost report per account [4] | The owner, with a split only through tags |
| Access the second consultant needs | They own the account | A sign-in to their member account, which the owner hands over | An IAM identity in the owner's account |
| Bedrock one-time steps | Per account | Per account, the Anthropic form once for the organization [35] | Once |
| Move to a consultancy's account | Invite the account into the consultancy's organization [6] | The same | Rebuild every installation |

### Isolation

AWS describes the account as the unit of isolation. Provisioning resources in dedicated accounts gives "clear security, access, and billing boundaries" [1]. In options A and B an S3 bucket, a secret or an IAM role of one installation is invisible to another installation unless someone writes a policy that shares it.

In option C the same isolation has to be built. The template gives each installation its own IAM role and limits that role to the bucket and the secrets of its installation, by resource name or by tag. Secrets Manager supports the tag conditions for this [12]. This works for the app's own role. It does not work for people. A consultant who runs the install command creates IAM roles, and anyone who can create roles in an account can give a role any permission. So in a shared account either the owner runs every install, or the second consultant is in practice an administrator who can read the owner's installation key. ADR-0023 uses that key to prove which installation exported a bundle. In a shared account the signature proves only that the bundle came from someone with access to the account.

V1 holds public or synthetic data, so the confidentiality risk of option C is low today. The trust model of the installation key is still tested only in options A and B.

### Quotas

Service quotas "are allocated for each account" [1]. Three of them matter here.

- Bedrock token and request quotas apply per model, per Region and per account [38]. Cohere Embed v4 through the EU inference profile has a default of 300,000 tokens and 2,000 requests a minute [38]. The first embedding run of a small project memory (30 million tokens, P1) therefore takes at least 100 minutes, and that of a mid-size one (200 million tokens) at least 11 hours. In option C two installations that index at once share this quota.
- The default quota for standard On-Demand EC2 instances is 5 vCPUs per Region [42]. One t4g.large uses 2. A shared account runs two installations and needs an increase for the third. Amazon EC2 raises the quota by itself as usage grows [42].
- A new organization holds 10 accounts by default, and "newly created accounts and organizations might experience a quota below the default" [3]. The management account can ask for more.

### Cost attribution

"An account is the default means by which AWS costs are allocated" [1]. In options A and B the bill per installation needs no setup.

In option C the split works through tags and needs three things.

- Every resource carries an `installation` tag. CloudFormation copies tags set on a stack to the resources it creates, which the Secrets Manager resource reference confirms for secrets [18]. I did not check every resource type.
- Someone activates the tag as a cost allocation tag in the payer account. A new tag key takes up to 24 hours to appear and up to 24 more to activate [9]. Costs from before the activation stay untagged [10].
- Bedrock model calls have no resource to tag. Bedrock records the IAM role of every call on both endpoints, so one role per installation splits the model cost. The role's tags reach Cost Explorer after the same activation, and the finest grain is one usage type per day [10].

### Access for the second consultant

- **Option A.** The consultant needs a payment method, a phone number that can receive calls or SMS, and an email address [8]. They are the administrator of their account. No one else has access.
- **Option B.** The owner runs `aws organizations create-account` with an email address that no other AWS account uses. AWS copies the contact data from the management account, and the procedure asks for no payment method [2]. The owner reaches the new account through `OrganizationAccountAccessRole` [2] and creates the consultant's sign-in there, for example an IAM Identity Center user, which costs nothing [11]. IAM Identity Center signs people in to AWS, not to the app, so it does not touch the rule of ADR-0015 against single sign-on for the app. Inside their account the consultant can be an administrator without any risk to other installations.
- **Option C.** The consultant gets an IAM identity in the owner's account. If the owner runs the install, that identity needs only three rights on the consultant's own EC2 instance, selected by tag: start a Session Manager session, start the instance and stop it. If the consultant runs the install, see the isolation section.

### When a consultancy account replaces the owner's

AWS resources do not move between accounts. Accounts move between organizations.

- **Options A and B.** The consultancy's management account sends an invitation, and the account accepts it. Since November 2025 the account does not have to leave its old organization first [6]. The S3 bucket, the installation key, the EC2 instance and all IAM roles stay as they are. Billing, service control policies and access to old cost data change [6]. Three conditions apply: an account created inside an organization must be at least four days old, the receiving organization at least seven days old, and both must have the same AWS seller of record [6]. The owner should delete `OrganizationAccountAccessRole` or hand it over, because it still trusts the old management account [6].
- **Option C.** Each installation is installed again in the consultancy's account, and each project memory moves as a project memory bundle (ADR-0021 to ADR-0027). The new installation creates a new installation key, so the other consultants must replace the public key in their lists of trusted signers, and bundles that were encrypted to the old key can no longer be opened (ADR-0023).

Two points hold in every option. A consultancy that restricts Regions with service control policies must allow the six Regions of the EU inference profiles or add the exception AWS documents [37]. And ADR-0013 says the installation runs "in the consultancy's tenant", but AWS has no tenant. The review should say whether tenant means one account or one organization. With one account per installation it means the organization.

### Recommendation

Use one AWS account per installation, as member accounts of the owner's organization while the owner pays (option B). It gives isolation, separate quotas and a bill per installation with no policy work, lets the second consultant run the install command as an administrator of their own account, and, like option A, lets an installation move into a consultancy's organization unchanged. The price is one CLI call per account and the Bedrock one-time steps in each account.

The template should still prefix and tag every resource with the installation name and create nothing that is unique per account. Then the same install command also works in option A and in option C, and the owner's first installation can stay in the existing account. The choice of account layout then stays open after the first installation.

This does not depend on the base stack.

## How the installation is created

| Tool | License | What it needs on the consultant's laptop | Where the record of created resources lives |
| --- | --- | --- | --- |
| CloudFormation template with the AWS CLI | AWS service with no charge for AWS resource types [18]. The AWS CLI is Apache-2.0 [19] | AWS CLI | In CloudFormation, in the account |
| OpenTofu 1.13 | MPL-2.0 [14]. The AWS provider is MPL-2.0 [15] | `tofu` binary and the AWS CLI | A state file on the laptop, or in an S3 bucket that must exist first [16] |
| Terraform 1.17 | Business Source License 1.1 [13] | `terraform` binary and the AWS CLI | The same as OpenTofu |
| AWS CDK 2.277 | Apache-2.0 [17] | Node.js, the CDK CLI and the AWS CLI | In CloudFormation, plus a bootstrap stack per account and Region [17] |
| Script with the AWS CLI or boto3 | Apache-2.0 | AWS CLI or Python | Nowhere |

**License check.** The rule is that licenses must allow free commercial use.

- Terraform's license is not an open-source license. Its additional use grant allows production use unless the use includes "offering the Licensed Work to third parties on a hosted or embedded basis in order to compete with IBM Corp.'s paid version(s)", and it states that "using the Licensed Work(s) for internal purposes within an organization is not considered a competitive offering" [13]. Installing our own app with Terraform is therefore allowed. The grant still puts a condition on commercial use that OpenTofu, the fork with the same language, does not have. Nothing here needs Terraform.
- OpenTofu is under the Mozilla Public License 2.0 [14]. The `hashicorp/aws` provider it uses is also MPL-2.0 [15].
- The AWS CDK is Apache-2.0 [17].
- CloudFormation is a service, not software we distribute. AWS charges nothing for resource types in the `AWS::*` namespace [18].

**Fit for one install command.**

- CloudFormation keeps the list of what it created in the account. `aws cloudformation delete-stack` deletes "all resources in that stack" unless a deletion policy retains them [18]. Nothing can be lost on a laptop. It supports the Managed Knowledge Base type [18]. Its limits are that it cannot delete a bucket that still has objects [18], and that it can generate a random password for a secret but not an SSH key [18].
- OpenTofu documents the two teardown flags this installation needs: `force_destroy` on a bucket deletes its objects, and `recovery_window_in_days = 0` deletes a secret at once [15]. It also supports the Managed Knowledge Base type [15]. Its state file is the weak point for a single consultant. If the file stays on the laptop and is lost, `tofu destroy` no longer knows what to delete. If it goes to S3, that bucket is a resource the install must create before the install, and remove after the teardown [16].
- The CDK produces CloudFormation and adds Node.js and a bootstrap stack with an S3 bucket, an ECR repository and IAM roles in every account [17]. The bootstrap stack is separate from the app's stack, so it needs its own teardown. At the size of this installation, one EC2 instance and a handful of supporting resources, the CDK saves nothing.
- A plain script has no record of what it created, so teardown has to search.

**Recommendation.** Write one CloudFormation template and a wrapper script. The consultant needs the AWS CLI anyway to open the web app, so the install adds no tool. The wrapper covers what the template cannot: empty the bucket before deletion, make the first call to each Marketplace model, and print the checklist of one-time steps. Choose OpenTofu instead if the owner wants the install code to carry over to a client-hosted installation in Azure, which the map names as the runner-up. That is a judgement about a later effort, not a need of V1.

One design note follows from ADR-0023. The installation key should be created on the EC2 instance at first start with `ssh-keygen` and written into an empty secret that the template created. Then the private key never exists on the consultant's laptop, in a template parameter or in a state file. This is my suggestion, not a tested design.

The tool choice does not depend on the base stack. The list of containers in the template does.

## What runs the app

The installation runs the base stack's web app, LiteLLM, the glue ingestion job, one git clone per connected repository with the `github/` files next to it (ADR-0018), and the chunk store. LiteLLM needs a PostgreSQL database for budgets and virtual keys [30] (P3).

| Option | Fit | Cost for 2 vCPU and 8 GB, all month |
| --- | --- | --- |
| One EC2 instance with Docker Compose | One disk for everything. Stop and start keep the disk | $56.06 (t4g.large) [20] |
| ECS on Fargate | Persistent files need Amazon EFS. An EBS volume attached through a service is ephemeral [24] | $78.27 on Arm, $97.82 on x86 [21] |
| App Runner | Closed to new customers [25] | Not available |
| ECS Express Mode | Fargate plus an Application Load Balancer, made for "stateless containerized applications" [26] | Fargate price plus $19.71 for the load balancer [22] |

**EC2.** A t4g.large has 2 vCPUs and 8 GiB and costs $0.0768 an hour in Frankfurt. A t4g.medium with 4 GiB costs $0.0384, and the x86 t3.large $0.096 [20]. A stopped instance is "not charged for usage or data transfer fees", but its EBS volumes are [23]. The data on EBS volumes persists across a stop, and the public IPv4 address that AWS assigned is released [23]. A gp3 volume costs $0.0952 per GB a month, and a public IPv4 address $0.005 an hour [20, 22]. Each start is billed for at least one minute [23]. An operating system shutdown stops the instance by default [23], so a nightly `shutdown` inside the instance is enough for an automatic stop.

**Fargate.** Arm capacity costs $0.03725 per vCPU-hour and $0.00409 per GB-hour, x86 $0.04656 and $0.00511 [21]. For a Fargate service the storage that persists is Amazon EFS or the newer S3 Files. An EBS volume persists only on a standalone task [24]. Port forwarding to a Fargate task works through ECS Exec [31, 34]. EFS Standard costs $0.36 per GB a month in Frankfurt and One Zone $0.192, against $0.0952 for EBS [22]. For 30 GB that is $10.80 against $2.86. A service scaled to zero tasks costs only storage, like a stopped instance. Fargate would remove operating system updates from the consultant's duties, which matters little for a proof of concept that is torn down again.

**Cost per installation on EC2.** The assumptions are a t4g.large, a 30 GB gp3 volume, one public IPv4 address while running, three secrets (the installation key, one GitHub token per ADR-0018 and the app's login secret) and 1 GB in S3.

| Line | Stopped | 168 hours | 730 hours |
| --- | --- | --- | --- |
| EC2 t4g.large, $0.0768 an hour [20] | $0 | $12.90 | $56.06 |
| Public IPv4 address, $0.005 an hour [22] | $0 | $0.84 | $3.65 |
| EBS gp3, 30 GB at $0.0952 [20] | $2.86 | $2.86 | $2.86 |
| Secrets Manager, 3 secrets at $0.40 [22] | $1.20 | $1.20 | $1.20 |
| S3 Standard, 1 GB at $0.0245 [22] | $0.02 | $0.02 | $0.02 |
| **Total** | **$4.08** | **$17.82** | **$63.79** |

With a t4g.medium the all-month total is $35.76. Session Manager, CloudFormation and AWS Budgets without actions add nothing [18, 22, 31]. Logs sent to CloudWatch would cost $0.63 per GB ingested [22]. Keeping container logs on the disk avoids that. A managed PostgreSQL on Amazon RDS would add at least $13.87 a month for a db.t4g.micro plus $0.137 per GB [22], so PostgreSQL runs as a container on the instance.

**What depends on the base stack.**

- Keiko "serves loopback only" and "the server always binds `127.0.0.1`" on port 1983 [27]. Its README describes npm and desktop installs and names no container image [27]. On an EC2 instance it can run on the host or in a container that shares the host's network. Port forwarding to the instance reaches a loopback port, so the recommended access method fits Keiko as it is. A load balancer would not reach it.
- Open WebUI and LightRAG publish container images. LightRAG's compose file runs one container on port 9621 with its state in mounted folders [28]. Open WebUI listens on port 8080 in its container [29].
- Memory is not measured. Keiko's issue #4456 reports 4.27 GB for one question over about 10,600 files (P3), and LiteLLM's guide starts at 1 vCPU and 4 GB per worker for load tests [30]. This is why the recommendation is 8 GiB and not 4. The trial run should measure it.
- I did not check whether each candidate's image exists for Arm. If one does not, the t3.large at $0.096 an hour replaces the t4g.large, which adds $14.02 for a full month.
- Keiko and Open WebUI keep state in SQLite (P3). On Fargate that file would sit on EFS, a network file system. I did not check whether either project supports that.

**Recommendation.** Run one EC2 instance per installation with Docker Compose, t4g.large, one 30 GB gp3 volume, in a public subnet with a security group that has no inbound rule.

## How the single user reaches the web app and logs in

**Session Manager port forwarding, recommended.** The wrapper's `open` command runs `aws ssm start-session --target <instance> --document-name AWS-StartPortForwardingSession` with the app's port, and the consultant browses to `http://localhost:<port>` [31]. Session Manager works "without the need to open inbound ports, maintain bastion hosts, or manage SSH keys", and IAM policies decide who may start a session to which instance [31]. It costs nothing for EC2 instances [31]. The laptop needs the AWS CLI and the Session Manager plugin, which is Apache-2.0 [19, 31]. The instance needs the SSM Agent, an instance role with the `AmazonSSMManagedInstanceCore` policy and outbound HTTPS [31]. A public IPv4 address with no inbound rule provides the outbound path for $3.65 a month while running. Without a public address the instance would need a NAT gateway at $0.052 an hour, $37.96 a month, because it must also reach GitHub and the container registries [20].

Two limits are documented. Session Manager cannot log the content of port forwarding sessions [31], and each consultant needs AWS credentials for the account of their installation.

**Login.** Access then has two checks and no single sign-on. The first is the consultant's AWS sign-in. The second is the app's own login.

- In Open WebUI the first account created "is the administrator", and sign-up "switches itself off once the admin account exists" [29].
- In LightRAG the accounts come from the `AUTH_ACCOUNTS` setting with a `TOKEN_SECRET` of at least 32 bytes, and its `env.example` says to expose the server only together with `LIGHTRAG_API_KEY` or `AUTH_ACCOUNTS` [28]. The install generates both values and keeps them in Secrets Manager.
- In Keiko, `keiko start` opens "an authenticated browser session at `http://127.0.0.1:1983`" [27]. I did not check how that session is authenticated when the browser runs on another machine at the far end of a tunnel.

**A public HTTPS address, the alternative.** An Application Load Balancer can require a sign-in through an Amazon Cognito user pool before it forwards a request. This needs an HTTPS listener, so also a certificate and a domain name [32]. The load balancer costs $0.027 an hour, $19.71 a month, plus capacity units and a public IPv4 address in each of at least two subnets [22, 32]. A Cognito user pool is free below 10,000 monthly active users [22]. This path gives a link that opens in any browser. The load balancer alone costs about five times a stopped installation. It is worth that only when someone without AWS credentials must use the app, which V1 excludes (ADR-0015).

**A consequence for ADR-0024.** Export shows a download link to the installation's S3 bucket, and ADR-0024 notes that the lifetime of that link depends on how the installation signs it. A presigned URL signed with the role credentials of an EC2 instance is "valid for the duration of the role credentials (typically 6 hours)", and for ECS tasks the credentials "typically rotate every 1-6 hours" [33]. Only an IAM user's long-lived key can sign a link for up to 7 days [33]. With the recommended instance role, a link lasts at most about 6 hours and possibly much less, depending on when the credentials were issued.

## One-time steps per account

**Model access.** Access to many Bedrock foundation models "is enabled by default with the correct AWS Marketplace permissions", in all commercial Regions where the model is supported, with no request page [35]. Three conditions remain, and each applies per account.

- Models sold through AWS Marketplace are subscribed on first use. The first call starts the subscription in the background, which can take up to 15 minutes. The caller of that first call needs the permissions `aws-marketplace:Subscribe`, `aws-marketplace:Unsubscribe` and `aws-marketplace:ViewSubscriptions`, and the account needs a valid payment method. After that, no caller needs Marketplace permissions [35]. Claude Haiku 5.5 has a Marketplace product ID [39]. Models from Amazon, Google, Qwen, NVIDIA, MiniMax, Mistral AI and Z.AI, and the gpt-oss models, have none and need no subscription [35]. I did not check the product IDs of Claude Sonnet 5.5, Cohere Embed v4 and Cohere Rerank 3.5.
- Anthropic asks first-time customers for use case details, but only for models "earlier than Claude Opus 4.7". The form is filed once per account or once at the organization's management account, and member accounts inherit it [35]. The shortlisted Claude Haiku 5.5 and Sonnet 5.5 are newer, so by this rule they need no form. The documentation lists only older models as examples, so this is my reading. Claude Haiku 4.5, the evidence anchor in the AWS models research, would need the form.
- Access "is determined separately for each AWS account, so accounts in the same AWS Organization can have different access" [35].

The install script should therefore make one test call to every Marketplace model with the installer's own administrator credentials. The role of the EC2 instance then needs no Marketplace permission.

**EU inference profiles.** A request to an EU profile stays in the EU geography, at the price of the source Region [37]. From Frankfurt, the EU profiles of Claude Haiku 5.5 and Cohere Embed v4 route to Frankfurt, Stockholm, Milan, Spain, Ireland and Paris [39]. Zurich is a destination only for calls made from Zurich [39]. This corrects the region list in the AWS models research, which named Zurich in place of Spain. All six Regions are in the EU, so ADR-0014 holds.

- The Regions do not have to be enabled in the account [37].
- The instance role must be allowed to invoke the inference profile and the foundation model in the source Region and in every destination Region [37].
- A deny statement with the condition `aws:RequestedRegion` equal to `unspecified` blocks global inference profiles [37]. In the instance role this enforces ADR-0014 in IAM, in addition to the LiteLLM route list.

**The `bedrock-mantle` endpoint.** Gemma 4 runs "only on the `bedrock-mantle` endpoint", under the path `/openai/v1` [39]. That endpoint needs the IAM action `bedrock-mantle:CreateInference`, has no cross-Region inference, and has quotas of its own [36, 38]. It accepts SigV4 requests or a Bedrock API key. A short-term key lasts at most 12 hours, and a long-term key creates an IAM user and is "recommended only for exploration" [40]. LiteLLM's Bedrock provider signs with the boto3 credential chain and also accepts a Bedrock API key [30]. The route should sign with the instance role so that no key has to be stored or renewed. Whether LiteLLM's route to `bedrock-mantle` can do that is a question for the LiteLLM routes review.

**Quotas that matter.**

| Quota | Default | Source |
| --- | --- | --- |
| Cross-Region tokens per minute, Claude Haiku 5.5 | 10,000,000 | [38] |
| Cross-Region tokens and requests per minute, Cohere Embed v4 | 300,000 and 2,000 | [38] |
| Tokens per day across all models on `bedrock-runtime` | 700,000,000 | [38] |
| Gemma 4 on `bedrock-mantle`, input and output tokens per minute | Shown only in the Service Quotas console of the account | [38] |
| Managed knowledge bases per account and Region | 10,000 | [41] |
| Raw data per managed knowledge base | 10 TB | [41] |
| Running standard On-Demand EC2 instances | 5 vCPUs | [42] |
| Accounts in an organization | 10 | [3] |

The quotas of an account "might be lower than the default values" depending on account age, usage and payment history [38]. The install script should read the applied values with the Service Quotas API and print them. Increases for `bedrock-mantle` go through a support case, not the Service Quotas console [38].

**Other one-time steps.**

- A Managed Knowledge Base needs a service role, and the creator needs `iam:PassRole` for it. The managed embedding model needs no model access [41]. The template creates the role.
- The payer account activates the `installation` cost allocation tag [9].
- In option B only, the owner creates the organization once. That moves an account on the free plan to the paid plan [6, 7].

## Cost limit and teardown

### Cost limit

An AWS budget tracks the bill, and the bill is late. "AWS billing data, which Budgets uses to monitor resources, is updated at least once per day" [43], and Bedrock cost "can take up to 24 hours to appear" in Cost Explorer [10]. Within such a day an installation could use the whole daily token quota. At the input price of Claude Haiku 5.5 ($0.11 per million tokens, P2), 700 million tokens cost about $77. At the input price of Claude Sonnet 5.5 ($2.20) they cost about $1,540. This is my estimate of the upper bound, from the default quota and input prices only.

The limit therefore has two layers.

- **LiteLLM, immediate.** A `max_budget` with a `budget_duration` under `litellm_settings` caps the spend of all proxy calls, and keys can carry their own budget. LiteLLM enforces every budget "against spend read from the database", so it needs PostgreSQL. Without a database the global budget fails open [30]. Global, key, team and user budgets are in the MIT-licensed part. Budgets per model on a key need an Enterprise license [30]. This layer covers every model call, because the app talks only to LiteLLM. It does not cover the EC2 instance or Managed Knowledge Base storage.
- **AWS Budgets, late but complete.** In options A and B a monthly cost budget lives in the installation's account and covers all of it. In option C it needs a filter on the `installation` tag. A budget action can apply an IAM policy, apply a service control policy, or stop chosen EC2 or RDS instances in the same account, automatically or after approval [43]. Two actions at 100 percent fit this installation: attach a deny policy for `bedrock:*` and `bedrock-mantle:*` to the instance role, and stop the instance. Monitoring is free, and the first two budgets with actions are free. Further ones cost $0.10 a day each [22, 43].

Idle cost has a simple control. The wrapper offers `stop`, and a nightly shutdown inside the instance stops it without AWS tooling [23].

### Teardown

`destroy <name>` should do these steps in order.

1. Offer a last export. Deleting the installation key means that bundles encrypted to this installation can never be opened again (ADR-0023).
2. Delete the managed knowledge bases of the installation. The app creates one per project memory while it runs, so they are not in the CloudFormation stack, and their storage costs $5 per GB a month until deleted (P1).
3. Empty the S3 bucket. "You can only delete empty buckets" [18].
4. Run `aws cloudformation delete-stack` and wait. If a resource fails to delete, the stack reports `DELETE_FAILED` and keeps the remaining resources until the next attempt [18].
5. Delete the secrets with `--force-delete-without-recovery`. A normal deletion keeps a secret for a recovery window of at least seven days, without charge [44].
6. List what still carries the tag with `aws resourcegroupstaggingapi get-resources --tag-filters Key=installation,Values=<name>`, which returns the tagged resources of a Region [45], and exit with an error if anything is left.

In options A and B closing the account is a second way to stop all charges. Charges stop at closure and restart only if the account is reopened, and AWS deletes the content after 90 days [5]. A closed member account counts against the organization's limit of 10 accounts for those 90 days [3, 5], so it does not replace `destroy`.

## What depends on the base stack

- The containers in the Compose file and their images, and whether an Arm image exists for each.
- The instance size. Keiko's reported 4.27 GB for one question (P3) is the largest known figure.
- Whether the app can listen beyond loopback. Keiko cannot [27]. The recommended access does not need it.
- How the first user is created: the first sign-up in Open WebUI, generated accounts in LightRAG, an authenticated browser session in Keiko [27, 28, 29].
- Whether the stack needs PostgreSQL beyond LiteLLM's database.

The account layout, the CloudFormation template approach, the access through Session Manager, the Bedrock steps, the budgets and the teardown steps do not depend on it.

## Findings for other tickets

- **[#18](https://github.com/Niko4417/OPT-cc/issues/18), LiteLLM routes.** Gemma 4 runs only on `bedrock-mantle` under `/openai/v1` with the action `bedrock-mantle:CreateInference`. Bedrock API keys last 12 hours or create an IAM user. LiteLLM budgets need PostgreSQL. An IAM deny on `aws:RequestedRegion` equal to `unspecified` blocks global inference profiles, which supports the gate of 0 off-policy model calls.
- **[#43](https://github.com/Niko4417/OPT-cc/issues/43), models per task.** The EU profile from Frankfurt routes to six EU Regions including Spain and not Zurich. Cohere Embed v4 is limited to 300,000 tokens a minute by default, so a first embedding run takes at least 100 minutes for a small and 11 hours for a mid-size project memory. Claude Haiku 5.5 needs a Marketplace subscription on first use, and Claude Haiku 4.5 also needs the Anthropic form.
- **[#12](https://github.com/Niko4417/OPT-cc/issues/12), trial run, and ADR-0024.** A download link signed with the instance role lasts at most about 6 hours.
- **[#31](https://github.com/Niko4417/OPT-cc/issues/31), coding agents.** On AWS the folder the app owns is on the EC2 volume, not on the consultant's machine, where the coding agent research placed the agents and the local MCP server. I did not research how agents reach it.
- **[#10](https://github.com/Niko4417/OPT-cc/issues/10), base stack.** Keiko binds only to loopback and names no container image in its README. Open WebUI and LightRAG publish images.
- **[#47](https://github.com/Niko4417/OPT-cc/issues/47), review of this research.** ADR-0013 says "the consultancy's tenant". The review should fix whether that is one AWS account or one AWS organization.

## Not verified

- Whether deleting a CloudFormation stack removes a secret at once or schedules it with a recovery window. The teardown steps call the forced deletion explicitly for that reason.
- The default Fargate vCPU quota, and the default quotas of Gemma 4 on `bedrock-mantle`.
- Whether Claude Haiku 5.5 and Sonnet 5.5 need the Anthropic use case form. The rule in [35] exempts models from Claude Opus 4.7 on, and names no 5.5 model.
- Marketplace product IDs of Claude Sonnet 5.5, Cohere Embed v4 and Cohere Rerank 3.5.
- Memory use and Arm images of Keiko, Open WebUI and LightRAG on AWS, and SQLite on EFS.
- How Keiko authenticates a browser session that arrives through a tunnel.
- Whether CloudFormation copies stack tags to every resource type used here, and whether Managed Knowledge Base cost can be split by tag.
- Whether the two free budgets with actions and the sign-up credits count per member account or per organization.
- That an AWS budget can filter on a user-defined tag. The part of the budget documentation I read names filters without listing them.
- That LiteLLM can sign requests to `bedrock-mantle` with SigV4 from the instance role.
- How the owner hands a member account to a consultant in detail. I read only the page on creating the account.
- That the SSM Agent is preinstalled on the specific machine image the template will use. AWS states it for "some" of its images [31].
- The upper bound of $77 to $1,540 a day is an estimate.
- I deployed nothing. No command in this note was run against an AWS account.

## Method

I read AWS documentation, the AWS Price List bulk files for eu-central-1, license files and project documentation on 2026-11-20. Prices come from the Price List files named in [20] to [22]. Each file states its publication date, between 2026-09-11 and 2026-11-19. License texts come from the `LICENSE` file on each repository's default branch. Release numbers come from the GitHub API. Model prices, project memory sizes and usage assumptions come from the cloud provider research (P1) and the AWS models research (P2) and are not checked again. Monthly figures use 730 hours.

## Sources

Accounts, billing and access:

1. [Benefits of using multiple AWS accounts](https://docs.aws.amazon.com/whitepapers/latest/organizing-your-aws-environment/benefits-of-using-multiple-aws-accounts.html), AWS whitepaper Organizing Your AWS Environment Using Multiple Accounts
2. [Creating a member account in an organization](https://docs.aws.amazon.com/organizations/latest/userguide/orgs_manage_accounts_create.html), AWS Organizations user guide
3. [Quotas and service limits for AWS Organizations](https://docs.aws.amazon.com/organizations/latest/userguide/orgs_reference_limits.html)
4. [Consolidating billing for AWS Organizations](https://docs.aws.amazon.com/awsaccountbilling/latest/aboutv2/consolidated-billing.html)
5. [Close an AWS account](https://docs.aws.amazon.com/accounts/latest/reference/manage-acct-closing.html) and [Closing a member account](https://docs.aws.amazon.com/organizations/latest/userguide/orgs_manage_accounts_close.html)
6. [Migrate an account to another organization](https://docs.aws.amazon.com/organizations/latest/userguide/orgs_account_migration.html)
7. [Choosing a plan](https://docs.aws.amazon.com/awsaccountbilling/latest/aboutv2/free-tier-plans.html), AWS Free Tier plans
8. [Create an AWS account](https://docs.aws.amazon.com/accounts/latest/reference/manage-acct-creating.html)
9. [Activating user-defined cost allocation tags](https://docs.aws.amazon.com/awsaccountbilling/latest/aboutv2/activating-tags.html)
10. [IAM principal attribution](https://docs.aws.amazon.com/bedrock/latest/userguide/cost-mgmt-iam-principal-tracking.html) and [Application inference profiles](https://docs.aws.amazon.com/bedrock/latest/userguide/cost-mgmt-application-inference-profiles.html), Amazon Bedrock user guide
11. [IAM Identity Center FAQs](https://aws.amazon.com/iam/identity-center/faqs/)
12. [Control access to secrets using attribute-based access control](https://docs.aws.amazon.com/secretsmanager/latest/userguide/auth-and-access-abac.html)

Tools and licenses:

13. [Terraform LICENSE](https://github.com/hashicorp/terraform/blob/main/LICENSE), Business Source License 1.1, licensor IBM, release v1.17.2 of 2026-11-11
14. [OpenTofu LICENSE](https://github.com/opentofu/opentofu/blob/main/LICENSE), MPL-2.0, release v1.13.9 of 2026-11-05
15. [terraform-provider-aws LICENSE](https://github.com/hashicorp/terraform-provider-aws/blob/main/LICENSE) and its resource docs for [`aws_secretsmanager_secret`](https://github.com/hashicorp/terraform-provider-aws/blob/main/website/docs/r/secretsmanager_secret.html.markdown), [`aws_s3_bucket`](https://github.com/hashicorp/terraform-provider-aws/blob/main/website/docs/r/s3_bucket.html.markdown) and [`aws_bedrockagent_knowledge_base`](https://github.com/hashicorp/terraform-provider-aws/blob/main/website/docs/r/bedrockagent_knowledge_base.html.markdown)
16. [OpenTofu S3 backend](https://github.com/opentofu/opentofu/blob/main/website/docs/language/settings/backends/s3.mdx)
17. [AWS CDK LICENSE](https://github.com/aws/aws-cdk/blob/main/LICENSE), Apache-2.0, release v2.277.0 of 2026-11-13, and [AWS CDK bootstrapping](https://docs.aws.amazon.com/cdk/v2/guide/bootstrapping.html)
18. CloudFormation: [pricing](https://aws.amazon.com/cloudformation/pricing/), [Delete a stack](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/cfn-console-delete-stack.html), [`AWS::S3::Bucket`](https://docs.aws.amazon.com/AWSCloudFormation/latest/TemplateReference/aws-resource-s3-bucket.html), [`AWS::SecretsManager::Secret`](https://docs.aws.amazon.com/AWSCloudFormation/latest/TemplateReference/aws-resource-secretsmanager-secret.html) and [`AWS::Bedrock::KnowledgeBase` configuration](https://docs.aws.amazon.com/AWSCloudFormation/latest/TemplateReference/aws-properties-bedrock-knowledgebase-knowledgebaseconfiguration.html)
19. [AWS CLI LICENSE](https://github.com/aws/aws-cli/blob/v2/LICENSE.txt), Apache-2.0, and [session-manager-plugin](https://github.com/aws/session-manager-plugin), Apache-2.0

Compute, storage and prices:

20. [AWS Price List, AmazonEC2, eu-central-1](https://pricing.us-east-1.amazonaws.com/offers/v1.0/aws/AmazonEC2/current/eu-central-1/index.csv), published 2026-11-19, read 2026-11-20. Usage types `EUC1-BoxUsage:t4g.large`, `t4g.medium`, `t3.large`, `EUC1-EBS:VolumeUsage.gp3`
21. [AWS Price List, AmazonECS, eu-central-1](https://pricing.us-east-1.amazonaws.com/offers/v1.0/aws/AmazonECS/current/eu-central-1/index.csv), published 2026-11-11, read 2026-11-20. Fargate vCPU-hours and GB-hours, Arm and x86
22. AWS Price List for eu-central-1, read 2026-11-20: [AmazonVPC](https://pricing.us-east-1.amazonaws.com/offers/v1.0/aws/AmazonVPC/current/eu-central-1/index.csv) (public IPv4 address, interface endpoint), [AWSSecretsManager](https://pricing.us-east-1.amazonaws.com/offers/v1.0/aws/AWSSecretsManager/current/eu-central-1/index.csv), [AmazonS3](https://pricing.us-east-1.amazonaws.com/offers/v1.0/aws/AmazonS3/current/eu-central-1/index.csv), [AmazonEFS](https://pricing.us-east-1.amazonaws.com/offers/v1.0/aws/AmazonEFS/current/eu-central-1/index.csv), [AWSELB](https://pricing.us-east-1.amazonaws.com/offers/v1.0/aws/AWSELB/current/eu-central-1/index.csv), [AmazonRDS](https://pricing.us-east-1.amazonaws.com/offers/v1.0/aws/AmazonRDS/current/eu-central-1/index.csv), [AmazonCognito](https://pricing.us-east-1.amazonaws.com/offers/v1.0/aws/AmazonCognito/current/eu-central-1/index.csv), [AmazonCloudWatch](https://pricing.us-east-1.amazonaws.com/offers/v1.0/aws/AmazonCloudWatch/current/eu-central-1/index.csv), and the global [AWSBudgets](https://pricing.us-east-1.amazonaws.com/offers/v1.0/aws/AWSBudgets/current/index.csv) file
23. [How EC2 instance stop and start works](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/how-ec2-instance-stop-start-works.html) and [Stop and start Amazon EC2 instances](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/Stop_Start.html)
24. [Storage options for Amazon ECS tasks](https://docs.aws.amazon.com/AmazonECS/latest/developerguide/using_data_volumes.html)
25. [AWS App Runner availability change](https://docs.aws.amazon.com/apprunner/latest/dg/apprunner-availability-change.html)
26. [Amazon ECS Express Mode](https://docs.aws.amazon.com/AmazonECS/latest/developerguide/express-service-overview.html)
27. [Keiko README](https://github.com/oscharko-dev/Keiko/blob/dev/README.md), branch `dev`, read 2026-11-20
28. LightRAG [`env.example`](https://github.com/HKUDS/LightRAG/blob/main/env.example) and [`docker-compose.yml`](https://github.com/HKUDS/LightRAG/blob/main/docker-compose.yml), read 2026-11-20
29. [Open WebUI quick start](https://github.com/open-webui/docs/blob/main/docs/getting-started/quick-start/index.mdx), read 2026-11-20
30. LiteLLM: [budgets and rate limits](https://docs.litellm.ai/docs/proxy/users), [production guide](https://docs.litellm.ai/docs/proxy/prod), [Bedrock provider](https://docs.litellm.ai/docs/providers/bedrock) and [LICENSE](https://github.com/BerriAI/litellm/blob/main/LICENSE), read 2026-11-20

Reaching the app:

31. Session Manager: [overview](https://docs.aws.amazon.com/systems-manager/latest/userguide/session-manager.html), [Start a session](https://docs.aws.amazon.com/systems-manager/latest/userguide/session-manager-working-with-sessions-start.html), [instance permissions](https://docs.aws.amazon.com/systems-manager/latest/userguide/setup-instance-permissions.html), [AMIs with the SSM Agent preinstalled](https://docs.aws.amazon.com/systems-manager/latest/userguide/ami-preinstalled-agent.html) and [Systems Manager pricing](https://aws.amazon.com/systems-manager/pricing/)
32. [Authenticate users using an Application Load Balancer](https://docs.aws.amazon.com/elasticloadbalancing/latest/application/listener-authenticate-users.html) and [Application Load Balancers](https://docs.aws.amazon.com/elasticloadbalancing/latest/application/application-load-balancers.html)
33. [Download and upload objects with presigned URLs](https://docs.aws.amazon.com/AmazonS3/latest/userguide/using-presigned-url.html)
34. [Monitor Amazon ECS containers with ECS Exec](https://docs.aws.amazon.com/AmazonECS/latest/developerguide/ecs-exec.html)

Amazon Bedrock:

35. [Request access to models](https://docs.aws.amazon.com/bedrock/latest/userguide/model-access.html)
36. [Inference endpoints supported by Amazon Bedrock](https://docs.aws.amazon.com/bedrock/latest/userguide/endpoints.html)
37. [Cross-Region inference](https://docs.aws.amazon.com/bedrock/latest/userguide/cross-region-inference.html), [Geographic cross-Region inference](https://docs.aws.amazon.com/bedrock/latest/userguide/geographic-cross-region-inference.html) and [Prerequisites for inference profiles](https://docs.aws.amazon.com/bedrock/latest/userguide/inference-profiles-prereq.html)
38. [Quotas for Amazon Bedrock](https://docs.aws.amazon.com/bedrock/latest/userguide/quotas.html), [Quotas for the bedrock-runtime endpoint](https://docs.aws.amazon.com/bedrock/latest/userguide/quotas-runtime.html), [Quotas for the bedrock-mantle endpoint](https://docs.aws.amazon.com/bedrock/latest/userguide/quotas-mantle.html) and [Amazon Bedrock service quotas](https://docs.aws.amazon.com/general/latest/gr/bedrock.html), read 2026-11-20
39. Model cards for [Claude Haiku 5.5](https://docs.aws.amazon.com/bedrock/latest/userguide/model-card-anthropic-claude-haiku-5-5.html), [Cohere Embed v4](https://docs.aws.amazon.com/bedrock/latest/userguide/model-card-cohere-embed-v4.html) and [Gemma 4 31B](https://docs.aws.amazon.com/bedrock/latest/userguide/model-card-google-gemma-4-31b.html), read 2026-11-20
40. [API keys](https://docs.aws.amazon.com/bedrock/latest/userguide/api-keys.html), Amazon Bedrock user guide
41. Managed knowledge bases: [prerequisites](https://docs.aws.amazon.com/bedrock/latest/userguide/kb-managed-prereqs.html), [create](https://docs.aws.amazon.com/bedrock/latest/userguide/kb-managed-create.html) and [service quotas](https://docs.aws.amazon.com/bedrock/latest/userguide/kb-managed-quotas.html)
42. [Amazon EC2 instance type quotas](https://docs.aws.amazon.com/ec2/latest/instancetypes/ec2-instance-quotas.html)

Cost limit and teardown:

43. AWS Budgets: [best practices](https://docs.aws.amazon.com/cost-management/latest/userguide/budgets-best-practices.html), [Configuring budget actions](https://docs.aws.amazon.com/cost-management/latest/userguide/budgets-controls.html), [Configuring a budget action](https://docs.aws.amazon.com/cost-management/latest/userguide/budgets-action-configure.html), [Creating a cost budget](https://docs.aws.amazon.com/cost-management/latest/userguide/create-cost-budget.html) and [pricing](https://aws.amazon.com/aws-cost-management/aws-budgets/pricing/)
44. [Delete an AWS Secrets Manager secret](https://docs.aws.amazon.com/secretsmanager/latest/userguide/manage_delete-secret.html)
45. [GetResources](https://docs.aws.amazon.com/resourcegroupstagging/latest/APIReference/API_GetResources.html), Resource Groups Tagging API reference

Project research:

- P1: [Microsoft, AWS and OVHcloud for a project memory](https://github.com/Niko4417/OPT-cc/blob/research/cloud-provider/docs/research/cloud-provider.md), 2026-11-19
- P2: [The most cost-efficient models per task on AWS](https://github.com/Niko4417/OPT-cc/blob/research/aws-models/docs/research/aws-models.md), 2026-11-19
- P3: [Stack scoring research](https://github.com/Niko4417/OPT-cc/blob/research/stack-scoring/docs/research/stack-scoring.md)
