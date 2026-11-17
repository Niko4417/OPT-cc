# Project memory

A local-first system that lets consultants ask natural-language questions about everything a modernization project has produced: code, documentation, workshop results and decision records.

## Language

**Modernization project**:
One client engagement that modernizes an existing software system.
_Avoid_: Client project, engagement, Mandant

**Project memory**:
The searchable knowledge of exactly one modernization project, answering questions with citations to its sources.
_Avoid_: Projektgedächtnis, RAG system, knowledge store

**Decision record**:
A dated record of one decision made in a modernization project, with its status and the decision that supersedes it, if any.
_Avoid_: ADR (when meaning meeting decisions too), decision log entry

**Workshop result**:
A text record of a workshop in a modernization project, such as minutes or a transcript.
_Avoid_: Workshop output, meeting notes

**Model location**:
Where a language, embedding or reranking model runs: on-device (the consultant's own machine), client-hosted (inside the client's network) or third-party cloud.
_Avoid_: Locality, provider, deployment target

**Model policy**:
The furthest model location one project memory may send its content to. The default is on-device only.
_Avoid_: Model-use policy, privacy mode
