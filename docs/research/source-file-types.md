# Source file types the plans do not cover yet

Date: 2026-11-20. Map: [#1](https://github.com/Niko4417/OPT-cc/issues/1). These are research findings, not owner decisions.

## Question

Which file types turn up among the sources of a modernization project that the current plans for a project memory do not cover yet, and which of them are worth supporting and testing in V1? The owner decided that the test dataset must hold at least one item of every source type V1 ingests, and asked whether file types were forgotten. A second question is what the app should do with a file type it cannot read, so the consultant sees that it was skipped.

## Short answer

The plans name PDF, Word, slides, spreadsheets, CSV, TSV, Markdown, plain text, code, audio and video. They miss five groups that the evidence supports and that cost little to add.

1. Meeting transcripts that already exist as text. Microsoft Teams offers a transcript as `.docx` or `.vtt`, and Zoom saves `.vtt`. Docling 2.151.1 reads `.vtt` with start time, end time and speaker for every cue.
2. Email. Docling reads `.eml` and `.msg`, but it lists files attached to a message by name only and does not keep the Message-ID.
3. Exports from wikis and trackers that arrive as files. Confluence exports HTML and XML as zip archives, and its Word export is a multi-part mail file with a `.doc` extension. Docling reads HTML and MHTML. Nothing in the plans unpacks a zip archive.
4. Office files from before 2007 (`.doc`, `.xls`, `.ppt`). Docling reads them only when LibreOffice is installed next to it.
5. Diagrams. Docling reads no diagram source format. draw.io files are XML that a short reader of our own can turn into text. Images in repositories are not routed to the planned image reader.

The planned readers also lose content in file types that are already in scope. Docling puts PowerPoint speaker notes and Word and Excel comments on a separate layer that its default export leaves out. It reads the cached values of Excel cells and no formulas. It reads no VBA macros. It runs no text recognition on pictures inside Word, PowerPoint and Excel files.

No study counts the file types among the sources of modernization projects. I found none in this research. The frequency evidence is therefore indirect: vendor documentation of export formats, tool usage surveys and counts of file extensions in seven public repositories. Public repositories hold almost no office files (12 of 57,186 files), so what a client's file share holds rests on judgment.

A file the app cannot read should be stored, listed with a reason and a hint, and findable by name. It should never be dropped without a trace or chunked as garbage. The section "What the app should do with a file it cannot read" describes this.

## How to read the table

"Covered" means that a reader already in the plans reads the file type today, and at most a routing rule is missing. The planned readers are Docling for documents, tree-sitter through chonkie for code, and the heading-aware text splitter. "Partly covered" names what is lost. "Not covered" means none of them reads it.

The verdict has three classes.

- **V1** stands for "support and test in V1". The test dataset gets at least one item of this type.
- **Text or notice** stands for "accept as a file and treat as plain text or skip with a visible notice".
- **Later** stands for "leave for later". The file is stored and listed as not read.

License marks follow the rule that a license must allow free commercial use. MIT, BSD, Apache-2.0, ISC, CC0 and the Python license pass. GPL and AGPL fail for code we ship. LGPL and MPL-2.0 allow commercial use under conditions, so they are marked "owner call".

Docling facts come from release v2.151.1 (2026-11-19) and from `main` at commit `5182e55` (2026-11-20), both read on 2026-11-20. Grammar facts come from `tree-sitter-language-pack` 1.25.2 (2026-11-15, 371 grammars). Release dates of readers come from PyPI and the GitHub API on 2026-11-20.

## Table over all file types

| File type | Evidence of frequency | Current plan | Open reader where not covered | Citation anchor | Verdict |
|---|---|---|---|---|---|
| Teams or Zoom transcript `.vtt` | Teams offers the transcript as `.docx` or `.vtt` [(Microsoft)][teams]. Zoom saves audio transcripts as VTT [(Zoom)][zoom]. | Covered by Docling, not named in any ADR. | Not needed. | Time range and speaker. | V1 |
| Teams transcript `.docx` | Same Microsoft page. | Partly covered as a Word file. Times and names arrive as text, not as time anchors (judgment, not tested). | Not needed. | Heading or paragraph. | V1, as one more Word test item |
| Subtitles `.srt` | Judgment, no source. | Not covered by Docling. The text splitter reads it as plain text. | `pysubs2` (MIT, 1.9.1, 2026-10-30, library). | Time range. | Text or notice |
| Chat exports (Slack zip of JSON, Teams HTML transcripts) | Slack exports a zip with one JSON file per channel and day [(Slack)][slack]. Frequency in projects is judgment, no source. | HTML is covered by Docling. Slack JSON is not covered in a useful way. | Own reader on the Python standard library. | Channel and message timestamp. | Later |
| Email `.eml` | In a survey at one Dutch bank, 27% (7 of 26) passed on architecture knowledge by email in their last project [(Gebreweld)][gebreweld]. | Partly covered. Docling lists attached files by name only and drops the Message-ID. | Python standard library `email` for attached files and headers. | Message-ID, then the anchor inside an attached file. | V1 |
| Email `.msg` | Judgment, no source for the share of Outlook. | Partly covered, same losses as `.eml`. | `python-oxmsg` (MIT, 0.0.2, 2025-02-03, library), which Docling already uses. | Message-ID. | V1 |
| Mailbox `.pst` | Judgment, no source. | Not covered. | `libpff` (LGPL-3.0-or-later, owner call, 2026-10-30, library). Apache Tika (Apache-2.0, Java). `libpst` (GPL-2.0, fails). | Folder path and Message-ID. | Later |
| Mailbox `.mbox` | Judgment, no source. | Not covered. | Python standard library `mailbox`. | Message-ID. | Later |
| draw.io `.drawio` | One file in AWS's mainframe sample application (measured). Named by 2 of 28 architects in the bank survey [(Gebreweld)][gebreweld]. | Partly covered. It would be chunked as raw XML with style and geometry noise. | Own reader on the Python standard library. No maintained reading library found. | Diagram page name and cell id. | V1 |
| Visio `.vsdx`, `.vsd` | 9 of 31 architects in the bank survey capture architecture knowledge in Visio [(Gebreweld)][gebreweld]. One `.vsd` in `openemr/openemr` (measured). | Not covered. | `vsdx` (BSD-3-Clause, 0.6.1, 2026-01-04, library, `.vsdx` only). LibreOffice (MPL-2.0, owner call, runs without a window). | Page name and shape id. | Later, first in line |
| PlantUML, Mermaid | Judgment, no source. | Covered as code. Both grammars exist. | Not needed. | Line range. | Text or notice |
| BPMN 2.0 XML `.bpmn` | Named by 2 of 28 architects in the bank survey [(Gebreweld)][gebreweld]. | Partly covered as XML text. The flow between tasks is not interpreted. | `bpmn-moddle` (MIT, v10.3.1, 2026-09-21, JavaScript library). | Element id and line range. | Text or notice |
| ArchiMate exchange file, Archi `.archimate` | 17 of 31 architects (55%) in the bank survey use ArchiMate [(Gebreweld)][gebreweld]. | Partly covered as XML text. | Own reader on the Python standard library. Archi (MIT) is a desktop application. | Element identifier. | Text or notice |
| Enterprise Architect `.eap`, `.eapx`, `.qea` | 14 of 31 architects (45%) in the bank survey use Sparx Enterprise Architect [(Gebreweld)][gebreweld]. | Not covered. | `.qea` is SQLite, read by the Python standard library `sqlite3`. `.eap` needs `access-parser` (Apache-2.0, 0.0.6, 2025-01-23) or Jackcess (Apache-2.0, 5.2.0, 2026-10-16, Java). | Element GUID (not verified). | Later |
| Structurizr DSL | Judgment, no source. | Covered as plain text with line windows. No grammar exists. | Not needed. | Line range. | Text or notice |
| UML files `.xmi`, `.uml` | 21,316 UML files in 3,295 of 1.24 million GitHub repositories, 48.3% of them `.uml` or `.xmi` [(Hebig)][hebig]. | Partly covered as XML text. | Not needed for text. | Element id and line range. | Text or notice |
| SVG exports | 0.6% of the UML files in the same study. 2,109 `.svg` files in the seven repositories (measured), most of them icons (judgment). | Not covered by Docling. Chunked as XML with path data noise. | Own reader on the Python standard library that keeps text elements. | File and element id. | Text or notice |
| PNG and JPEG exports, screenshots | 51.7% of the UML files in the study are images [(Hebig)][hebig]. 58% of sketches are archived [(Baltes and Diehl)][baltes]. PNG is the most or second most common extension in three of seven repositories (measured). | Partly covered. Planned for project artifacts. Not routed for repository files. Text recognition returns labels and no arrows. | Not needed. | File and bounding box. | V1, for the routing gap |
| Word, Excel, PowerPoint 97 to 2003 `.doc`, `.xls`, `.ppt` | 5 such files in the seven repositories (measured). Otherwise judgment, no source. | Covered only if LibreOffice is installed next to Docling. | Not needed. | As for the newer formats. | V1 |
| Rich Text Format `.rtf` | Judgment, no source. | Covered through the same LibreOffice route. | Not needed. | Heading path. | Later |
| OpenDocument `.odt`, `.ods`, `.odp` | 4 files in the seven repositories (measured). | Covered by Docling's own reader. | Not needed. | Heading path, sheet, slide. | Later |
| OneNote `.one` | Judgment, no source. | Not covered. | `onenote_parser` (MPL-2.0, owner call, v2.1.0, 2026-10-19, Rust library). Apache Tika (Apache-2.0, Java). | Section and page title. | Later |
| Microsoft Project `.mpp` | Judgment, no source. | Not covered. | MPXJ (LGPL, owner call, v16.11.0, 2026-11-02, Java library). | Task id. | Later |
| Excel with macros `.xlsm` | Judgment, no source. | Partly covered. Docling reads the sheets and no VBA code. | `oletools` (BSD, 0.60.2, 2024-07-02, library) extracts VBA source. | Sheet and cell range. Module and line range for VBA. | V1 for sheets, later for VBA |
| Excel formulas | Judgment, no source. | Partly covered. Docling reads cached values only. | `openpyxl` (MIT), which Docling already uses, can load formulas. | Sheet and cell. | Later |
| Access `.mdb`, `.accdb` | Judgment, no source. | Not covered. | `access-parser` (Apache-2.0, tables only). Jackcess (Apache-2.0, Java). `mdbtools` (GPL-2.0, fails). | Table and row. | Later |
| Word comments and tracked changes | Judgment, no source. | Partly covered. Comments sit on a layer that the default export leaves out. Deleted text and the author of each change are lost. | Not needed. | Heading path and comment id. | V1 |
| PowerPoint speaker notes | Judgment, no source. | Partly covered. Notes sit on the same left-out layer. | Not needed. | Slide number. | V1 |
| Scanned PDF | Already planned in the document ingestion research. | Covered. | Not needed. | Page. | Planned |
| OpenAPI and Swagger | REST is used by 93% of respondents [(Postman)][postman]. No number for specification files found. | Covered as YAML or JSON code. Operations are not separate records. | `openapi-spec-validator` (Apache-2.0, 0.9.0, 2026-05-20) for a later splitter. | Line range and JSON pointer. | V1, as a test item |
| WSDL and XSD | 1 `.wsdl` and 51 `.xsd` files in the seven repositories (measured). | Covered as XML code. | `zeep` (MIT, 4.3.3) and `xmlschema` (MIT, 4.3.2) for a later splitter. | Line range. | V1, as a test item |
| AsyncAPI | Judgment, no source. | Covered as YAML code. | Not needed. | Line range. | Text or notice |
| SQL DDL dumps `.sql`, `.ddl` | AWS Transform lists Db2 DDL among the database assets it takes [(AWS)][aws-transform]. 122 `.sql` and 6 `.ddl` files in the seven repositories (measured). | Covered as code. The `sql` grammar exists. | `sqlglot` (MIT, 30.33.0, 2026-11-17) for a later splitter per table. | Line range. | V1, as a test item |
| ER model files (erwin, PowerDesigner) | Judgment, no source. | Not covered. | None found. | None. | Later |
| ReqIF `.reqif`, `.reqifz` | Judgment, no source. | Partly covered as XML text for `.reqif`. Not covered for the zipped `.reqifz`. | `reqif` (Apache-2.0, 0.1.1, 2026-11-15, library). | Requirement identifier. | Later |
| Jupyter notebook `.ipynb` | 14.1% of developers use Jupyter [(Stack Overflow)][so]. None in the seven repositories (measured). | Not covered. Read as JSON it would index embedded output images as text. | `nbformat` (BSD-3-Clause, 5.11.1, 2026-08-17, library). | Cell index. | Text or notice |
| AsciiDoc | 213 `.adoc` files in `apache/ofbiz-framework` (measured). | Covered, by Docling and by a grammar. No ADR names it. | Not needed. | Line range in a repository. | Text or notice |
| reStructuredText | The 748 Python enhancement proposals are `.rst` files (large legacy repositories note). | Covered as text. The `rst` grammar exists. Docling does not read it. | `docutils` (0.23, 2026-05-27, mixed public domain, BSD and GPL parts). | Line range. | Text or notice |
| LaTeX | 4 `.tex` files in the seven repositories (measured). | Covered, by Docling and by a grammar. | Not needed. | Line range. | Text or notice |
| Generated HTML documentation (Javadoc, Doxygen) | Judgment, no source. | Partly covered. The generated-file check catches Doxygen and not Javadoc. | Not needed. | File and heading. | Text or notice |
| HTML and MHTML exports | Confluence is used by 32.8% and Jira by 46.4% of developers [(Stack Overflow)][so]. Confluence exports HTML, XML, PDF and Word [(Atlassian)][confluence-export]. | Covered by Docling. No ADR names HTML as a document. | Not needed. | File and heading path. | V1 |
| man pages | Judgment, no source. | Covered as plain text. No grammar exists. | Not needed. | Line range. | Text or notice |
| XML, JSON, YAML, properties configuration | 4,427 `.xml` files in the seven repositories (measured). | Covered as code. | Not needed. | Line range. | Planned |
| Terraform, Helm, Dockerfile, CI pipelines | Docker is used by 71.1% and Terraform by 17.8% of developers [(Stack Overflow)][so]. | Covered as code. Grammars exist for `hcl`, `terraform`, `dockerfile`, `gotmpl`, `yaml` and `groovy`. | Not needed. | Line range. | Planned |
| Log files | Judgment, no source. | Covered as plain text. No rule exists for them. | Not needed. | Line range. | Text or notice |
| JUnit XML test reports | Judgment, no source. | Covered as XML text. | `junitparser` (Apache-2.0, 5.0.3, 2026-09-01). | Test case name. | Text or notice |
| Sample data in EBCDIC | AWS Transform takes usage records in raw binary EBCDIC [(AWS)][aws-transform]. Otherwise judgment, no source. | Not covered. ADR-0019 skips it as "unknown encoding". | Docling's EBCDIC reader, which needs the record layout. | Record number. | Later |
| Archives `.zip`, `.tar.gz` | Confluence and Slack exports are zip archives [(Atlassian)][confluence-export] [(Slack)][slack]. 29 `.zip` files in the seven repositories (measured). | Not covered. | Python standard library `zipfile` and `tarfile`. | Archive path, inner path, inner anchor. | V1 |
| Archives `.7z`, `.rar` | Judgment, no source. | Not covered. | `py7zr` (LGPL-2.1-or-later, owner call, 1.1.4, 2026-10-12). `libarchive` (v3.8.11, 2026-10-30). | As for zip. | Text or notice |
| Pictures inside Word, PowerPoint and Excel files | Judgment, no source. | Partly covered. Docling keeps the picture and reads no text in it. | Docling's own image path, fed with the extracted pictures. | Page or slide and picture number. | Later, with a count in the report |
| Compiled help `.chm`, `.hlp` | Judgment, no source. | Not covered. | Apache Tika (Apache-2.0, Java). `pychm` (GPL, fails). | Topic. | Later |

## Meeting and workshop records

A recorded Teams meeting already has a transcript. Microsoft's help page says that organizers "can download it as a .docx or .vtt file" and that the transcript "includes each speaker's name and time stamp" ([Microsoft][teams]). Zoom's help page says "Audio transcripts are saved in VTT format" ([Zoom][zoom]). A consultant who holds such a file should not have to run speech recognition on the recording again, because the platform's transcript carries real speaker names.

Docling reads `.vtt`. Its reader turns each cue into a text item with `start_time`, `end_time`, the cue identifier and the `voice`, which is the speaker name in a WebVTT voice tag ([webvtt_backend.py, lines 173 to 178][docling-vtt]). `docling-serve` 1.38.2 lists `vtt` among its input formats ([usage page][docling-serve-usage]). The glue step should write the cues in the transcript format of ADR-0002, one `**Name:**` line per speaker turn with its time range, so audio, video and ready transcripts end in one format and cite the same way. I did not test whether Zoom writes speaker names as voice tags or as plain text in front of the cue.

The `.docx` transcript goes through the Word reader. The time stamps then arrive as ordinary text, so an answer can quote them but cannot cite a time range as an anchor. This is a judgment from the file format. I did not convert a Teams file.

Docling has no `.srt` input format. The extension is absent from its format table ([base_models.py, lines 154 to 192][docling-formats-code]). An `.srt` file is plain text, so the text splitter reads it with the cue numbers and times in the text. `pysubs2` (MIT) or `srt` (MIT, 3.5.3, 2023-03-28) could convert it to VTT later. I found no source on how often `.srt` files turn up, so this is the class "text or notice".

Chat exports differ per platform. A Slack export is a zip archive with a folder per channel and one JSON file per day, and it holds links to uploaded files, not the files ([Slack][slack]). Microsoft's classic eDiscovery export wrote Teams chats as HTML transcript files ([Microsoft Learn][teams-ediscovery]). That page now applies only to the 21Vianet cloud, and I did not check the new eDiscovery experience. Chat exports can hold private messages of people who are not on the project. Leave them for later.

## Email

ADR-0006 itself uses "a decision log emailed by the client" as its example of a project artifact. The one survey with numbers that I found asked 45 architects at one Dutch bank. In their last project, 27% (7 of 26 answers) passed on architecture knowledge by email, and 37% (11 of 30) named email as a tool for sharing it ([Gebreweld][gebreweld], table 3). One bank is a small base, but banks are typical clients of a modernization project. The test dataset already holds 38 `.eml` files, which are test fixtures of the mail app and no project mail (measured).

Docling reads `.eml` and `.msg`. It writes the subject as title, then From, To, Cc and Date lines, the body and any calendar invitation ([email_backend.py, lines 544 to 617][docling-email]). Three things are lost.

- Docling lists the files attached to a message by name and type only. Its source says "Only attachment metadata is surfaced; the encoded payload is never included" ([lines 525 to 542][docling-email]). A decision log attached to a mail would not be read.
- The Message-ID and In-Reply-To headers do not reach the document, so there is no stable anchor and no thread link.
- Quoted earlier messages in the body are kept as body text, so the same text appears once per reply (judgment, not tested).

The glue step is small. Python's standard library `email` module reads the headers and saves each attached file, which then goes through the router like any other file with the mail as its parent. For `.msg`, Docling uses `python-oxmsg` (MIT), whose latest version is 0.0.2 from 2025-02-03, so `.msg` needs a test item of its own. `extract-msg`, the better known reader, is GPL and fails the license rule.

A `.pst` file holds a whole mailbox. Reading it needs `libpff` (LGPL-3.0-or-later, Python package `libpff-python` 20261030 from 2026-11-08) or Apache Tika's `OutlookPSTParser` in a Java process ([Tika formats][tika]). `libpst` with its `readpst` tool is GPL-2.0 and its last release is from 2021-12-12. A whole mailbox also brings mail that has nothing to do with the project. Leave `.pst` for later and tell the consultant to add the relevant messages as `.msg` files. `.mbox` is cheap, because Python's standard library `mailbox` module reads it ([Python documentation][py-mailbox]), but I found no sign that clients hand over mbox files.

## Diagrams and models

The evidence for diagrams is the best in this note, and it says that diagrams matter and that they arrive in many formats.

- In the bank survey, 31 architects named the tools they use to capture architecture knowledge. ArchiMate had 17 mentions (55%), PowerPoint 15 (48%), Sparx Enterprise Architect 14 (45%), Confluence 11 (35%), Word 10 and Visio 9 ([Gebreweld][gebreweld], table 2).
- A scan of 1.24 million GitHub repositories found 21,316 UML files in 3,295 repositories. By format, 44.9% were `.uml`, 3.4% `.xmi`, 29.6% PNG, 16.6% GIF, 4.7% JPEG, 0.6% SVG and 0.2% BMP ([Hebig et al.][hebig], table 1). Images are 51.7% of the files. I read the table from the PDF's text layer, where the shares add up to 100%.
- In a survey of 394 software practitioners, 58% of sketches and diagrams were archived, and 94% of the digital ones ([Baltes and Diehl][baltes], section 3.3.2).

Docling reads none of the diagram source formats. Its format table has no entry for `.drawio`, `.vsdx`, `.svg`, `.bpmn` or `.archimate` ([base_models.py, lines 154 to 192][docling-formats-code]).

**draw.io.** A `.drawio` file "is an uncompressed file using the XML format" ([draw.io][drawio-formats]). Shape labels sit in `value` attributes and connectors name their source and target cells. A reader of our own on the Python standard library can write one line per shape and one sentence per connector ("Billing batch connects to Customer table"), grouped by diagram page. The anchor is the page name and the cell id. I found no maintained Python library that reads draw.io files. `drawpyo` (MIT) writes them. Older draw.io files may store each diagram compressed, which I could not confirm from the draw.io documentation in this research. draw.io can also embed the diagram data in PNG and SVG files it saves ([draw.io][drawio-formats]), so the same reader can later recover the source from such an image. The verdict is V1, because the reader is small, AWS's own mainframe sample application documents its architecture in a `.drawio` file (measured), and consultants draw new diagrams in draw.io themselves (judgment).

**Visio.** The `vsdx` library (BSD-3-Clause, 0.6.1 from 2026-01-04) reads pages, shapes and their text from `.vsdx` files in pure Python. The older binary `.vsd` needs LibreOffice, whose Draw filter "Microsoft Visio 2000-2013" covers `vdx`, `vsd`, `vsdm`, `vsdx` and `vstx` ([LibreOffice 26.8 help][lo-filters]), or Apache Tika. Docling's LibreOffice route does not include Visio. Converting a Visio file to PDF with LibreOffice and then reading the PDF with Docling would give page anchors with no new library, but I did not test it. Visio is the first type to add after V1. Until then the notice should ask for a PDF export, which V1 reads with page anchors.

**Text formats.** PlantUML and Mermaid files are text, and `tree-sitter-language-pack` 1.25.2 has grammars for both, so the code path reads them and cites line ranges. Structurizr DSL has no grammar and falls back to line windows. C4 is a notation and no file type. A C4 model arrives as Structurizr DSL, PlantUML, Mermaid, draw.io or an image.

**XML model formats.** BPMN 2.0, the ArchiMate exchange format ([The Open Group][archimate]), Archi's `.archimate` files and UML files in `.xmi` or `.uml` are XML. The code path reads them as XML text, where the names of tasks and elements are attribute values. A keyword search finds them. The relations between elements stay unread. Reading them properly means one reader per format that writes sentences from relations. That is later work, and the bank survey suggests starting with ArchiMate.

**Enterprise Architect.** Sparx stores a model as one database file. The current `.qea` format is SQLite, `.eap` is Microsoft Jet 3.5 and `.eapx` is Jet 4.0 ([Sparx user guide 17.2][sparx]). Python's standard library opens a `.qea` file. An `.eap` file needs `access-parser` (Apache-2.0), whose README says it "was tested on a limited subset of database files", or Jackcess in a Java process. I did not check the table layout. Leave these for later and ask for an HTML or PDF report from the tool.

**Images of diagrams.** Docling reads PNG, JPEG, TIFF, BMP, WEBP and GIF with text recognition, and the document ingestion research already plans this for project artifacts. Two gaps remain. First, ADR-0019 sends only "PDF, Word, slides and spreadsheets" from a repository to Docling, so an architecture picture in `docs/` is skipped as binary. PNG is the most or second most common extension in `dotnet-architecture/eShopModernizing` (329 of 1,299 files), `apache/ofbiz-framework` (1,337 of 6,475) and `openmainframeproject/cobol-programming-course` (223 of 367). Second, text recognition returns the labels of boxes and nothing about arrows. Docling can also ask a vision model to describe a picture (`do_picture_description`, [pipeline_options.py][docling-pipeline-options]), which falls under the model policy and was not tested. V1 should route repository images to the image path and filter out icons by pixel size. The threshold is an estimate to tune on the test dataset.

**SVG.** Docling does not read SVG, and there is no `svg` grammar, so the `xml` grammar would chunk it with all path data. A reader that keeps only `text`, `title` and `desc` elements is a few lines on the standard library. Until it exists, skip SVG files with a notice.

## Older and other office formats

**Word, Excel and PowerPoint 97 to 2003.** Docling's format page lists DOC, XLS and PPT as legacy binary Office formats and adds "requires LibreOffice". It says the same for RTF ([supported formats][docling-formats]). The document ingestion research mentions this route, but no ADR and no install step puts LibreOffice next to `docling-serve`. The systems a modernization project replaces are older than the 2007 file formats, so their specifications often are too (judgment, no source). The seven repositories hold 4 `.doc` files and 1 `.xls` file. Take the three formats into V1 with one test item each.

One trap belongs here. Confluence's "Export to Word" writes a `.doc` file that is a multi-part mail file. Atlassian's own ticket says "It is not a real binary .doc file which can be opened by other tools" ([CONFCLOUD-38237][confluence-doc], closed unresolved in 2025). The ticket does not name the format. A multi-part mail file with an HTML body is MHTML (judgment). Routing by extension would send it to LibreOffice. Routing by content sends it to Docling's MHTML reader, and Magika has an `mht` label for this ([Magika model configuration][magika]). A second example is `.dot`, which Docling maps to a Word 97 template and tree-sitter to Graphviz. `openemr/openemr` holds 480 `.dot` files (measured, content not checked).

**Notes, comments and tracked changes.** Docling reads all three kinds of side text and then leaves them out of its default output.

- PowerPoint speaker notes become a text item with the slide number on the `NOTES` content layer ([mspowerpoint_backend.py, lines 2385 to 2402][docling-pptx]). Hidden slides go to the `INVISIBLE` layer.
- Word comments become a comment group on the `NOTES` layer, linked to the paragraph they annotate ([msword_backend.py, lines 1127 to 1133][docling-docx]). Excel cell comments, old and threaded, are read too.
- The default export contains only the `BODY` layer: `DEFAULT_CONTENT_LAYERS = {ContentLayer.BODY}` ([docling-core 2.108.0][docling-core-layers]). The `docling-serve` usage page lists no option for content layers (search for "content_layer" on 2026-11-20, 0 hits). The ingestion job must therefore take Docling's JSON output and export the notes layer itself.
- Text inserted as a tracked change is kept. Text deleted as a tracked change is dropped, and so are the author and date of each change. This follows from comments in Docling's source ([msword_backend.py, lines 2307 to 2308][docling-docx-deleted]). I did not test it.

Speaker notes often hold the explanation that the slide only hints at, and review comments hold open questions (judgment, no source). Both are cheap to keep, so they belong in V1 with a deck and a Word file that test them.

**Excel.** Docling maps `.xlsm` to its Excel reader, so the sheets of a workbook with macros are read. Three things are lost in every Excel file.

- VBA code is not read. The Excel reader has no code for it (search for "vba" and "macro" in `msexcel_backend.py`, 0 hits). `oletools` (BSD, 0.60.2 from 2024-07-02) extracts VBA source from `.xlsm`, `.xls`, `.docm` and `.doc` as a Python library ([olevba documentation][olevba]). V1 can detect macros with no new library and say so in the report, because an `.xlsm` file is a zip archive that holds the macros in a `xl/vbaProject.bin` member (from the file format, not tested).
- Formulas are not read. Docling loads workbooks with `data_only=True`, which gives the cached value of each cell ([msexcel_backend.py, lines 978 to 985][docling-xlsx]). A business rule that lives in a formula is invisible.
- Hidden sheets go to the `INVISIBLE` layer and are left out of the default export ([lines 3151 to 3156][docling-xlsx-hidden]).

`.xlsb`, the binary Excel format, is not in Docling's format table. LibreOffice lists a filter for it ([LibreOffice help][lo-filters]). `pyxlsb` is LGPLv3+ and its last release is from 2022-10-14.

**Other formats.** OpenDocument files are read by Docling's own reader with no LibreOffice. OneNote, Microsoft Project and Access files have no reader in the plans. `onenote_parser` (MPL-2.0, Rust) reads the desktop format and notebooks downloaded from OneDrive. MPXJ (LGPL, Java) reads `.mpp`. An Access file is an application with tables, queries, forms and VBA, so it belongs with the binary code containers in the map's "Not yet specified". For all of these the notice should ask for a PDF export.

## Structured technical documents

Most files in this category are text with a grammar in `tree-sitter-language-pack` 1.25.2, so the code path of ADR-0019 already reads them and cites line ranges. The pack's language list has these names: `yaml`, `json`, `xml`, `dtd`, `sql`, `hcl`, `terraform`, `dockerfile`, `gotmpl`, `groovy`, `properties`, `ini`, `toml`, `proto`, `graphql`, `rst`, `latex`, `asciidoc`, `mermaid`, `plantuml`, `dot` and `cobol`. It has none of these: `svg`, `plsql`, `vba`, `jcl`, `roff`, `structurizr` and `helm`.

What the code path loses is the unit a question asks about. An OpenAPI file is chunked by YAML or JSON syntax nodes, so one operation can be split over two chunks or share a chunk with another. The same holds for a WSDL port type and for one `CREATE TABLE` statement in a long DDL dump. A splitter per operation or per table is later work with `openapi-spec-validator`, `zeep`, `xmlschema` or `sqlglot`, all with passing licenses. V1 should still hold one OpenAPI file, one WSDL with its XSD and one DDL file in the test dataset, because they are central to a modernization project and the test dataset has none. `thunderbird/thunderbird-android` holds 0 `.sql`, 0 `.wsdl` and 0 `.xsd` files (measured). AWS Transform lists "DB2 DCLGEN and DDL scripts" among the database assets of a mainframe project ([AWS][aws-transform]).

For text formats inside a repository, the text splitter is the better route than Docling even where Docling has a reader. Docling reads AsciiDoc and LaTeX, but its anchors are headings, and ADR-0019 cites repository files by line range. AsciiDoc, reStructuredText and LaTeX should go to the heading-aware text splitter with their heading syntax added. reStructuredText matters for decision records, since the 748 Python enhancement proposals are `.rst` files with a `Status:` header (large legacy repositories note on the branch `research/large-legacy-test-repositories`).

Generated HTML documentation repeats comments from source code that is already ingested. ADR-0019 uses GitHub Linguist's generated-file checks. For HTML, these catch pkgdown, mandoc, Doxygen, Org mode, latex2html, groff, makeinfo, texi2html and ronn, and they do not catch Javadoc ([generated.rb, lines 913 to 946][linguist]). Javadoc pages carry the comment "Generated by javadoc" ([OpenJDK `Head.java`][javadoc-head]), so one more rule tags them. If a client has lost the source and only the Javadoc remains, the consultant force-includes the folder and Docling's HTML reader takes it.

ReqIF is the OMG's XML format for exchanging requirements between tools ([OMG ReqIF 1.2][reqif-spec]). The `reqif` library (Apache-2.0, 0.1.1 from 2026-11-15) parses `.reqif` and zipped `.reqifz` files. I found no source on how often a modernization project receives ReqIF, so it waits. Jupyter notebooks are JSON with cell outputs embedded as encoded images. Reading them as JSON would index that data as text, so skip them with a notice until an `nbformat` reader exists. Files of ER modeling tools such as erwin and PowerDesigner have no open reader that I found. I did not search for one in depth. Ask for a DDL or PDF export.

## Data and configuration

Configuration files, infrastructure code and CI pipelines are code under ADR-0019 and need no new reader. The test dataset already holds 3,013 `.xml`, 239 `.json`, 57 `.yml` or `.yaml` files and 27 GitHub workflows (measured).

Log files are plain text, and ADR-0019 has no rule for them below its 20 MB cap. They are large, repeat themselves and can hold personal data. Skip `.log` files by default with a notice, and let a consultant force-include one. JUnit XML reports are build output. Tag them as generated.

Sample data in CSV and TSV is planned. Mainframe data files in EBCDIC are skipped as "unknown encoding" by ADR-0019. Docling now lists an EBCDIC reader for "Mainframe fixed-width data files" that "needs the COBOL record layout" ([supported formats][docling-formats]), and an AFP reader for mainframe print output. Both are worth a look once public legacy test data exists, together with the legacy languages in the map's "Not yet specified".

## Containers and embedded images

**Archives.** Many sources arrive packed. Confluence "can create a zipped archive of the HTML" and of the XML ([Atlassian][confluence-export]), a Slack export is a zip archive ([Slack][slack]), and AWS Transform takes a mainframe inventory as "a single .zip file" ([AWS][aws-transform]). The seven repositories hold 29 `.zip` files. Docling reads no general archive. The only archive inputs in its format table are its own DocLang bundle (`.dclx`) and a `tar.gz` layout for METS files ([base_models.py][docling-formats-code]). Python's standard library `zipfile` and `tarfile` unpack `.zip`, `.tar` and `.tar.gz` with no new dependency. V1 should unpack these, route every member by type, and cite the archive path, the inner path and the member's own anchor. The unpack step needs limits on member count, total size and nesting depth, and it must reject paths that leave the target folder. Docling sets the same kind of limits for the zip containers it reads (`max_total_bytes`, `max_file_bytes`, `max_member_count` in [backend_options.py][docling-backend-options]). An encrypted archive gets a notice. `.7z` and `.rar` need `py7zr` (LGPL-2.1-or-later) or `libarchive`, so they get a notice that asks the consultant to unpack them.

**Pictures inside documents.** Docling keeps a picture from a Word, PowerPoint or Excel file as a picture item. The pipeline for these formats has no text recognition step ([simple_pipeline.py][docling-simple-pipeline], read on 2026-11-20). A screenshot or diagram pasted into a Word specification therefore contributes no text. PDF pages do go through text recognition, as the document ingestion research describes. This is the largest loss among the planned formats (judgment), because architecture documents in Word are often a few paragraphs around pasted diagrams. Two fixes exist, and I tested neither. The job can take the pictures from Docling's output and send each through the image path. Or it can convert the file to PDF with LibreOffice and run the PDF path as well. V1 should at least count the pictures per document in the ingestion report.

## Other file types found

- Compiled help files (`.chm`, `.hlp`) are the user manuals of old desktop applications (judgment, no source). Magika labels both. Apache Tika has a `ChmParser`. `pychm` is GPL.
- Password-protected files cannot be read without the password. Docling takes a PDF password as an option. `msoffcrypto-tool` (MIT, 6.1.0 from 2026-11-03) decrypts Office files when the password is known.
- Key and certificate files should be skipped with their own reason. The test dataset holds 4 `.pem` files (measured). ADR-0019 has no rule for secrets.
- DocBook and DITA are XML and read as XML text.
- Docling also reads EPUB, Apple Pages, Numbers and Keynote, and calendar files. None of them is asked for. Keep them switched off.

## What the app should do with a file it cannot read

The tools around the project memory show two bad defaults. Open WebUI's default loaders hand any unknown file to a text loader, PNG and JPEG included (document ingestion research). The Bedrock Managed Knowledge Base accepts seven document formats plus JPEG and PNG images ([AWS][bedrock-formats]), which does not matter here because ADR-0017 feeds it our own chunks. Docling does the right thing at its level. For a format outside its allow-list it returns the status `SKIPPED` with the error "File format not allowed" ([document_converter.py, lines 1071 to 1079][docling-converter]).

ADR-0019 already requires an ingestion report for repository files that lists every skipped file with path, size and reason, and a force-include per project memory. Project artifacts have no such rule yet. I recommend one rule for both origins.

1. **Detect the type by content, not by extension.** Magika's `standard_v3_3` model has 214 labels, among them `outlook`, `one`, `sevenzip`, `rar`, `sqlite`, `chm`, `xlsb`, `vtt`, `srt`, `svg`, `ipynb`, `eml`, `mht` and `ics` ([Magika model configuration][magika]). It has no label for Visio, draw.io, Microsoft Project or Access. Two small checks cover those. A zip archive with a `visio/` folder is a `.vsdx` file (judgment from the file format, not tested), and a draw.io file is XML with an `mxfile` root element.
2. **Keep the file.** Store every added file among the plain files of the project memory (ADR-0022), read or not. A later version can then read it without a new upload, and the bundle carries it.
3. **Give every file one status.** The statuses are read, read in part, stored and not read, skipped by rule, blocked and failed. "Read in part" names the loss, for example "3 pictures not read" or "VBA macros not read". "Blocked" covers password protection and archive limits.
4. **Tell the consultant twice.** Show the status right after the upload or sync, with the type in plain words and a hint. An example is "Visio drawing, not read. Export it as PDF and add the PDF." Then keep a list of all files that were not read or read in part per project memory, with counts per reason. This is the ingestion report of ADR-0019, extended to project artifacts.
5. **Make the file findable by name.** Write one record per unread file with its name, path, type, size and dates. A question such as "is there a Visio diagram of the billing batch?" then finds the file, and the answer can say that the file exists and was not read. Without this record the project memory would answer as if the file did not exist.
6. **Never chunk bytes as text.** A file goes to the plain text path only if it decodes as text under the encoding rule of ADR-0019. A consultant can force a text reading per path.
7. **Switch on only what is tested.** `docling-serve` takes a `from_formats` list that "defaults to all formats" ([usage page][docling-serve-usage]). Set it to the formats with a test item. A format Docling could read but nobody tested then shows as "stored and not read" and does not slip in untested.
8. **Test the notice.** Put one file of a type V1 does not read and one password-protected Word file into the test dataset. A check without a language model asserts the status, the reason and the hint, next to the edge-case files of ADR-0019.

## What this means for the test dataset

If the owner adopts the V1 class as proposed, the companion repository needs these synthetic items, each marked `synthetic: true` where the format allows it.

- One `.vtt` transcript with speaker names and one Teams-style `.docx` transcript.
- One `.eml` and one `.msg` message, each with a Word decision log attached.
- One `.doc`, one `.xls` and one `.ppt` file.
- One PowerPoint file with speaker notes and a hidden slide, and one Word file with comments and tracked changes.
- One `.xlsm` workbook with a macro, a formula and a hidden sheet.
- One Confluence HTML export as a zip archive and one Confluence Word export.
- One `.drawio` diagram and one PNG export of it in a repository `docs/` folder.
- One OpenAPI file, one WSDL with an XSD, and one DDL file.
- One file V1 does not read, for example a `.vsdx`, and one password-protected Word file.

Half of these should be German, as the document ingestion research already asks for the planned formats.

## Method and limits

- I read the map issue, ADR-0002, 0004, 0006, 0011, 0018 and 0019 on `main`, and the research notes on the branches `research/document-ingestion`, `research/github-ingestion`, `research/local-transcription`, `research/aws-models` and `research/large-legacy-test-repositories`, all on 2026-11-20.
- Docling. I read `docs/usage/supported_formats.md` and nine source files at commit `5182e556` of `docling-project/docling` (2026-11-20), one day after release v2.151.1 (2026-11-19, from `gh api repos/docling-project/docling/releases/latest`). A format added on `main` after the release may be missing from the released package. `docling-serve` 1.38.2 lists no `calendar` input, for example. I read source code and ran no conversion.
- Grammars. I searched `sources/language_definitions.json` of `xberg-io/tree-sitter-language-pack` at tag v1.25.2 for each name. The file has 371 entries.
- Magika. I read `assets/models/standard_v3_3/config.min.json` at commit `9f2f5281` of `google/magika`. It has 214 target labels.
- Readers. License, version and date come from `https://pypi.org/pypi/<name>/json` and from `gh api repos/<owner>/<repo>` and its `releases/latest` endpoint on 2026-11-20. GitHub reports no license id for `oletools`, `zeep`, `junitparser` and `libarchive`. For the first three I used the PyPI metadata. I did not read the license file of `libarchive`.
- Extension counts. I listed the default branch tree of seven public repositories through the GitHub API (`git/trees/<sha>?recursive=1`) on 2026-11-20 and counted file extensions. A count by extension says nothing about the content of a file. The repositories and commits are `thunderbird/thunderbird-android` at `2d881297` (8,604 files), `aws-samples/aws-mainframe-modernization-carddemo` at `59cc6c2f` (329), `dotnet-architecture/eShopModernizing` at `63bc9ec4` (1,299), `apache/ofbiz-framework` at `0f1fc60b` (6,475), `openmainframeproject/cobol-programming-course` at `780b1c96` (367), `shopware/shopware` at `551447a2` (29,580) and `openemr/openemr` at `48c71737` (10,532). Together they hold 57,186 files, of which 12 are office files (4 `.doc`, 2 `.docx`, 1 `.xls`, 1 `.xlsx`, 1 `.odt`, 3 `.ods`), 60 are PDF, 1 is Visio and 1 is draw.io. None is a `.msg`, `.pst`, `.mbox`, `.vtt`, `.srt`, `.one`, `.mpp`, `.mdb`, `.accdb`, `.bpmn`, `.archimate`, `.eap`, `.qea`, `.ipynb` or `.reqif` file. The seven repositories are my choice and no random sample.
- Web pages. I read vendor pages and papers through a fetch tool that returns a summary with quotes, on 2026-11-20. A quote may differ from the page in small ways. I read the Hebig paper from its PDF text.

Limits:

- I ran no parser and converted no file. Every statement on what a reader keeps or loses comes from documentation or source code.
- No study of file types in modernization projects was found. The bank survey has 45 respondents from one company and is an arXiv preprint. The UML study and the sketch survey are from 2016 and 2014. Rows marked "judgment, no source" have no evidence beyond my own reasoning.
- Public repositories cannot show what a client's file share holds, because teams rarely commit office files, mail or model databases.
- Not verified: how draw.io stores compressed diagrams, whether Zoom's VTT files carry voice tags, the table layout of Enterprise Architect files, whether LibreOffice converts Visio files well, whether the new Microsoft eDiscovery exports Teams chats as HTML, whether Jira still exports HTML, and whether the MPXJ Python package needs a Java runtime.
- The Stack Overflow figures describe all developers, not modernization projects. Postman's report gives no share for OpenAPI files.
- I could not retrieve the technical report of Khadka et al. on how professionals see legacy systems (HTTP 403), so this note makes no claim from it. I could not retrieve usage figures from the JetBrains developer survey.
- Apache Tika facts come from the formats page of version 3.2.3. The document ingestion research names 4.2.0 as the current release, whose formats page I did not read.
- Whether LGPL and MPL-2.0 readers pass the license rule is an owner call.

## Sources

All read on 2026-11-20.

Vendor documentation:

- [View live transcription in Microsoft Teams meetings][teams], Microsoft Support
- [Audio transcription for cloud recordings][zoom], Zoom Support
- [eDiscovery (Premium) workflow for content in Microsoft Teams][teams-ediscovery], Microsoft Learn
- [Export content to Word, PDF, HTML and XML][confluence-export], Atlassian
- [CONFCLOUD-38237, "Export to Word" results in '.doc' files][confluence-doc], Atlassian
- [Export search results][jira-export], Atlassian, which lists CSV, Google Sheets, Word, Excel and XML for Jira
- [How to read Slack data exports][slack], Slack
- [Mainframe source artifacts collection][aws-transform], AWS Transform user guide
- [Prerequisites for your Amazon Bedrock knowledge base data][bedrock-formats], AWS
- [File-based projects, user guide 17.2][sparx], Sparx Systems, read through a search result summary
- [Save file formats][drawio-formats] and [Export diagrams][drawio-export], draw.io
- [ArchiMate Model Exchange File Format][archimate], The Open Group
- [ReqIF 1.2][reqif-spec], Object Management Group
- [File conversion filter names, LibreOffice 26.8 help][lo-filters], The Document Foundation
- [Supported document formats, Apache Tika 3.2.3][tika]
- [mailbox, Python 3.15 documentation][py-mailbox]

Studies and surveys:

- [Gebreweld, "Architecture Knowledge Representation and Communication Industry Survey", arXiv 2309.11572][gebreweld]
- [Hebig, Ho-Quang, Chaudron, Robles and Fernández, "The quest for open source projects that use UML: mining GitHub", MODELS 2016][hebig], DOI 10.1145/2976767.2976778
- [Baltes and Diehl, "Sketches and Diagrams in Practice", FSE 2014, arXiv 1706.09172][baltes]
- [Stack Overflow Developer Survey 2025, Technology][so], 30,065 responses on documentation and collaboration tools, 24,473 on infrastructure tools, 26,143 on editors
- [Postman, State of the API 2025][postman], over 5,700 respondents

Source code and package data:

- Docling at commit `5182e556`: [supported formats][docling-formats], [base_models.py][docling-formats-code], [webvtt_backend.py][docling-vtt], [email_backend.py][docling-email], [mspowerpoint_backend.py][docling-pptx], [msword_backend.py][docling-docx], [msexcel_backend.py][docling-xlsx], [simple_pipeline.py][docling-simple-pipeline], [pipeline_options.py][docling-pipeline-options], [backend_options.py][docling-backend-options], [document_converter.py][docling-converter]
- [docling-core 2.108.0, content_layer.py][docling-core-layers]
- [docling-serve 1.38.2, usage page][docling-serve-usage]
- [tree-sitter-language-pack 1.25.2, language_definitions.json][tslp]
- [Magika, standard_v3_3 model configuration][magika]
- [GitHub Linguist, generated.rb][linguist] at commit `cfd7a011`
- [OpenJDK, Head.java of the javadoc tool][javadoc-head]
- [olevba documentation, oletools 0.60.2][olevba]
- Reader repositories: [vsdx][vsdx], [access_parser][access-parser], [onenote.rs][onenote-rs], [reqif][reqif-lib], [python-oxmsg][oxmsg], [libpff][libpff], [MPXJ][mpxj], [pysubs2][pysubs2], [py7zr][py7zr]

[teams]: https://support.microsoft.com/en-us/office/view-live-transcription-in-microsoft-teams-meetings-dc1a8f23-2e20-4684-885e-2152e06a4a8b
[zoom]: https://support.zoom.com/hc/en/article?id=zm_kb&sysparm_article=KB0064927
[teams-ediscovery]: https://learn.microsoft.com/en-us/purview/ediscovery-teams-workflow
[confluence-export]: https://confluence.atlassian.com/doc/export-content-to-word-pdf-html-and-xml-139475.html
[confluence-doc]: https://jira.atlassian.com/browse/CONFCLOUD-38237
[jira-export]: https://support.atlassian.com/jira-software-cloud/docs/export-search-results/
[slack]: https://slack.com/help/articles/220556107-How-to-read-Slack-data-exports
[aws-transform]: https://docs.aws.amazon.com/transform/latest/userguide/transform-app-mainframe-source-artifacts.html
[bedrock-formats]: https://docs.aws.amazon.com/bedrock/latest/userguide/knowledge-base-ds.html
[sparx]: https://sparxsystems.com/resources/user-guides/17.2/repository/file-based-projects.pdf
[drawio-formats]: https://www.drawio.com/doc/faq/save-file-formats
[drawio-export]: https://www.drawio.com/doc/faq/export-diagram
[archimate]: https://www.opengroup.org/open-group-archimate-model-exchange-file-format
[reqif-spec]: https://www.omg.org/spec/ReqIF/1.2/About-ReqIF
[lo-filters]: https://help.libreoffice.org/latest/en-US/text/shared/guide/convertfilters.html
[tika]: https://tika.apache.org/3.2.3/formats.html
[py-mailbox]: https://docs.python.org/3/library/mailbox.html
[gebreweld]: https://arxiv.org/abs/2309.11572
[hebig]: https://se.informatik.uni-rostock.de/storages/uni-rostock/Alle_IEF/Inf_SE/models.pdf
[baltes]: https://arxiv.org/abs/1706.09172
[so]: https://survey.stackoverflow.co/2025/technology/
[postman]: https://www.postman.com/state-of-api/2025/
[docling-formats]: https://github.com/docling-project/docling/blob/5182e556308958f3ac4762bb7d03a092c2f339c9/docs/usage/supported_formats.md
[docling-formats-code]: https://github.com/docling-project/docling/blob/5182e556308958f3ac4762bb7d03a092c2f339c9/docling/datamodel/base_models.py#L154-L192
[docling-vtt]: https://github.com/docling-project/docling/blob/5182e556308958f3ac4762bb7d03a092c2f339c9/docling/backend/webvtt_backend.py#L173-L178
[docling-email]: https://github.com/docling-project/docling/blob/5182e556308958f3ac4762bb7d03a092c2f339c9/docling/backend/email_backend.py#L525-L617
[docling-pptx]: https://github.com/docling-project/docling/blob/5182e556308958f3ac4762bb7d03a092c2f339c9/docling/backend/mspowerpoint_backend.py#L2385-L2402
[docling-docx]: https://github.com/docling-project/docling/blob/5182e556308958f3ac4762bb7d03a092c2f339c9/docling/backend/msword_backend.py#L1127-L1133
[docling-docx-deleted]: https://github.com/docling-project/docling/blob/5182e556308958f3ac4762bb7d03a092c2f339c9/docling/backend/msword_backend.py#L2307-L2308
[docling-xlsx]: https://github.com/docling-project/docling/blob/5182e556308958f3ac4762bb7d03a092c2f339c9/docling/backend/msexcel_backend.py#L978-L985
[docling-xlsx-hidden]: https://github.com/docling-project/docling/blob/5182e556308958f3ac4762bb7d03a092c2f339c9/docling/backend/msexcel_backend.py#L3151-L3156
[docling-simple-pipeline]: https://github.com/docling-project/docling/blob/5182e556308958f3ac4762bb7d03a092c2f339c9/docling/pipeline/simple_pipeline.py
[docling-pipeline-options]: https://github.com/docling-project/docling/blob/5182e556308958f3ac4762bb7d03a092c2f339c9/docling/datamodel/pipeline_options.py#L1511-L1551
[docling-backend-options]: https://github.com/docling-project/docling/blob/5182e556308958f3ac4762bb7d03a092c2f339c9/docling/datamodel/backend_options.py
[docling-converter]: https://github.com/docling-project/docling/blob/5182e556308958f3ac4762bb7d03a092c2f339c9/docling/document_converter.py#L1071-L1079
[docling-core-layers]: https://github.com/docling-project/docling-core/blob/v2.108.0/docling_core/types/doc/common/content_layer.py
[docling-serve-usage]: https://github.com/docling-project/docling-serve/blob/v1.38.2/docs/usage.md
[tslp]: https://github.com/xberg-io/tree-sitter-language-pack/blob/v1.25.2/sources/language_definitions.json
[magika]: https://github.com/google/magika/blob/9f2f5281a215f71ebfab72f90637df541e4a7113/assets/models/standard_v3_3/config.min.json
[linguist]: https://github.com/github-linguist/linguist/blob/cfd7a0113aa1/lib/linguist/generated.rb#L913-L946
[javadoc-head]: https://github.com/openjdk/jdk/blob/master/src/jdk.javadoc/share/classes/jdk/javadoc/internal/doclets/formats/html/markup/Head.java
[olevba]: https://github.com/decalage2/oletools/blob/v0.60.2/oletools/doc/olevba.md
[vsdx]: https://github.com/dave-howard/vsdx
[access-parser]: https://github.com/claroty/access_parser
[onenote-rs]: https://github.com/msiemens/onenote.rs
[reqif-lib]: https://github.com/strictdoc-project/reqif
[oxmsg]: https://github.com/scanny/python-oxmsg
[libpff]: https://github.com/libyal/libpff
[mpxj]: https://github.com/joniles/mpxj
[pysubs2]: https://github.com/tkarabela/pysubs2
[py7zr]: https://github.com/miurahr/py7zr
