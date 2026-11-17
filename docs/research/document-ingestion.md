# How to ingest folders and documents beyond the repository

Date: 2026-11-17. Ticket: [#16](https://github.com/Niko4417/OPT-cc/issues/16). Map: [#1](https://github.com/Niko4417/OPT-cc/issues/1).

## Question

How should the project memory ingest a local folder or repository that is not on GitHub, plus documents added by hand: PDF, Word, PowerPoint, Excel, scanned pages and images with text? Which local, open-source document parser fits best on German and English layout, tables, OCR, commercial licensing, speed and memory on a 64 GB Mac, and citations to a page or section? How does this fit next to the GitHub ingestion job from ticket [#5](https://github.com/Niko4417/OPT-cc/issues/5), and how do the shortlisted stacks (Keiko, Open WebUI, LightRAG) handle such documents themselves?

## Answer

Use Docling as the document parser, run natively as `docling-serve` on 127.0.0.1, inside one document ingestion job that sits next to the GitHub ingestion job and writes the same chunk records.

- Docling is the only parser in the comparison that combines permissive licenses for code and models, a native Mac install, OCR that reads German, table structure and page anchors. Its code is MIT, its layout and table models are Apache-2.0 and CDLA-Permissive-2.0, it installs with pip, it reads PDF, DOCX, PPTX, XLSX, HTML, email and images, and on macOS it picks Apple's Vision OCR, which reads German. Every Docling item keeps its page number and bounding box, and its chunker keeps the heading path. Open WebUI and LightRAG can both call `docling-serve` directly.
- Docling's weak spot is quality evidence. It is absent from the current OmniDocBench and olmOCR-bench tables. Its last OmniDocBench entry (Docling 2.14.0, January 2025) has an English text edit distance of 0.416, far behind the leaders. For hard scans, keep a second route to PaddleOCR-VL 1.6 (Apache-2.0, 96.34 on OmniDocBench v1.6, runs on Apple Silicon through MLX-VLM or llama.cpp), measured on our own pages first.
- MinerU 4 has strong scores, but its license adds a commercial-license duty above USD 20 million monthly revenue or 100 million monthly users, counted with affiliates. Marker's code is now Apache-2.0, but its model weights are free only for organizations under USD 5 million funding or revenue. PyMuPDF and pymupdf4llm are AGPL-3.0, and their Office support is proprietary. Unstructured is Apache-2.0 but scores lowest on OmniDocBench tables and sends telemetry by default. Apache Tika has no layout or table model for scans.
- No published benchmark I found reports a German-only score for any of these parsers. OmniDocBench and olmOCR-bench publish no per-language German result, and Datalab's OmniParseBench reports only a "not English" slice. The gold set from ticket [#11](https://github.com/Niko4417/OPT-cc/issues/11) needs German scanned and Office pages of its own.
- Memory decides how parsing runs. Ticket [#4](https://github.com/Niko4417/OPT-cc/issues/4) leaves 6.65 GB free at typical load and 4.41 GB at the upper bound. Docling on CPU peaks at 1.98 GB for 9 pages and 5.28 GB for 144 pages ([docling#4705](https://github.com/docling-project/docling/issues/4705)). So the job converts PDFs in windows of at most 32 pages with Docling's `page_range`, runs one worker, and starts a VLM parser only while the model servers are stopped.
- A local git repo from any host goes through the same path as the GitHub job, minus the REST calls. `git ls-files` at a pinned commit lists the files and applies every `.gitignore`. `git diff --name-status` gives the changes since the last sync. `git log` can add commit messages as history documents. A plain folder without git is walked with gitignore-style rules from a `.memoryignore` file, fingerprinted with git blob SHA-1 hashes, and rescanned on demand or watched through FSEvents.
- Keiko already parses PDF, DOCX, PPTX and XLSX in TypeScript with page numbers, page labels and section paths, but it has no OCR for scanned PDF pages and reads only the root `.gitignore`. Open WebUI's default loaders keep PDF pages but read PNG and JPEG files as text, so it needs the Docling engine. LightRAG keeps pages in its parse sidecars, but its answer references carry only the file path.

## Method

- I read source code and docs on 2026-11-17 through the GitHub API, and model cards through the Hugging Face API. I installed nothing, downloaded no models and measured no parser myself.
- Repos and commits read:

| Repo | Commit or release |
|---|---|
| docling-project/docling | `ce69639a`, release v2.150.1 (2026-11-14) |
| docling-project/docling-serve | release v1.38.2 (2026-10-27) |
| opendatalab/MinerU | `b4efa4b2`, release 4.2.3 (2026-11-16) |
| datalab-to/marker | `d580b638`, release v2.2.0 (2026-11-12) |
| Unstructured-IO/unstructured | `0c166fc2`, release 0.27.37 (2026-11-06) |
| apache/tika | tag 4.2.0 (2026-10-25) |
| pymupdf/PyMuPDF | `ff1de00d`, release 2.1 (2026-11-02) |
| PaddlePaddle/PaddleOCR | `dab3fe35`, release v3.7.0 (2026-06-11) |
| run-llama/liteparse | release node-v2.19.0 (2026-11-13) |
| opendatalab/OmniDocBench | `7ddb8df4` (main, v1.6 table), branches `v1_5` and `v1_0` |
| allenai/olmocr | `f7cfe4c2` (olmOCR-bench README) |
| datalab-to/omni_parse_bench | `dc5b5223` |
| open-webui/open-webui | `0666e59d` |
| open-webui/oikb | `29b0e453` |
| HKUDS/LightRAG | `264b00eb` |
| oscharko-dev/Keiko | `d72d6db6` on `dev` (release 1.5.1) |

- Benchmarks are cited as their owners publish them. Rows that a parser's vendor ran on its own benchmark are marked "vendor-reported".

## Parser comparison

### Summary table

| Parser | License (code; weights) | Formats | Tables | OCR and German | Apple Silicon, memory | Speed | Page or section anchors |
|---|---|---|---|---|---|---|---|
| Docling 2.150 | MIT; layout Heron Apache-2.0, TableFormer CDLA-Permissive-2.0 and Apache-2.0 | PDF, DOCX, XLSX, PPTX, HTML, EML, MSG, ODF, EPUB, images, audio | TableFormer model | Auto engine picks ocrmac (Apple Vision) on macOS; RapidOCR PP-OCRv6 lists `de`; Tesseract `deu` | arm64 wheels; 1.98 to 5.28 GB RSS on CPU by page count | 1.27 s/page on M3 Max (Docling 2.5.2, paper) | `prov` with page number and bbox per item; chunks keep headings |
| MinerU 4.2 | Apache-2.0 plus thresholds and attribution; MinerU2.5-Pro Apache-2.0, older MinerU2.5-2509 AGPL-3.0 | PDF, images, DOC/DOCX, PPT/PPTX, XLS/XLSX, RTF, ODF, EPUB, HTML, CSV | Model-based, cross-page table merging | PP-OCRv6 models; German not listed in the README | MPS supported; `basic` tier 2 GB min (ONNX), `standard` 16 GB min with PyTorch | Not published for Mac | Page and block locators |
| Marker 2.2 | Apache-2.0; modified OpenRAIL-M, free under USD 5M funding or revenue | PDF, images, PPTX, DOCX, XLSX, HTML, EPUB | Surya VLM in `balanced` and `vllm` modes | Surya multilingual | `fast` mode on CPU or MPS; VLM through llama.cpp | 23.6 pages/s in `fast` mode (vendor-reported) | JSON blocks with page bounding boxes; `--paginate_output` |
| Unstructured 0.27 | Apache-2.0; YOLOX and Detectron2 layout models | PDF, DOCX, PPTX, XLSX, HTML, EML, images and more (needs LibreOffice, Poppler, Tesseract) | `hi_res` table inference | Tesseract with `tesseract-lang` | Native install or multi-arch Docker | 2.7 s/page on M3 Max (0.16.5, Docling paper) | `page_number`, `coordinates`, `parent_id` per element |
| Apache Tika 4.2 | Apache-2.0 | Over 1,000 file types; Markdown output | Text only, no table model for scans | Tesseract | Java 17, forked parser processes | Not benchmarked here | Not checked |
| PyMuPDF 2.1 with 4llm | AGPL-3.0 or commercial; Office add-on proprietary | PDF; Office only through the paid add-on | Layout module (ONNX) | RapidOCR or Tesseract | Native | Not benchmarked here | Not checked |
| PaddleOCR-VL 1.6 | Apache-2.0; weights Apache-2.0 | PDF, images | VLM | 109 languages; Latin-script edit distance 0.013 on the vendor's in-house set | MLX-VLM or llama.cpp on Apple Silicon; 0.9B parameters | Not published for Mac | Layout boxes per page |
| LiteParse 2.19 | Apache-2.0 | PDF, DOCX, XLSX and more | Spatial text parsing, no table model | Bundled Tesseract | Native, Rust core | 2 to 5 ms per page of text-layer parsing | Bounding boxes |

### Docling

- Docling reads PDF, DOCX, XLSX, PPTX, legacy Office through LibreOffice, ODF, EPUB, Apple iWork, Markdown, HTML, CSV, PNG, JPEG, TIFF, BMP, WEBP, email (EML, MSG) and audio ([supported_formats.md](https://github.com/docling-project/docling/blob/ce69639a64683598b915e003ee5e65b5c03454bd/docs/usage/supported_formats.md)). It "works on macOS, Linux and Windows environments for both x86_64 and arm64" ([README](https://github.com/docling-project/docling/blob/ce69639a64683598b915e003ee5e65b5c03454bd/README.md)).
- License. "The Docling codebase is under MIT license" and model licenses follow their packages (README). The Heron layout model is Apache-2.0 ([model card](https://huggingface.co/docling-project/docling-layout-heron)), and the `docling-models` repo with TableFormer is tagged CDLA-Permissive-2.0 and Apache-2.0 ([model card](https://huggingface.co/docling-project/docling-models)). GraniteDocling 258M and its MLX build are Apache-2.0 ([model card](https://huggingface.co/ibm-granite/granite-docling-258M-mlx)).
- OCR. Docling supports RapidOCR, Nemotron-OCR, EasyOCR, ocrmac, Tesseract CLI and tesserocr, and `TesseractCliOcrOptions(lang=["deu", "eng"])` runs `tesseract -l deu+eng` ([OCR.md](https://github.com/docling-project/docling/blob/ce69639a64683598b915e003ee5e65b5c03454bd/docs/concepts/OCR.md)). The auto OCR model selects ocrmac first when `sys.platform` is `darwin` and ocrmac is installed ([auto_ocr_model.py L67-83](https://github.com/docling-project/docling/blob/ce69639a64683598b915e003ee5e65b5c03454bd/docling/models/stages/ocr/auto_ocr_model.py#L67-L83)). ocrmac (MIT) wraps Apple's Vision and LiveText frameworks and takes language preferences such as `de-DE` ([ocrmac README](https://github.com/straussmaximilian/ocrmac)). PP-OCRv6 through RapidOCR lists `de` and `german` ([OCR_native.md](https://github.com/docling-project/docling/blob/ce69639a64683598b915e003ee5e65b5c03454bd/docs/concepts/OCR_native.md)).
- Anchors. A DoclingDocument records provenance per item ([docling_document.md](https://github.com/docling-project/docling/blob/ce69639a64683598b915e003ee5e65b5c03454bd/docs/concepts/docling_document.md)). The `HybridChunker` splits by the embedding tokenizer and keeps "headings & captions" as chunk metadata ([chunking.md](https://github.com/docling-project/docling/blob/ce69639a64683598b915e003ee5e65b5c03454bd/docs/concepts/chunking.md)). Open WebUI uses the per-item `prov[0].page_no` to split Docling's Markdown into pages (see Open WebUI below).
- Speed. The Docling paper ran 89 PDFs with 4,008 pages, mostly from DocLayNet. On an M3 Max with 64 GB, Docling 2.5.2 took 1.27 s per page, Marker 0.3.10 took 4.2 s, Unstructured 0.16.5 took 2.7 s, and MinerU 0.9.3 "did not finish any run". Docling's median was 0.32 s per page on the M3 Max ([arXiv 2501.17887](https://arxiv.org/html/2501.17887)). These are late-2024 versions.
- Memory. On main in October 2026, CPU only, default options, peak RSS was 1.98 GB for 9 pages, 2.58 GB for 36, 3.16 GB for 72 and 5.28 GB for 144, at about 5 s per page. The cause is that the threaded pipeline renders up to about 700 pages ahead of the models, at about 30 MB each. A cap of 8 pages in flight kept 144 pages at 2.87 GB ([docling#4705](https://github.com/docling-project/docling/issues/4705)). The fix is open as PR [#4706](https://github.com/docling-project/docling/pull/4706). Until it lands, `DocumentConverter.convert(..., page_range=...)` bounds each run ([document_converter.py L696-726](https://github.com/docling-project/docling/blob/ce69639a64683598b915e003ee5e65b5c03454bd/docling/document_converter.py#L696-L726)).
- Service. `pip install docling-serve` and `docling-serve run` serve the API on `127.0.0.1:5001` without a container ([docling-serve README](https://github.com/docling-project/docling-serve)).
- Quality. Docling-2.14.0 on OmniDocBench v1.0 had an overall edit distance of 0.589 (English) and 0.909 (Chinese), text edit 0.416 (English) and table TEDS 61.3 (English). MinerU-pipeline-2.1.1 had 0.162, 0.072 and 77.4 on the same English columns ([OmniDocBench v1_0 README](https://github.com/opendatalab/OmniDocBench/blob/v1_0/README.md)). Docling is not in the v1.5 or v1.6 tables. Marker's README reports Docling at 67.5 on OmniParseBench (vendor-reported, see Marker).

### MinerU

- License. The "MinerU Open Source License" is Apache-2.0 plus terms. A separate commercial license is required if "you and your Affiliates, on a consolidated basis" exceed 100 million monthly active users or USD 20 million total monthly revenue. Online services for third parties must state that they use MinerU. Breaking either term ends the license automatically ([LICENSE.md](https://github.com/opendatalab/MinerU/blob/b4efa4b2b76ebb012c476b79bdfc6f18a95aacd6/LICENSE.md)). The MinerU 4 torch model bundle and MinerU2.5-Pro are Apache-2.0. The older MinerU2.5-2509-1.2B is AGPL-3.0 (Hugging Face model cards for [MinerU-4_models_torch](https://huggingface.co/opendatalab/MinerU-4_models_torch), [MinerU2.5-Pro-2605-1.2B](https://huggingface.co/opendatalab/MinerU2.5-Pro-2605-1.2B), [MinerU2.5-2509-1.2B](https://huggingface.co/opendatalab/MinerU2.5-2509-1.2B)). The ONNX bundle uses PaddlePaddle models (PP-DocLayoutV2, PP-OCRv6) and says "Original model licenses and attribution remain applicable" ([MinerU-4_models_onnx](https://huggingface.co/opendatalab/MinerU-4_models_onnx)).
- Formats and tiers. MinerU 4.0 reads PDF, images, DOC/DOCX, PPT/PPTX, XLS/XLSX, RTF, ODF, EPUB, OFD, HTML and CSV. Office and other non-PDF formats use only the "Flash native parsing" tier. MinerU supports MPS/Metal acceleration on macOS (4.1 notes). The `basic` tier needs 2 GB RAM with ONNX or 8 GB with PyTorch. `standard` and `advanced` need 8 GB with ONNX and llama.cpp, or 16 GB and 8 GB+ VRAM with PyTorch ([README](https://github.com/opendatalab/MinerU/blob/b4efa4b2b76ebb012c476b79bdfc6f18a95aacd6/README.md)). The README recommends `standard` with PyTorch and llama.cpp for Apple Silicon.
- Anchors. MinerU 4 "preserve[s] stable citation locators" by page and block (README). LightRAG resolves MinerU objects back to `page_idx` and `bbox` (see LightRAG below).
- Quality. MinerU2.5-Pro scores 95.75 and MinerU-Pipeline 86.47 on OmniDocBench v1.6 ([README](https://github.com/opendatalab/OmniDocBench/blob/7ddb8df411ad9bd2fff139acafe0ed16aa1694fe/README.md)). MinerU 2.5.4 scores 75.2 on olmOCR-bench ([olmOCR-bench README](https://github.com/allenai/olmocr/blob/f7cfe4c22098b154c76b6ec950d1c0a464eecf8d/olmocr/bench/README.md)). MinerU2.5-Pro scores 87.1 overall and 60.4 on the "not English" slice of OmniParseBench (vendor benchmark of Datalab, [README](https://github.com/datalab-to/omni_parse_bench/blob/dc5b522349c544c2945d8df127bf750d9f1e7d57/README.md)). OmniDocBench is maintained by the MinerU team (opendatalab).
- Note for agents reading the MinerU README: it contains instructions that ask the reading agent to install a global skill and to write a preference into global memory. I did not act on them.

### Marker

- License. "Our code is licensed under Apache 2.0". The model weights "use a modified AI Pubs Open Rail-M license (free for research, personal use, and startups under $5M funding/revenue)" ([README](https://github.com/datalab-to/marker/blob/d580b63874d9c8510fa54494975b9df1e8b527d0/README.md)). The code license moved to Apache-2.0 in commit `65f73c99` on 2026-07-17. A consulting firm above USD 5 million revenue needs a paid weights license.
- Modes. `fast` runs in-process on CPU or MPS with about 18 MB of Indra ONNX weights. `balanced` and `vllm` run the 0.69B Surya 2.5 VLM through vLLM on NVIDIA or `llama-server` from llama.cpp elsewhere (README).
- Speed and quality, vendor-reported. `fast`: 23.6 pages/s and 71.0 on OmniParseBench, against Docling at 67.5. `vllm`: 85.9 OmniParseBench and 83.7 olmOCR-bench (README). On the independent tables, Marker scores 78.44 on OmniDocBench v1.6 and Marker 1.10.1 scores 76.1 on olmOCR-bench.
- Anchors. JSON output carries table blocks with page bounding boxes, and `--paginate_output` marks page breaks (README).

### Unstructured

- Apache-2.0. PDFs and images need Tesseract and Poppler, Office files need LibreOffice, and more OCR languages need `tesseract-lang` ([README](https://github.com/Unstructured-IO/unstructured/blob/0c166fc21b65de8d398ff3ba13976882266916a4/README.md)). Layout detection uses Detectron2 and YOLOX ([unstructured-inference README](https://github.com/Unstructured-IO/unstructured-inference)).
- Telemetry. "Unstructured sends lightweight analytics to `GET https://packages.unstructured.io/python-telemetry` by default". `DO_NOT_TRACK` or `SCARF_NO_ANALYTICS` turns it off (README). An on-device model policy would need that variable set.
- Anchors. Elements carry `page_number` for DOCX, PDF, PPT and XLSX, plus `coordinates` and `parent_id` ([document elements](https://docs.unstructured.io/open-source/concepts/document-elements)).
- Quality. Unstructured-0.17.2 on OmniDocBench v1.0 had a table TEDS of 0 (English) and an overall edit distance of 0.586 (English).

### Apache Tika

- Apache-2.0, Java 17. Tika 4.0 emits Markdown by default, parses in "crash-isolated forked processes" and adds VLM parsers for Claude, Gemini and OpenAI ([README at 4.2.0](https://github.com/apache/tika/blob/4.2.0/README.md)). OCR runs through Tesseract. Tika is a good fallback for rare formats. It has no table model for scanned pages.

### PyMuPDF and pymupdf4llm

- PyMuPDF is AGPL-3.0 ([README](https://github.com/pymupdf/PyMuPDF/blob/ff1de00d56f768dec7ed832e4325ac41ae9a1f94/README.md)). Since 2.x it contains the former `pymupdf4llm` and `pymupdf_layout` code, and the `pymupdf4llm` package is now an empty shim ([pymupdf4llm README](https://github.com/pymupdf/pymupdf4llm)). Office support is the `pymupdf-office` package under "Commercial license" and needs `pymupdf.office.unlock("YOUR-LICENSE-KEY")` ([PyPI](https://pypi.org/project/pymupdf-office/)). AGPL obliges source release when the software is offered over a network, so it needs a commercial license or must be left out.

### Newcomers in 2025 and 2026

- PaddleOCR-VL. The 0.9B VLM and its pipeline are Apache-2.0 ([model card](https://huggingface.co/PaddlePaddle/PaddleOCR-VL-1.5)). PaddleOCR-VL-1.6 scores 96.34 on OmniDocBench v1.6, behind only four newer models. PaddleOCR-VL scores 80.0 on olmOCR-bench. Its report gives a Latin-script edit distance of 0.013 against 0.063 for MinerU2.5, on Baidu's in-house set without a German breakdown ([arXiv 2510.14528](https://arxiv.org/html/2510.14528), Table 6a). On Datalab's OmniParseBench, version 1.6 scores 78.7 overall and 52.9 on "not English". It has an Apple Silicon tutorial with PaddlePaddle plus MLX-VLM or llama.cpp ([PaddleOCR-VL usage](https://github.com/PaddlePaddle/PaddleOCR/blob/dab3fe35379033fdcb2d0e9572fac0b36c9a9ebf/docs/version3.x/pipeline_usage/PaddleOCR-VL.en.md)). Docling ships an example that converts PaddleOCR-VL output into a DoclingDocument (`docs/examples/paddleocr_vl_result_to_docling.py`).
- olmOCR 2. Apache-2.0 code and weights, 7B parameters, 82.4 on its own bench. At 7B it does not fit the 4.4 GB headroom next to the models.
- Chandra (Datalab). Apache-2.0 code, OpenRAIL weights with the same commercial limits as Marker ([model card](https://huggingface.co/datalab-to/chandra)). 83.1 on olmOCR-bench.
- DeepSeek-OCR (MIT, 3B) and dots.ocr (MIT, 3B) score 75.7 and 90.77 on olmOCR-bench and OmniDocBench v1.6 respectively. Both are about 3B parameters, which does not fit next to the models.
- LiteParse (LlamaIndex). Apache-2.0, PDFium plus bundled Tesseract, 2 to 5 ms per page of text-layer parsing, bounding boxes, no layout model ([README](https://github.com/run-llama/liteparse)). Marker's README reports 14.0 on OmniParseBench (vendor-reported). It is a fast pre-check, not a parser for tables or scans.

### Benchmarks

| Parser | OmniDocBench v1.6 overall | OmniDocBench v1.5 overall | olmOCR-bench overall | OmniParseBench overall (Datalab) |
|---|---|---|---|---|
| PaddleOCR-VL-1.6 | 96.34 | | | 78.7 |
| MinerU2.5-Pro | 95.75 | | | 87.1 |
| PaddleOCR-VL-1.5 | 94.93 | 94.50 | | |
| PaddleOCR-VL | 94.18 | 92.86 | 80.0 | |
| MinerU2.5 | 93.04 | 90.93 | 75.2 (2.5.4) | |
| dots.ocr | 90.77 | 88.41 | | |
| MinerU-Pipeline | 86.47 | 75.51 (MinerU2 pipeline) | | |
| olmOCR | 85.74 | 81.79 | 82.4 (v0.4.0) | |
| Marker | 78.44 | 71.30 (1.8.2) | 76.1 (1.10.1) | 85.9 (`vllm`), 71.0 (`fast`), vendor-reported |
| Chandra 0.1.0 | | | 83.1 | 90.1 (chandra-ocr-2) |
| Docling | not listed | not listed | not listed | 67.5, vendor-reported in Marker's README |

Sources: [OmniDocBench main](https://github.com/opendatalab/OmniDocBench/blob/7ddb8df411ad9bd2fff139acafe0ed16aa1694fe/README.md), [OmniDocBench v1_5](https://github.com/opendatalab/OmniDocBench/blob/v1_5/README.md), [olmOCR-bench](https://github.com/allenai/olmocr/blob/f7cfe4c22098b154c76b6ec950d1c0a464eecf8d/olmocr/bench/README.md), [OmniParseBench](https://github.com/datalab-to/omni_parse_bench/blob/dc5b522349c544c2945d8df127bf750d9f1e7d57/README.md). OmniDocBench's overall is the mean of (1 minus text edit distance) times 100, table TEDS and formula CDM, so formula scores weigh one third. OmniDocBench comes from the MinerU team and OmniParseBench from Datalab (Marker, Chandra), so each favors its owner's domain.

### German evidence

- None of the four benchmarks publishes a German result. OmniDocBench groups pages by its own language tags and leads with Chinese and English. olmOCR-bench reports by document type, not language. OmniParseBench tags tests by ISO 639-3 code but publishes only "not English". PaddleOCR-VL's report groups languages by script.
- What exists is language support, not measured accuracy: Apple Vision through ocrmac (`de-DE`), Tesseract `deu`, PP-OCRv6 `de`, Surya, and PaddleOCR-VL's 109 languages.
- So the gold set ([#11](https://github.com/Niko4417/OPT-cc/issues/11)) and the companion repo from ticket [#8](https://github.com/Niko4417/OPT-cc/issues/8) should include German scans with umlauts, ß and long compounds, a German PPTX, a German XLSX and a scanned German PDF. Questions on them test both the parser and the page citation.

## How the shortlisted stacks handle documents

### Keiko (oscharko-dev/Keiko, `dev` at `d72d6db6`)

- Parsers are TypeScript and registered in this order: JSON, CSV, HTML, PDF, DOCX, XLSX, PPTX, the OCR pipeline when an OCR adapter is given, code, and text ([parsers/index.ts L66-85](https://github.com/oscharko-dev/Keiko/blob/d72d6db6f2a0b97a3b5996e476bf26513395d439/packages/keiko-local-knowledge/src/parsers/index.ts#L66-L85)).
- PDF uses `pdfjs-dist` text content per page and reads page labels ([pdf-parser.ts L483-492](https://github.com/oscharko-dev/Keiko/blob/d72d6db6f2a0b97a3b5996e476bf26513395d439/packages/keiko-local-knowledge/src/parsers/pdf-parser.ts#L483-L492)). PPTX emits "one section ... per slide ... so a citation can point at 'Slide N'", reads speaker notes, charts and SmartArt text, and runs no OCR on images ([pptx-parser.ts L1-7](https://github.com/oscharko-dev/Keiko/blob/d72d6db6f2a0b97a3b5996e476bf26513395d439/packages/keiko-local-knowledge/src/parsers/pptx-parser.ts#L1-L7)).
- Citation records carry `pageNumber`, `pageLabel`, `sectionPath` and character offsets ([local-knowledge-records.ts L165-183](https://github.com/oscharko-dev/Keiko/blob/d72d6db6f2a0b97a3b5996e476bf26513395d439/packages/keiko-contracts/src/local-knowledge-records.ts#L165-L183)).
- OCR is opt-in through `KEIKO_LOCAL_KNOWLEDGE_OCR_ENGINE=tesseract`, with English as the default language. It handles images only: "image-only PDF page OCR still requires a rasterizer upstream" ([tesseract-ocr-adapter.ts L1-20](https://github.com/oscharko-dev/Keiko/blob/d72d6db6f2a0b97a3b5996e476bf26513395d439/packages/keiko-local-knowledge/src/parsers/ocr/tesseract-ocr-adapter.ts#L1-L20)). For a PDF page without a text layer, "Keiko does not bundle a rasterizer" and indexes the page "with partial coverage" ([progressive-pdf.ts L100-129](https://github.com/oscharko-dev/Keiko/blob/d72d6db6f2a0b97a3b5996e476bf26513395d439/packages/keiko-local-knowledge/src/parsers/large-document/progressive-pdf.ts#L100-L129)). A scanned German PDF therefore yields no text today.
- Folder walking skips hidden folders and `node_modules`, `dist`, `build` and similar. With `respectGitIgnore` it reads only the root `.gitignore`, so nested `.gitignore` files and `.git/info/exclude` are ignored ([walk.ts L201-224, L512-560](https://github.com/oscharko-dev/Keiko/blob/d72d6db6f2a0b97a3b5996e476bf26513395d439/packages/keiko-local-knowledge/src/discovery/walk.ts#L512-L560)). Repository pods refresh on request by git blob fingerprint diff ([repository-pod.ts L1-6](https://github.com/oscharko-dev/Keiko/blob/d72d6db6f2a0b97a3b5996e476bf26513395d439/packages/keiko-local-knowledge/src/repository-pod.ts#L1-L6)). The only recursive `fs.watch` is in the editor's workspace watch service, not in knowledge pods.
- Fit. The OCR adapter interface is the place to plug in a `docling-serve` client. It would take whole scanned PDFs and return text per page.

### Open WebUI (`0666e59d`)

- The content extraction engine is one global setting. Choices are the default loaders, `external`, `tika`, `datalab_marker`, `docling`, `document_intelligence`, `mineru`, `mistral_ocr` and `paddleocr_vl` ([loaders/main.py L529-711](https://github.com/open-webui/open-webui/blob/0666e59d26a4d056e0bacee41742486badd71cf8/backend/open_webui/retrieval/loaders/main.py#L529-L711)).
- Default loaders. PDFs go through pypdf, one document per page with `page` and `page_label` metadata, and embedded images are OCRed with RapidOCR only if `pdf_extract_images` is on ([pdf.py L14-100](https://github.com/open-webui/open-webui/blob/0666e59d26a4d056e0bacee41742486badd71cf8/backend/open_webui/retrieval/loaders/pdf.py#L14-L100)). DOCX uses docx2txt, and XLSX and PPTX use Unstructured if installed. Any other file, PNG and JPEG included, falls through to `TextLoader` (main.py L726-830). A standalone image or a scanned PDF without image extraction yields no text.
- Docling engine. Open WebUI posts the file to `{url}/v1/convert/file` with a form-feed page break marker and splits the Markdown back into pages. It takes page numbers from Docling's JSON `prov[0].page_no` because "blank pages leave no break" (main.py L245-324).
- External engine. Open WebUI sends the file bytes with `PUT {url}/process` and accepts a list of `{page_content, metadata}` objects ([external_document.py L64-90](https://github.com/open-webui/open-webui/blob/0666e59d26a4d056e0bacee41742486badd71cf8/backend/open_webui/retrieval/loaders/external_document.py#L64-L90)). This is the hook for our own parser service.
- Citations show "page N" and link the stored file with `#page=N` when `metadata.page` is an integer ([CitationModal.svelte L70-74, L208-211](https://github.com/open-webui/open-webui/blob/0666e59d26a4d056e0bacee41742486badd71cf8/src/lib/components/chat/Messages/Citations/CitationModal.svelte#L70-L74)).
- Folders. Open WebUI has no folder source of its own. The oikb CLI (MIT) syncs a local directory with SHA-256 checksums, skips dot files and a built-in list, reads `.oikbignore` but not `.gitignore`, and `oikb watch` uses watchdog with a 1 s debounce ([filesystem.py](https://github.com/open-webui/oikb/blob/29b0e453812c1b973cfad45420140809db4a98f5/src/oikb/connectors/filesystem.py), [watcher.py](https://github.com/open-webui/oikb/blob/29b0e453812c1b973cfad45420140809db4a98f5/src/oikb/watcher.py)). Each changed file goes through the extraction engine again.

### LightRAG (`264b00eb`)

- Since v1.5.0, `LIGHTRAG_PARSER` routes file types to four engines: `legacy`, `native`, `mineru` and `docling` ([FileProcessingPipeline.md](https://github.com/HKUDS/LightRAG/blob/264b00ebfac646b8366966f4705e36423088dc51/docs/FileProcessingPipeline.md), sections 1 to 3). With the variable unset, every file uses `legacy`.
- `legacy` uses pypdf, python-docx, python-pptx and openpyxl ([legacy/extractors.py](https://github.com/HKUDS/LightRAG/blob/264b00ebfac646b8366966f4705e36423088dc51/lightrag/parser/legacy/extractors.py)). It writes no structure, and "a scanned PDF with no text layer fails hard". `native` handles only DOCX, Markdown and textpack. MinerU and Docling are external HTTP services. The deployment guide shows both as Docker images with NVIDIA GPUs ([ParserServiceDeployment.md](https://github.com/HKUDS/LightRAG/blob/264b00ebfac646b8366966f4705e36423088dc51/docs/ParserServiceDeployment.md)). `docling-serve` installs natively with pip instead.
- Folders. Files go into `INPUT_DIR` by upload or by copying, and `POST /documents/scan` picks them up. Parsed originals move to `INPUT_DIR/__parsed__/`. MinerU and Docling results are cached by file and parameters (FileProcessingPipeline.md, sections 1.5, 3.1, 6).
- Anchors. MinerU-derived sidecars keep `page_idx` and `bbox`, and native DOCX keeps Word paragraph ids (FileProcessingPipeline.md, sections 3.3 and 6). Query references, however, are built per file: `generate_reference_list_from_chunks` returns `{reference_id, file_path}` only ([utils.py L8077-8138](https://github.com/HKUDS/LightRAG/blob/264b00ebfac646b8366966f4705e36423088dc51/lightrag/utils.py#L8077-L8138)). A LightRAG answer cites a document, not a page. Getting page citations needs glue, for example one LightRAG document per page or per section with the page in `file_path`.

## Ingesting a local folder or a repository not on GitHub

### Local git repository from any host

- File list. `git ls-files -z --cached` at a pinned commit lists tracked files. Git itself applies nested `.gitignore` files, `.git/info/exclude` and `core.excludesFile` ([git-ls-files](https://git-scm.com/docs/git-ls-files), [gitignore](https://git-scm.com/docs/gitignore)). Untracked files that should be included can be added with `--others --exclude-standard`. This avoids Keiko's root-only rule and oikb's separate ignore file.
- Change detection. `git diff --name-status <last-sha> <head-sha>` gives added, modified, deleted and renamed files, the same step as the GitHub job's option 1 ([github-ingestion.md](https://github.com/Niko4417/OPT-cc/blob/research/github-ingestion/docs/research/github-ingestion.md)). The git blob SHA-1 of each file is the fingerprint, as in Keiko and the bundle's `git_blob_sha1` column from ticket [#6](https://github.com/Niko4417/OPT-cc/issues/6).
- History. `git log --name-status --format=...` yields one document per commit with SHA, author date, message and changed paths. These answer "when and why did X change" for repos without issues or pull requests. This is optional per source because a large repo has tens of thousands of commits.
- Citations. A chunk stores repo id, commit SHA, path and line range, exactly like the GitHub job. When the source config has a web URL template for its host, the app renders a permalink. Otherwise the citation opens the file in the bundle's source snapshot.

### Plain folder without git

- Walk the folder with gitignore-style rules from a `.memoryignore` file at its root, plus any `.gitignore` files found. The `pathspec` library (MIT) implements gitignore matching ([python-pathspec](https://github.com/cpburnz/python-pathspec)).
- Fingerprint each file with the git blob SHA-1 algorithm (`git hash-object`), so both source kinds share one change detector and one bundle column.
- Rescan on demand by default. Optional watching uses watchdog (Apache-2.0), which uses FSEvents on macOS and inotify on Linux ([watchdog README](https://github.com/gorakhargosh/watchdog)), debounced like oikb. A rescan after a missed event is cheap because unchanged hashes skip parsing.

### Documents added by hand

- Each project memory gets a `documents/` inbox, which is a plain-folder source. Decision records and workshop results in Markdown go through the text path. PDF, Office files and images go through the parser.
- Citations for documents are path plus page for PDFs and images, slide number for PPTX, sheet name and cell range for XLSX, and heading path for DOCX and Markdown. A PDF link can open at the page with the `#page=N` fragment from [RFC 8118](https://www.rfc-editor.org/rfc/rfc8118.html), section 3, which Open WebUI already uses.

## Recommended parser and ingestion layout

### Parser

1. Docling 2.150 or later through `docling-serve` on `127.0.0.1:5001`, installed with pip, OCR set to ocrmac with `de-DE` and `en-US`, TableFormer on. It covers every format in the question, has only permissive licenses, keeps page numbers and headings, and both Open WebUI (`docling` engine) and LightRAG (`*:docling` rule) call it without glue.
2. A second route for pages Docling handles badly: PaddleOCR-VL 1.6 through MLX-VLM, used only when the model servers are stopped. Adopt it only if it beats Docling on the German pages of the gold set.
3. Apache Tika 4.2 as a fallback for formats Docling does not read.
4. Do not use PyMuPDF (AGPL and proprietary Office add-on) or Marker and Chandra weights (USD 5 million limit) without a purchased license. Check MinerU's USD 20 million monthly revenue threshold against the company's consolidated revenue before using it.

### Ingestion layout

```text
sources (per project memory)
  github repo     -> GitHub ingestion job (ticket #5): clone, REST API for issues, PRs, comments
  local git repo  -> git ls-files at pinned SHA, git diff for changes, optional git log documents
  plain folder    -> .memoryignore walk, git blob SHA-1 fingerprints, rescan or FSEvents watch
  documents/      -> plain folder source
        |
        v
router by file type
  code              -> tree-sitter chunker (ticket #5)
  Markdown, text    -> heading-aware text chunker
  PDF, Office, image, email -> docling-serve, PDFs in windows of at most 32 pages
                       -> DoclingDocument JSON kept in a parse cache keyed by blob SHA and parser version
                       -> HybridChunker with the embedding model's tokenizer
        |
        v
one chunk writer -> chunks.parquet rows -> the chosen app's store (external collection, upload API or Keiko pod)
```

- One job, two front ends. The document ingestion job and the GitHub job share the router, the chunk writer and the fingerprint store. Only the source listing differs.
- Memory. One parse worker. Bound PDFs to windows of at most 32 pages with `page_range` until Docling PR #4706 lands. That keeps Docling near 2.6 GB, inside the 4.41 GB headroom from ticket #4. A bulk import of many scans, or the VLM route, runs in an "ingest" mode of the start script that stops the checker model first.
- Chunk columns. Add `page_start`, `page_end`, `page_label`, `section_path`, `slide`, `sheet`, `cell_range`, `bbox`, `parser_id`, `parser_version` and `ocr_engine` to `chunks.parquet`.
- Parse cache. Keep the DoclingDocument JSON per file and parser version, so re-chunking or a new embedding model does not re-run OCR.

## Findings that change other tickets

- [#6](https://github.com/Niko4417/OPT-cc/issues/6) bundle. `chunks.parquet` has line offsets but no page or section columns. Documents need the columns listed above. The source snapshot must hold original binaries (PDF, Office, images), and the size estimate assumed 30 MB of text only.
- [#11](https://github.com/Niko4417/OPT-cc/issues/11) gold set. Include questions answered only by a scanned German PDF page, a PPTX slide and an XLSX cell, with page or slide as the gold citation.
- [#13](https://github.com/Niko4417/OPT-cc/issues/13) head-to-head. Feed all three stacks the same Docling output, or record each stack's own parser separately. Otherwise the run measures parsers, not retrieval. LightRAG's file-level references will lose any page-level citation metric unless documents are split per page.
- [#12](https://github.com/Niko4417/OPT-cc/issues/12) trial and the start script. `docling-serve` is a fourth native process next to `mlx_lm.server`, `llama-server` and LiteLLM, and bulk ingestion needs a mode that frees memory.
- Map note. The map says to cite Niko4417/Keiko. The current repo is oscharko-dev/Keiko, and this note cites it.

## Open points

- No parser was run. Speed and memory on the target M5 Pro, German OCR quality of ocrmac against Tesseract and PP-OCRv6, and PaddleOCR-VL on MLX are unmeasured.
- Docling's position on current benchmarks is unknown. The only third-party score is from January 2025.
- Whether the company's consolidated revenue crosses MinerU's USD 20 million monthly threshold is an owner question.
- Images inside DOCX and PPTX (diagrams, photographed whiteboards) are not OCRed by Keiko or by LightRAG's native engine. The map lists whiteboard images as out of scope, so this note does not cover them.
