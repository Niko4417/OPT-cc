# Microsoft, AWS and OVHcloud for a project memory

Date: 2026-11-19. Ticket: [#36](https://github.com/Niko4417/OPT-cc/issues/36). Map: [#1](https://github.com/Niko4417/OPT-cc/issues/1). Context: [ADR-0008](https://github.com/Niko4417/OPT-cc/blob/main/docs/adr/0008-leave-the-installation-location-open-until-the-provider-review-and-bake-off.md), [ADR-0009](https://github.com/Niko4417/OPT-cc/blob/main/docs/adr/0009-choose-one-cloud-provider-by-research-and-measure-its-managed-retrieval.md), [ADR-0010](https://github.com/Niko4417/OPT-cc/blob/main/docs/adr/0010-count-a-consultancy-tenant-as-third-party-cloud.md), [ADR-0011](https://github.com/Niko4417/OPT-cc/blob/main/docs/adr/0011-one-app-to-ask-and-add-knowledge-and-notes-are-project-artifacts.md), [model server research](https://github.com/Niko4417/OPT-cc/blob/research/model-server/docs/research/model-server.md).

## Question

Which one cloud provider fits a project memory best: Microsoft (Microsoft Foundry, Foundry IQ, Azure AI Search), AWS (Amazon Bedrock, Bedrock Knowledge Bases, AgentCore, Amazon Q Business or its successor) or OVHcloud (AI Endpoints, AI Deploy, Managed Kubernetes, managed PostgreSQL with pgvector, OpenSearch)? Compare sources and connectors, managed and agentic retrieval, graph features, models, EU data residency and sovereignty, access control, cost per project memory with idle cost, evaluation and tracing, and lock-in. Verify four claims made to the owner. Two clouds are never used side by side (ADR-0009).

## Answer

**OVHcloud fits this project memory best, with Microsoft as the alternative if a client requires managed, Entra-based permission trimming.** OVHcloud is the only one of the three that serves the lab answer model Qwen3.6-35B-A3B pay-per-token in the EU, its embedding and speech models are open-weight and therefore reproducible on-device, its parent company is French, and a commercial deal is in place. The price is that OVHcloud offers no managed knowledge base, no graph retrieval, no reranker model, no document-level permission trimming and no evaluation tooling. The app builds all of that on PostgreSQL with pgvector and Managed Kubernetes. Under ADR-0009 the bake-off then has no managed retrieval arm, because a managed layer may only come from the chosen provider.

The managed platforms do less for this use case than the earlier claims suggested. No managed knowledge base of the three providers has a GitHub connector, so the app ingests GitHub itself on every provider, as [the GitHub ingestion research](https://github.com/Niko4417/OPT-cc/blob/research/github-ingestion/docs/research/github-ingestion.md) already concluded. Both managed platforms treat source code as plain text.

Product names as of 2026-11-19: Azure AI Foundry is now **Microsoft Foundry**, and Microsoft renamed Azure AI Search to **Foundry IQ** ("Azure AI Search is now Foundry IQ", M1). Amazon Q Business is closed to new customers, and its successor is **Amazon Quick** (A9). AWS added **Amazon Bedrock Managed Knowledge Base** on 2026-06-17 next to the older customer-managed Bedrock Knowledge Bases (A3).

| Criterion | Microsoft | AWS | OVHcloud |
| --- | --- | --- | --- |
| Managed retrieval | Foundry IQ knowledge bases with agentic retrieval, GA in REST API `2026-10-01`, some features preview (M1, M3) | Bedrock Managed Knowledge Base with agentic retrieval, GA 2026-06-17 (A3, A4) | None. Self-hosted on PostgreSQL with pgvector or OpenSearch (O6) |
| GitHub connector | Not in Foundry IQ. Microsoft 365 Copilot has GitHub pull request connectors (M14) | Not in Bedrock. Q Business had one but is closed to new customers. Amazon Quick's GitHub integration is actions only (A9, A10) | None |
| Source code | Plain text, no syntax-aware chunking (M8, M9) | Customer-managed formats list has no code extensions (A6) | Whatever the app builds |
| Audio and video | Not in the Foundry IQ ingestion formats (M9) | Managed KB extracts .mp3, .wav, .m4a, .flac, .ogg, .mp4, .mov, .m4v (A5) | Whisper large-v3 on AI Endpoints (O1) |
| Graph | Knowledge graphs, public preview since 2026-11-17 (M2) | GraphRAG with Neptune Analytics, GA, Frankfurt, S3 sources only (A11) | None |
| Lab models in the EU, pay-per-token | Neither. Qwen3.6 and Gemma 4 only as Fireworks provisioned throughput, excluded from the EU Data Boundary (M10, M11) | Gemma 4 31B in Frankfurt. No Qwen3.6 (A1) | Qwen3.6-35B-A3B. Gemma 4 26B-A4B, not 31B (O1) |
| Reranker | Semantic ranker, Cohere rerank v4 (M7, M12) | Amazon Rerank, Cohere Rerank 3.5 in Frankfurt (A1) | None in the catalog (O1) |
| Region in Germany | Germany West Central (Frankfurt), but new Dedicated search services are blocked there and Serverless is not offered (M5) | eu-central-1 (Frankfurt) for Bedrock, Managed KB, AgentCore (A1, A7, A12) | Frankfurt for instances, databases and object storage. AI Endpoints runs in France (O1, O7) |
| US CLOUD Act | Exposed, US parent (S1) | Exposed, US parent | French parent, no US parent |
| Per-user result trimming | Entra token at query time, GA for SharePoint, ADLS Gen2 and Blob. Purview labels in preview (M6) | ACL-aware filtering by email, which AWS says "is not authorization" (A8) | The app builds it |
| Small project memory, per month | $109 (Basic) or $38 (Serverless preview) | $49 | €94 |
| Idle cost, small | $97 (Basic) or $24 (Serverless) | $44 | €92 (managed PostgreSQL) or €37 (pgvector on the cluster node) |
| Mid-size project memory, per month | $221 (Basic) or $177 (Serverless) | $121 | €207 |
| Lock-in | Medium: index content exportable, retrieval API proprietary, also served over MCP (M1, M4) | High for Managed KB: unnamed managed embedding model, no direct datastore access (A4) | Low: open-source stores, open-weight models |
| Commercial deal | None | None | Commercial deal in place |

The cost figures rest on the assumptions in [Cost per project memory](#cost-per-project-memory). Each provider uses a different answer model in them, so they compare price, not answer quality.

**AWS** has the lowest idle cost on a managed retrieval layer and serves the lab checker model Gemma 4 31B in Frankfurt. It loses on the missing Qwen3.6, on permission filtering that trusts an email address passed by the app, and on Managed KB lock-in. **Microsoft** has the most complete permission model and the most mature agentic retrieval. It loses on models: neither lab model runs pay-per-token with EU processing. Germany West Central also turns away new Dedicated search services today, and the scale-to-zero Serverless tier is a preview outside Germany.

## Sources and connectors

The project memory needs GitHub repository files, issues, pull requests and review comments, uploaded project artifacts including audio and video, and notes typed into the app (ADR-0006, ADR-0011).

**Microsoft.** Foundry IQ knowledge sources are a search index, Azure Blob, Azure SQL, file upload, OneLake, indexed SharePoint, remote SharePoint (preview), Fabric Data Agent (preview), Fabric Ontology (preview), MCP server (preview), Work IQ (preview) and Bing web (M3). There is no GitHub knowledge source. An MCP server knowledge source could call a GitHub MCP server at query time, but remote sources "don't return citation URLs" and nothing is indexed (M3). Microsoft 365 Copilot has a GitHub Cloud Pull Requests connector that indexes pull request metadata into Microsoft Graph and honors repository permissions (M14). It serves Microsoft 365 Copilot. I found no documented path from it into a Foundry IQ knowledge base. The file knowledge source accepts 200 files per knowledge source on the GA API `2026-10-01` and 10,000 on `2026-11-01-preview` (M9). Its standard extraction goes through the Content Understanding skill, which reads PDF, image, Office, HTML, TXT, MD, RTF and EML files and no audio or video (M9).

**AWS.** The customer-managed Bedrock Knowledge Bases accept S3, Confluence, SharePoint, Salesforce, a web crawler and a custom source. From 2026-09-30, AWS no longer lets customers create new Confluence, SharePoint, Salesforce or web crawler connectors on customer-managed knowledge bases and points to the Managed Knowledge Base (A6). The Managed Knowledge Base lists Box, S3, Confluence, Confluence Data Center, Custom, Dropbox, Google Drive, OneDrive, Salesforce, ServiceNow, SharePoint, Web Crawler and Zendesk (A5). There is no GitHub connector. Managed KB extracts images, audio (.mp3, .wav, .m4a, .flac, .ogg) and video (.mp4, .mov, .m4v), and syncs on demand or on a daily, weekly or monthly schedule (A5). Amazon Q Business had GitHub Cloud and GitHub Server connectors, but "Amazon Q Business is no longer open to new customers" (A9). Amazon Quick's GitHub integration works over MCP for actions, and its compatibility table says "Knowledge Base: No" (A10).

**OVHcloud.** OVHcloud has no connector product. Its RAG reference architecture combines AI Endpoints, Managed PostgreSQL with pgvector and Object Storage, assembled by the customer (O6).

On all three providers the app runs its own GitHub ingestion. On Microsoft and AWS it can then push pre-chunked records into the managed index: a Foundry IQ search index wrapped as a knowledge source, or the Bedrock custom connector.

## Managed retrieval, agentic retrieval and graphs

**Microsoft.** A Foundry IQ knowledge base references knowledge sources and runs agentic retrieval: an LLM plans subqueries, runs them in parallel, reranks semantically and returns extractive results with citations (M1, M3). The retrieval reasoning effort is minimal, low or medium. Minimal skips model-based planning (M3). Indexed sources get a generated indexer pipeline that chunks, embeds and refreshes on a schedule (M1, M3). Knowledge bases and some knowledge sources are GA. The portal experience for agentic retrieval is still preview (M1). On 2026-11-17 Microsoft announced knowledge graphs (entities mapped with an ontology, GQL queries) and automatic topics in public preview, plus a retrieval model, Microsoft Retriever 1, also in preview (M2). The `2026-11-01-preview` API adds a topic graph for search index knowledge sources (M3).

**AWS.** Managed Knowledge Base manages the datastore, parsing, chunking, a "built-in managed model" for embeddings and a managed reranker "at no extra cost", and supports agentic retrieval. The customer-managed variant does not (A4). GraphRAG in customer-managed knowledge bases uses Amazon Neptune Analytics. It is GA, available in Frankfurt, and accepts only S3 as a data source (A11). The smallest Neptune Analytics graph in Frankfurt, 16 m-NCU, costs $0.581 an hour, about $424 a month whether queried or not (A13).

**OVHcloud.** OVHcloud has no managed knowledge base, no agentic retrieval and no graph product. I searched the AI Endpoints catalog, the docs and the newsroom and found only reference architectures (O1, O6). The app owns retrieval: pgvector or OpenSearch for vectors and keywords, its own query planning through LiteLLM, and any graph in PostgreSQL or a self-hosted graph database. Sibling tickets #33 and #35 cover how.

## Models offered

The lab models are Qwen3.6-35B-A3B (answer) and Gemma 4 31B (checker) (model server research).

| | Microsoft Foundry | Amazon Bedrock (Frankfurt) | OVHcloud AI Endpoints |
| --- | --- | --- | --- |
| Qwen3.6-35B-A3B | Fireworks only, provisioned throughput, minimum 90 PTU, Global or US data zone (M11). The "Direct from Azure" entry is fine-tuning only (M13) | Not offered. Qwen3 32B, Qwen3 235B, Qwen3-Coder-30B in-region (A1) | Yes, €0.17 / €1.15 per million input / output tokens, 256k context (O1) |
| Gemma 4 31B | Fireworks provisioned only, minimum 200 PTU (M11). Hugging Face collection on managed compute VMs (M17, M20) | Yes, in-region on the `bedrock-mantle` endpoint, $0.17 / $0.48 (A1, A2) | Gemma 4 26B-A4B only, €0.09 / €0.35 (O1) |
| Other open-weight | DeepSeek V4 Flash, Mistral Large 3, Mistral Medium 3.5 in the EU data zone (M12) | gpt-oss, MiniMax, Mistral Devstral 2, Nemotron, GLM in-region (A1) | gpt-oss-20b and 120b, Qwen3.5, Qwen3.8-27B, Mistral Small 3.2, DeepSeek V4.1 Flash, Nemotron (O1) |
| Frontier | GPT-5.x and GPT-6.x in the EU data zone (M12). Claude is sold by Anthropic as a Marketplace offer (M15) | Claude Sonnet and Opus 5.x via EU geo inference. GPT-6 and Claude Fable/Mythos only global (A1) | None |
| Embeddings | text-embedding-3-large, EU data zone, $0.143 per million tokens (M12, M16) | Titan Text Embeddings V2 in-region, Cohere Embed v4 via EU geo (A1). Managed KB has its own model (A4) | Qwen3-Embedding-8B (€0.10), bge-m3 (€0.01) (O1) |
| Reranker | Semantic ranker, Cohere rerank v4 fast and pro (M7, M12) | Amazon Rerank, Cohere Rerank 3.5 in-region (A1) | None |
| Speech to text | Azure services outside Foundry IQ, not checked | Managed KB audio extraction (A5) | Whisper large-v3 and v3-turbo (O1) |

The Fireworks route is "currently excluded from EU Data Boundary commitments", and its data zone deployments support "only the US data zone" (M10, M11). Running a lab model on Microsoft with EU processing therefore means a managed compute GPU VM billed per hour (M17), which outweighs every other cost line in this report.

**LiteLLM compatibility.** LiteLLM has an `ovhcloud/` provider that covers the AI Endpoints catalog, including embeddings (O4). Bedrock models use the `bedrock/` provider, and the `bedrock-mantle` endpoint takes OpenAI-format requests, so `openai/` with that `api_base` also works (A2). Azure OpenAI deployments use `azure/` and Foundry models use `azure_ai/`. LiteLLM's rerank table lists Bedrock and Azure AI but no OVHcloud entry (model server research, S2). LiteLLM can also query a Bedrock Knowledge Base or an Azure AI Search index as a vector store during chat completions (L1). Every route declares `model_info.model_location` as the model server research proposes. A route in the consultancy's tenant is `third-party-cloud` and a route in the client's own tenant is `client-hosted` (ADR-0010):

```yaml
model_list:
  - model_name: answer
    litellm_params:
      model: ovhcloud/qwen3.6-35b-a3b      # catalog name on 2026-11-19
      api_key: os.environ/OVHCLOUD_API_KEY
    model_info:
      model_location: third-party-cloud    # client-hosted if the client owns the tenant
  - model_name: embed
    litellm_params:
      model: ovhcloud/Qwen3-Embedding-8B
      api_key: os.environ/OVHCLOUD_API_KEY
    model_info:
      model_location: third-party-cloud
      mode: embedding
```

## EU data residency and sovereignty

**Regions.**

- Microsoft: Germany West Central (Frankfurt) supports agentic retrieval and semantic ranker, but it is "in high demand, which prevents the creation of new Dedicated Foundry IQ services". Germany North lists no AI features. Serverless runs in Denmark East, Poland Central, Sweden Central, Switzerland North and UK South, not in Germany (M5). Foundry models with EU data zone processing can be deployed from Germany West Central (M12). Data zone deployments process prompts "within any EU member nation" (M12).
- AWS: Bedrock, Managed KB, GraphRAG and AgentCore run in eu-central-1 (Frankfurt) (A1, A7, A11, A12). EU geo cross-region inference keeps prompts "within geographic boundaries (e.g., all EU Regions)" (A1). The AWS European Sovereign Cloud region `eusc-de-east-1` (Germany) lists only Amazon Nova Pro and Nova Lite in the Bedrock model table on 2026-11-19 (A14), and the Managed KB region list does not include it (A7). An AWS Security Blog post is titled "Run open weight models on Amazon Bedrock in AWS European Sovereign Cloud", but I could not read its body and the model table does not confirm it (A15).
- OVHcloud: AI Endpoints models report datacenter country FR (O1), and the launch release names Gravelines (O3). AI Deploy runs in GRA (France) with L4, L40S, A100, H100 and H200 GPUs, and in BHS (Canada) with V100S GPUs only (O5). The regions page appears to list Frankfurt for instances, managed databases and object storage, but its table did not parse cleanly, and I could not confirm Managed Kubernetes in Frankfurt from it (O7).

**US CLOUD Act.** Microsoft and Amazon are US companies. On 2026-06-10 a Microsoft France director told a French Senate inquiry, under oath, that he could not guarantee that data of French public bodies would never be handed to US authorities (S1). The EU Data Boundary and EU data zones govern where data is processed, not which law applies to the provider. OVHcloud's parent is French. I found no OVHcloud statement on the CLOUD Act itself.

**Qualifications.** OVHcloud's SNC Cloud Platform received the ANSSI SecNumCloud qualification on 2026-09-01, starting in France, with Germany, Italy and Poland next (O2). The release does not say that AI Endpoints is part of it. OVHcloud states that it meets the BSI C5 catalogue (O8). AWS lists Amazon Bedrock in its C5 scope (page updated 2026-01-13), excluding Bedrock Marketplace. AgentCore is only internally assessed so far (A16). Azure keeps a combined C5, SOC 2 Type 2 and CSA STAR report (M21). The service-level scope is in that report on the Service Trust Portal, which I did not open. Microsoft's sovereign offers are Sovereign Public Cloud, Sovereign Private Cloud (Azure Local, Microsoft 365 Local, Foundry Local) and partner clouds such as Delos Cloud in Germany (M18). I found no source that places Foundry IQ in Delos Cloud or Sovereign Private Cloud.

**What German clients require.** Federal agencies must use cloud services with a C5 attestation under the BSI minimum standard for external cloud services (S2). I found no survey of what private-sector modernization clients require, so this report does not claim a usual requirement. ADR-0010 matters here: a project memory in a client's own Azure or AWS tenant counts as client-hosted, which is easier to accept for clients that already run on those clouds.

## Access control and sharing

**Microsoft.** The app calls Foundry IQ with its own identity in `Authorization` and passes the user's Entra token in `x-ms-query-source-authorization`. Foundry IQ then trims results to documents the user may read (M6). This is GA in REST API `2026-10-01` for SharePoint ACLs, ADLS Gen2 ACLs and RBAC scopes, Blob container RBAC and pushed ACLs. Purview sensitivity label enforcement "remains in preview", is "limited to single-tenant scenarios", and works only for labels on Blob, ADLS Gen2, SharePoint and OneLake content (M6). For pushed GitHub chunks the app would write Entra user or group IDs into the permission fields itself. Security filters on plain strings are GA for other identity systems (M6).

**AWS.** Managed KB crawls ACLs from its connectors and filters by a user context that the app passes at query time. Users are matched by email, and AWS states "ACL awareness is not authorization" because it cannot verify the user context (A8). The custom connector takes ACLs from metadata the app supplies (A8). Service access uses IAM.

**OVHcloud.** No document-level trimming exists. The app enforces permissions in its own queries, for example with PostgreSQL row-level security, and authenticates users against the client's or consultancy's identity provider.

## Cost per project memory

**Assumptions.**

| | Small | Mid-size |
| --- | --- | --- |
| Sources | 1 repository, 1,500 issues and pull requests, 300 documents, 5 hours of recordings | 5 repositories, 10,000 issues and pull requests, 3,000 documents, 40 hours of recordings |
| Indexed text | 30 million tokens, 60,000 chunks | 200 million tokens, 400,000 chunks |
| Raw source size | 0.5 GB | 5 GB |
| Index size (1,024-dimension float vectors plus text) | about 0.4 GB | about 2.5 GB |
| Users and questions | 5 consultants, 500 questions a month | 20 consultants, 5,000 questions a month |

Each question sends 20,000 input and 1,000 output tokens to the answer model and makes one agentic retrieval with three subqueries. Ten percent of the text changes each month and is embedded again. One always-on container with 1 vCPU and 2 GB runs the app and LiteLLM. Prices are list prices without VAT, in the currency each provider publishes, read on 2026-11-19. The OVHcloud figures do not reflect the commercial deal.

**Microsoft** (Poland Central or France Central; Germany West Central accepts no new Dedicated services):

| Line | Small | Mid-size |
| --- | --- | --- |
| Foundry IQ Dedicated Basic, $0.101 per hour (M7) | $73.73 | $73.73 |
| or Foundry IQ Serverless (preview), storage $0.259 per GB-month plus compute $0.2895 per CU-hour, assuming 10 and 100 CU-hours (M7) | $3.00 | $29.60 |
| Agentic retrieval tokens, 50 million free a month (M7) | $0 | $0 |
| Semantic ranker, 1,000 free, then $1 per 1,000 (M7) | $0.50 | $14.00 |
| Re-embedding with text-embedding-3-large, EU data zone (M16) | $0.43 | $2.86 |
| Answer model gpt-5.4-mini, EU data zone, $0.825 / $4.95 (M16) | $10.73 | $107.25 |
| Container Apps, one warm idle replica (M7) | $23.65 | $23.65 |
| **Total with Basic** | **$109** | **$221** |
| **Total with Serverless** | **$38** | **$177** |
| Idle cost with Basic, with Serverless | $97, $24 | $97, $24 |

Microsoft publishes no compute-units-per-query figure for Serverless, so the CU-hour lines are guesses that the bake-off must measure. Serverless storage alone is $0.10 and $0.65 a month, and Container Apps can scale to zero, so the true Serverless floor is under $1. The pricing page still labels Serverless "preview", Microsoft has billed it since 2026-09-13, and the 2026-11-17 announcement plans GA for December 2026 (M2, M4, M7). Basic is a fixed cost "around the clock", and each added replica adds the same hourly rate (M4).

**AWS** (Frankfurt, Managed Knowledge Base):

| Line | Small | Mid-size |
| --- | --- | --- |
| Managed KB storage, $5 per GB of raw data per month (A2) | $2.50 | $25.00 |
| Agentic retrieval, $4 per 1,000 calls plus $1 per 1,000 underlying Retrieve calls (A2) | $3.50 | $35.00 |
| Parsing, embedding, reranking (A2) | $0 | $0 |
| Answer model Gemma 4 31B, $0.17 / $0.48 (A2) | $1.94 | $19.40 |
| Fargate, 1 vCPU and 2 GB (A17) | $41.45 | $41.45 |
| **Total** | **$49** | **$121** |
| Idle cost | $44 | $66 |

Storage is charged on raw data, so uploading 40 hours of video at about 1 GB per hour would add about $200 a month. The app should upload audio tracks or transcripts instead. GraphRAG would add a Neptune Analytics graph at about $424 a month and requires a customer-managed knowledge base (A11, A13).

**OVHcloud** (Frankfurt for compute and database, France for AI Endpoints):

| Line | Small | Mid-size |
| --- | --- | --- |
| Managed PostgreSQL Essential, DB1-4 or DB1-7, one node (O9) | €54.46 | €107.82 |
| Managed Kubernetes Free control plane (O9) | €0 | €0 |
| One node for app, LiteLLM and a CPU reranker, b3-8 or b3-16 (O9) | €37.38 | €74.68 |
| Object Storage, €0.007 per GB-month (O9) | €0.00 | €0.04 |
| Re-embedding with Qwen3-Embedding-8B, €0.10 per million (O1) | €0.30 | €2.00 |
| Answer model Qwen3.6-35B-A3B, €0.17 / €1.15 (O1) | €2.28 | €22.75 |
| **Total** | **€94** | **€207** |
| Idle cost | €92 | €183 |

One-off costs on top: the first embedding run costs €3 and €20, and Whisper large-v3 transcription of 5 and 40 hours costs €0.74 and €5.90 (O1). On Microsoft the first embedding run costs $4.29 and $28.60. On AWS it is included. Running pgvector inside the Kubernetes node instead of the managed database lowers the small OVHcloud idle cost to €37, without managed backups or failover.

The answer model drives most of the usage cost. With the same model on every provider the gap would shrink, but the lab model is only available pay-per-token on OVHcloud.

## Evaluation and tracing

**Microsoft.** Microsoft announced evaluations, monitoring and tracing in Microsoft Foundry as GA on 2026-03-16, with tracing following by the end of March. Its RAG evaluators are groundedness and retrieval quality. Traces use OpenTelemetry and land in Application Insights. Custom evaluators are still preview (M19). Foundry IQ returns references and activity logs from the retrieve action (M3).

**AWS.** Bedrock RAG evaluation is GA since 2025-03-20 and can evaluate a Bedrock knowledge base or "bring your own inference responses" from an external RAG system, with citation precision and coverage metrics (A18). AgentCore Observability and Evaluations run in Frankfurt (A12). Managed KB has its own observability page (A4).

**OVHcloud.** AI Endpoints offers model metrics with call counts and token usage (O10). Evaluation and tracing are self-hosted, for example on the Kubernetes cluster.

The bake-off's own gold question set and evaluation harness work on all three providers, so the managed evaluators are a convenience, not a requirement.

## Lock-in and export

- Microsoft: index documents, including vectors stored as retrievable fields, can be read back and moved with Microsoft's backup and restore samples. Knowledge source definitions are JSON, but GET responses "can redact or omit secrets" (M3, M4). Agentic retrieval is a proprietary API, also served through the Foundry IQ MCP server (M1).
- AWS: Managed KB uses a service-managed embedding model and reranker. The comparison table offers "direct access" to the vector store only for customer-managed knowledge bases (A4). Vectors from the managed model cannot be reused elsewhere, so a move means embedding everything again. Customer-managed knowledge bases keep vectors in a store the customer owns.
- OVHcloud: PostgreSQL, pgvector, OpenSearch and Kubernetes are open source, and the models are open-weight. The same Qwen3-Embedding and Qwen3.6 weights run on-device, which keeps vectors in a project memory bundle valid when it moves between a cloud and a laptop. Sibling ticket #34 covers embedding lock-in in depth.

## Verdicts on earlier claims

1. "The managed platforms handle ingestion, chunking, embedding and index refresh." **Partly confirmed.** Foundry IQ and Bedrock Managed KB do this for their own connectors and for uploaded files (M1, M3, A4, A5). Neither has a GitHub knowledge connector, so the app ingests code, issues and pull requests itself. AWS Managed KB schedules syncs at most daily, and the app triggers anything faster on demand (A5). The Foundry IQ file upload holds 200 files per knowledge source on the GA API (M9).
2. "Purview sensitivity labels trim results per user." **Partly confirmed.** Foundry IQ enforces Purview labels at query time, but the feature is in preview, single-tenant only, and works only for labelled content in Blob, ADLS Gen2, SharePoint or OneLake (M6). GitHub content and notes typed into the app carry no Purview labels. ACL-based trimming for SharePoint, ADLS Gen2 and Blob is GA (M6).
3. "They are built for documents rather than code." **Confirmed.** The Foundry IQ blob indexer lists document formats and treats code as plain text. The Content Understanding skill accepts no code formats (M8, M9). The customer-managed Bedrock formats list has no code extensions (A6). Neither offers syntax-aware chunking.
4. "OVHcloud has no managed knowledge base or graph RAG." **Confirmed** as of 2026-11-19. The AI Endpoints catalog, the docs and the newsroom show none. OVHcloud's own RAG guidance assembles AI Endpoints, PostgreSQL with pgvector and Object Storage (O1, O6).

## Open points

- Measure Foundry IQ Serverless compute units per query, if Microsoft stays in the running.
- Check whether Bedrock Managed KB accepts pre-chunked records without its own chunking through the custom connector.
- Ask OVHcloud whether AI Endpoints is in scope for SecNumCloud or C5, and whether a reranker model is planned.
- Confirm Managed Kubernetes availability in OVHcloud's Frankfurt region.

## Method

I read official documentation, pricing APIs and catalogs, and announcements on 2026-11-19. Azure prices come from the Azure Retail Prices API. AWS prices come from the AWS Price List API and the Bedrock pricing page. OVHcloud prices come from the public order catalog for the German subsidiary (EUR, without VAT) and the AI Endpoints catalog API. I parsed the Bedrock regional availability page for every model available in eu-central-1 and `eusc-de-east-1`. I did not deploy anything, did not run queries and did not open audit reports behind logins. Where a primary page did not load, the text says so.

## Sources

Microsoft:

- M1: [What is Foundry IQ?](https://learn.microsoft.com/en-us/azure/foundry/agents/concepts/what-is-foundry-iq), ms.date 2026-11-04
- M2: [Foundry IQ: introducing knowledge graphs engineered for agents](https://techcommunity.microsoft.com/blog/azure-ai-foundry-blog/foundry-iq-introducing-knowledge-graphs-engineered-for-agents/4572683), 2026-11-17
- M3: [What is a knowledge source?](https://learn.microsoft.com/en-us/azure/search/agentic-knowledge-source-overview), ms.date 2026-11-09
- M4: [Choose a pricing model and service tier](https://learn.microsoft.com/en-us/azure/search/search-sku-tier), ms.date 2026-11-09
- M5: [Supported regions for Foundry IQ](https://learn.microsoft.com/en-us/azure/search/search-region-support), ms.date 2026-11-03
- M6: [Document-level access control](https://learn.microsoft.com/en-us/azure/search/search-document-level-access-overview), ms.date 2026-11-09
- M7: [Azure Retail Prices API](https://prices.azure.com/api/retail/prices), service names "Azure Cognitive Search" and "Azure Container Apps", regions germanywestcentral, francecentral, polandcentral, swedencentral, read 2026-11-19; [Foundry IQ pricing page](https://azure.microsoft.com/en-us/pricing/details/search/)
- M8: [Azure Blob indexer](https://learn.microsoft.com/en-us/azure/search/search-how-to-index-azure-blob-storage), ms.date 2026-11-09
- M9: [Create a file knowledge source](https://learn.microsoft.com/en-us/azure/search/agentic-knowledge-source-how-to-file), ms.date 2026-11-17, and [Azure Content Understanding skill](https://learn.microsoft.com/en-us/azure/search/cognitive-search-skill-content-understanding), ms.date 2026-11-05
- M10: [Fireworks models on Microsoft Foundry](https://learn.microsoft.com/en-us/azure/foundry/how-to/fireworks/enable-fireworks-models), ms.date 2026-10-19
- M11: [Fireworks model catalog, deployment types and regions](https://learn.microsoft.com/en-us/azure/foundry/how-to/fireworks/fireworks-model-catalog), ms.date 2026-11-10
- M12: [Region availability for Foundry Models sold by Azure](https://learn.microsoft.com/en-us/azure/foundry/foundry-models/concepts/models-sold-directly-by-azure-region-availability), ms.date 2026-09-03
- M13: [Qwen3.6-35B-A3B in the Foundry catalog](https://ai.azure.com/catalog/models/qwen3.6-35b-a3b), read 2026-11-19
- M14: [GitHub Cloud Pull Requests Microsoft 365 Copilot connector](https://learn.microsoft.com/en-us/microsoftsearch/github-cloud-pull-requests-overview)
- M15: [Data, privacy and security for Claude models in Microsoft Foundry](https://learn.microsoft.com/en-us/azure/foundry/responsible-ai/claude-models/data-privacy), ms.date 2026-06-23
- M16: Azure Retail Prices API, service name "Foundry Models", region germanywestcentral, meters "text embedding 3 large DZ" and "5.4 mini Inp Dz" / "5.4 mini Opt Dz", read 2026-11-19
- M17: [Microsoft Foundry Models overview](https://learn.microsoft.com/en-us/azure/foundry/concepts/foundry-models-overview), ms.date 2026-07-28
- M18: [Microsoft Sovereign Cloud adds governance, productivity and support for large AI models](https://blogs.microsoft.com/blog/2026/02/24/microsoft-sovereign-cloud-adds-governance-productivity-and-support-for-large-ai-models-securely-running-even-when-completely-disconnected/), 2026-02-24
- M19: [Generally available: evaluations, monitoring and tracing in Microsoft Foundry](https://techcommunity.microsoft.com/t5/microsoft-foundry-blog/generally-available-evaluations-monitoring-and-tracing-in/ba-p/4502760), 2026-03-16
- M20: [Gemma 4 now available in Microsoft Foundry](https://techcommunity.microsoft.com/blog/azure-ai-foundry-blog/gemma-4-now-available-in-microsoft-foundry/4510984), 2026-04-14
- M21: [Cloud Computing Compliance Criteria Catalog (C5)](https://learn.microsoft.com/en-us/compliance/regulatory/offering-c5-germany), updated 2026-09-01

AWS:

- A1: [Regional availability by models](https://docs.aws.amazon.com/bedrock/latest/userguide/models-region-compatibility.html), read 2026-11-19
- A2: [Amazon Bedrock pricing](https://aws.amazon.com/bedrock/pricing/), read 2026-11-19, and [Endpoints supported by Amazon Bedrock](https://docs.aws.amazon.com/bedrock/latest/userguide/endpoints.html)
- A3: [Amazon Bedrock Managed Knowledge Base is now generally available](https://aws.amazon.com/about-aws/whats-new/2026/06/amazon-bedrock-managed-knowledge-base/), 2026-06-17
- A4: [Build a managed knowledge base](https://docs.aws.amazon.com/bedrock/latest/userguide/kb-build-managed.html)
- A5: [Connect a data source (managed knowledge bases)](https://docs.aws.amazon.com/bedrock/latest/userguide/kb-managed-connect-ds.html)
- A6: [Connect a data source to your knowledge base](https://docs.aws.amazon.com/bedrock/latest/userguide/data-source-connectors.html) and [Prerequisites for your knowledge base data](https://docs.aws.amazon.com/bedrock/latest/userguide/knowledge-base-ds.html)
- A7: [Supported AWS Regions for managed knowledge bases](https://docs.aws.amazon.com/bedrock/latest/userguide/kb-managed-regions.html)
- A8: [Access Control Lists awareness enablement](https://docs.aws.amazon.com/bedrock/latest/userguide/kb-managed-acl.html)
- A9: [Amazon Q Business availability change](https://docs.aws.amazon.com/amazonq/latest/qbusiness-ug/qbusiness-availability-change.html)
- A10: [GitHub integration in Amazon Quick](https://docs.aws.amazon.com/quick/latest/userguide/github-integration.html)
- A11: [Build a knowledge base with graphs from Amazon Neptune Analytics](https://docs.aws.amazon.com/bedrock/latest/userguide/knowledge-base-build-graphs.html) and [GraphRAG generally available](https://aws.amazon.com/about-aws/whats-new/2025/03/amazon-bedrock-knowledge-bases-graphrag-generally-available), 2025-03
- A12: [AgentCore supported AWS Regions](https://docs.aws.amazon.com/bedrock-agentcore/latest/devguide/agentcore-regions.html)
- A13: [AWS Price List API, Amazon Neptune, eu-central-1](https://pricing.us-east-1.amazonaws.com/offers/v1.0/aws/AmazonNeptune/current/eu-central-1/index.json), usage type `EUC1-MemoryOptimizedGraphUsage:16-m-NCU`, read 2026-11-19
- A14: [Regional availability by models, AWS European Sovereign Cloud docs](https://docs.aws.eu/bedrock/latest/userguide/models-region-compatibility.html), read 2026-11-19
- A15: [Run open weight models on Amazon Bedrock in AWS European Sovereign Cloud](https://aws.amazon.com/blogs/security/run-open-weight-models-on-aws-bedrock-in-aws-european-sovereign-cloud/) (title only, body not readable)
- A16: [AWS services in scope for C5](https://aws.amazon.com/compliance/services-in-scope/C5/) and [Compliance validation for Amazon Bedrock AgentCore](https://docs.aws.amazon.com/bedrock-agentcore/latest/devguide/compliance-validation.html)
- A17: [AWS Price List API, Amazon ECS, eu-central-1](https://pricing.us-east-1.amazonaws.com/offers/v1.0/aws/AmazonECS/current/eu-central-1/index.json), Fargate vCPU-hours and GB-hours, read 2026-11-19
- A18: [Amazon Bedrock now supports RAG evaluation (generally available)](https://aws.amazon.com/about-aws/whats-new/2025/03/amazon-bedrock-rag-evaluation-generally-available), 2025-03-20, and [Evaluate RAG sources](https://docs.aws.amazon.com/bedrock/latest/userguide/evaluation-kb.html)

OVHcloud:

- O1: [AI Endpoints catalog API](https://catalog.endpoints.ai.ovh.net/rest/v1/models_v2), read 2026-11-19 (documented in [AI Endpoints catalog API](https://docs.ovhcloud.com/en/guides/public-cloud/ai-machine-learning/ai-endpoints-catalog-api.md))
- O2: [OVHcloud obtains SecNumCloud qualification for SNC Cloud Platform](https://corporate.ovhcloud.com/en/newsroom/news/ovhcloud-obtains-secnumcloud-qualification-snc-cloud-platform/), 2026-09-01
- O3: [OVHcloud launches AI Endpoints](https://corporate.ovhcloud.com/en/newsroom/news/ai-endpoints/), 2025-04-29
- O4: [LiteLLM OVHcloud AI Endpoints provider](https://docs.litellm.ai/docs/providers/ovhcloud) and [AI Endpoints LiteLLM integration](https://docs.ovhcloud.com/en/guides/public-cloud/ai-machine-learning/ai-endpoints-litellm-integration)
- O5: [AI Deploy features, capabilities and limitations](https://docs.ovhcloud.com/en/guides/public-cloud/ai-machine-learning/ai-deploy-capabilities) and [AI Deploy scaling strategies](https://docs.ovhcloud.com/en/guides/public-cloud/ai-machine-learning/ai-deploy-apps-deployments)
- O6: [Reference architecture: retrieval augmented generation](https://blog.ovhcloud.com/reference-architecture-retrieval-augmented-generation-rag/)
- O7: [Public Cloud regions availability](https://www.ovhcloud.com/en/public-cloud/regions-availability/), read 2026-11-19
- O8: [C5 at OVHcloud](https://www.ovhcloud.com/en/compliance/c5/)
- O9: [OVHcloud public order catalog, Public Cloud, subsidiary DE](https://eu.api.ovh.com/v1/order/catalog/public/cloud?ovhSubsidiary=DE), EUR without VAT, read 2026-11-19
- O10: [AI Endpoints capabilities](https://docs.ovhcloud.com/en/guides/public-cloud/ai-machine-learning/ai-endpoints-capabilities.md), lastUpdated 2026-11-10

LiteLLM:

- L1: [LiteLLM knowledge bases and vector stores](https://docs.litellm.ai/docs/completion/knowledgebase)

Sovereignty and compliance:

- S1: [French Senate, commission d'enquête on public procurement, hearing of Microsoft France, 2025-06-10](https://senat.fr/compte-rendu-commissions/20250609/ce_commande_publique.html)
- S2: [BSI minimum standard for the use of external cloud services](https://www.bsi.bund.de/EN/Themen/Oeffentliche-Verwaltung/Mindeststandards/Externe_Cloud-Dienste/Externe_Cloud-Dienste.html)
