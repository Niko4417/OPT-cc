# Project memory

A system that lets consultants ask natural-language questions about everything a modernization project has produced: code, documentation, workshop results and decision records.

## Language

**Modernization project**:
One client engagement that modernizes an existing software system.
_Avoid_: Client project, engagement, Mandant

**Project memory**:
The searchable knowledge of exactly one modernization project, answering questions with citations to its sources.
_Avoid_: Projektgedächtnis, RAG system, knowledge store

**Decision record**:
A dated record of one decision made in a modernization project, with its decision status and the decision record that supersedes it, if any. ADRs, RFCs with a status and decisions made in workshops are all decision records.
_Avoid_: ADR (when meaning meeting decisions too), decision log entry

**Decision status**:
Where a decision record stands: proposed, accepted, rejected, deprecated or superseded. A status the project memory can't map to one of these is unknown.
_Avoid_: State, phase

**Decision log**:
A client document that lists many decisions in one file, such as a Word table or an Excel sheet. Its entries become decision records only after a human confirms them.
_Avoid_: Decision register

**Decision register**:
The list of every decision record in one project memory, with its decision status, date, source, supersession chain and open decision conflicts.
_Avoid_: Decision log, ADR index

**Decision conflict**:
Two or more decision records whose relationship the project memory can't settle by itself: a newer record that contradicts an older one without saying it supersedes it, a supersession that runs against creation order, a cycle, or two records that supersede the same one. Only a human resolves it.
_Avoid_: Contradiction, inconsistency

**Ruling**:
A human's resolution of one decision conflict, kept in the project memory. It holds until one of the records it covers changes.
_Avoid_: Override, verdict

**Workshop result**:
A text record of a workshop in a modernization project, such as minutes or a transcript.
_Avoid_: Workshop output, meeting notes

**Project artifact**:
Anything a consultant adds to a project memory by hand, beyond the repositories it is connected to, such as a decision log, slides, a spreadsheet, an interview recording, a recorded workshop or a note typed into the app. The term says how a source arrived, not what it says, so a decision record or workshop result can be a project artifact or repository content.
_Avoid_: Artifact (alone), evidence, attachment, additional document

**Model location**:
Where a language, embedding or reranking model runs: on-device (the consultant's own machine), client-hosted (inside the client's network) or third-party cloud. A cloud tenant owned by the client counts as client-hosted, and one owned by the consultancy counts as third-party cloud.
_Avoid_: Locality, provider, deployment target

**Model policy**:
The furthest model location one project memory may send its content to.
_Avoid_: Model-use policy, privacy mode

**Project memory bundle**:
One project memory packed as a single signed and encrypted file that another consultant can import into their installation, holding its sources, its human-made text with history, chunks, vectors, model policy and provenance.
_Avoid_: Export, capsule, snapshot

**Installation**:
One running copy of the app with its own storage and its own key, used by one consultant.
_Avoid_: Instance, environment, deployment
