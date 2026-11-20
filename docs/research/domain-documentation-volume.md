# How much domain knowledge documentation a mature system carries

Research note for the test dataset of the project memory. It follows the [documentation graph research](https://github.com/Niko4417/OPT-cc/blob/research/documentation-graph/docs/research/documentation-graph.md), which found that the test dataset has no domain knowledge and no glossary. It does not repeat that note's evidence. All counts are my own and were taken on 2026-11-20 and 2026-11-21 unless a line gives another date or names another counter. "Words" always means the counting rule in the section "Method and limits".

## Question

How much domain knowledge documentation does a software system of 10 to 20 years carry, how is it spread over versions and ages, and how large should the domain knowledge part of the test dataset be to resemble that?

The owner rejected a guessed size of "8 documents and a glossary of 40 terms". Domain knowledge is the glossary's term for what the client's business does and how: business rules, processes, domain terms and functional specifications.

## Short answer

The guess was too small. Measured against the sizes recommended below, it is 4 to 7 times too small in documents for a small project memory and 40 to 90 times for a large one.

- Eleven public systems with a word count carry 0.05 to 2.9 million words of documentation in 15 to 5,644 files or pages. The median is 0.77 million words and 1,167 files or pages.
- Where the functional and user part can be separated, it holds 0.05 to 1.25 million words, with a median of 0.33 million words over seven systems. Eight documents of the median size of a public requirements document (6,774 words) are 54,000 words. That equals the smallest system measured, the whole LedgerSMB book.
- Old material stays. In four wikis, 13%, 49%, 57% and 66% of all pages were last changed before 2017. OpenMRS keeps 59% of its pages in a space named "Archives". Of 569 mail RFCs, 24% are obsoleted by a later RFC and still published.
- Several versions of one document are normal. Odoo keeps 34 version branches of its documentation, Nextcloud 27, Moodle one wiki per release. The XJustiz specification lists 23 versions since 2010 and OSCI-XMeld 33.
- Glossaries in open-source projects are small, 8 to about 100 terms. Glossaries that company experts picked for industrial requirements sets have 51, 140 and 200 terms. Organization-wide dictionaries have 143 (FIM) and 796 (NHS) definitions. Forty terms are below every industrial figure found.

Recommended size of the domain knowledge part. The derivation and the judgment calls are in the section "Recommended size".

| | Small project memory (`thunderbird-android`) | Large project memory (`shopware`) |
|---|---|---|
| Current domain documents | 30 to 60 | 300 to 700 |
| Words in current documents | 150,000 to 300,000 | 0.6 to 1.5 million |
| Glossary terms, each with a German and an English label | 100 to 200 | 400 to 800 |
| Superseded or outdated versions on top | 15 to 30 documents, with at least 3 chains of 3 or more versions | 100 to 300 documents, with at least 10 such chains |
| German share of the words | 60% to 80% of the synthetic documents | about 50% |

Words are the firmer figure. The document counts assume about 5,000 words per document for the small project memory and about 2,000 for the large one, and a text modeled as wiki pages needs 5 to 20 pages per document.

Real text can fill a large part of this for the mail domain. The IETF mail RFCs may be copied unmodified into a public repository. Mozilla's Thunderbird support articles are CC BY-SA 3.0 and may be copied with attribution and the same license. Shopware's own documentation carries no license that allows copying, so it can only be fetched at test time.

## Measured systems

I picked eleven public business systems and one mail client family. Eight have documents or a repository at least 10 years old by a date I measured: Odoo, Nextcloud, LedgerSMB, OFBiz, OpenEMR, OpenMRS, Dolibarr and Thunderbird for Android. The ERPNext code repository was created on GitHub on 2011-06-08, its archived manual in 2019. For Moodle, Shopware and iDempiere no date in this note shows the age of the project.

### Volume, split and glossary

Split rule. "Functional and user" is text that says what the business does or how a user carries out a business task in the product, by business area. "Developer and admin" is installation, hosting, API and development text. I split by the top-level folder or wiki space where the project separates the two, and by a keyword rule on page titles where it does not. The keyword rules are crude, and the table marks them as estimates. No project separates domain knowledge from user manuals, so I could not split those two.

| System | Where the documentation lives | Files or pages, words | Functional and user part | Developer and admin part | Glossary |
|---|---|---|---|---|---|
| Odoo | Separate repository [`odoo/documentation`](https://github.com/odoo/documentation), branch `20.0`, commit `4ab060e1` | 1,202 RST files, 1,528,555 words | `content/applications`: 963 files, 1,123,564 words | `developer`, `administration`, `contributing`: 199 files, 362,225 words. `legal`: 33 files, 41,666 words | Developer glossary only, 8 term lines |
| Nextcloud | Separate repository [`nextcloud/documentation`](https://github.com/nextcloud/documentation), commit `4ebe12f` | 546 files, 445,292 words | `user_manual`: 88 files, 50,734 words | `admin_manual`: 200 files, 203,507 words. `developer_manual`: 247 files, 185,395 words | Developer glossary, 25 terms, added 2026-05-26 |
| Shopware | Developer documentation in [`shopware/docs`](https://github.com/shopware/docs), commit `b0966a82`. User documentation on [docs.shopware.com](https://docs.shopware.com/en) | Developer: 1,053 Markdown files, 744,633 words. User: 344 English pages with 327,619 words and 343 German pages with 298,237 words | The user site. In the developer repository only `concepts/commerce`, 17 files, 10,368 words | The rest of the developer repository. The code repository adds 189 ADR files, 1,037,405 bytes | None found in either |
| LedgerSMB | Book repository [`ledgersmb/ledgersmb-book`](https://github.com/ledgersmb/ledgersmb-book), commit `d50b0d9` of 2026-10-28. ADRs in the code repository | 13 LaTeX files with 49,500 words, 50,641 words with the two Markdown files. 42 ADRs | The whole book. It does not separate the two | Parts "Administration" (6,214 words) and "Customization" (2,210 words) | 27 glossary entries and 40 acronyms |
| Apache OFBiz | AsciiDoc in the code repository [`apache/ofbiz-framework`](https://github.com/apache/ofbiz-framework), commit `0f1fc60`. Two Confluence spaces on [cwiki.apache.org](https://cwiki.apache.org/confluence/display/OFBIZ) | Repository: 213 AsciiDoc files, 108,694 words. Wiki: 1,056 pages, 684,043 words | Repository `applications`: 183 files, 86,543 words. Wiki space "End-User Documentation": 350 pages, 189,193 words | Repository `framework`, `docs`, `themes`: 30 files, 22,151 words. Wiki space "Project Open Wiki": 706 pages, 494,850 words, mixed | Five glossary files in the repository with 99 entries (accounting 58, human resources 21, common 14, party 5, order 1). Wiki glossaries with 82 and 36 table rows, and a German accounting glossary with 32 rows |
| OpenEMR | MediaWiki at [open-emr.org/wiki](https://www.open-emr.org/wiki/) | 1,167 content pages, 771,312 words | Estimate by title keywords: 267 pages, 164,979 words on clinical, billing and certification topics, and 40 pages, 51,653 words of user guides | Estimate by title keywords: 313 pages, 251,031 words. 547 pages, 303,649 words match no rule | None found |
| OpenMRS | Confluence at [openmrs.atlassian.net/wiki](https://openmrs.atlassian.net/wiki), four public spaces | 5,644 pages, 2,879,038 words | Not separable. Space "Documentation": 1,217 pages, 958,386 words | Space "Projects": 958 pages, 553,785 words. "Resources": 131 pages, 106,915 words. "Archives": 3,338 pages, 1,259,952 words | One page "OpenMRS Glossary", 1,923 words, 105 headings |
| Dolibarr | MediaWiki at [wiki.dolibarr.org](https://wiki.dolibarr.org), one wiki for all languages | 2,448 content pages, 1,588,349 words | Estimate by title keywords: 1,953 pages, 1,251,266 words. 1,038 pages have a title that starts with "Module", "Módulo" or "Modul" | Estimate by title keywords: 495 pages, 337,083 words | Three glossary pages, one per language, 135 to 325 words each |
| ERPNext | Archived repository [`frappe/erpnext_documentation`](https://github.com/frappe/erpnext_documentation), last commit `0d8e02e` of 2021-07-28. The current manual is on docs.frappe.io and was not measured | 2,254 Markdown files with 780,692 words in two version folders. Version 12: 1,126 files, 391,274 words. Version 13: 1,128 files, 389,418 words | All of it is a user manual by business module | None in this repository | None found |
| iDempiere | Documentation site repository [`idempiere/idempiere.github.io`](https://github.com/idempiere/idempiere.github.io), commit `89165f5`, started 2023. The older wiki refused automated reads | 535 Markdown files, 242,611 words | `docs/basic-functional`: 6 files, 4,657 words | Everything else. `docs/new-features` alone has 419 files, 132,363 words | Only an OSGi glossary |
| Moodle | One MediaWiki per release and language at [docs.moodle.org](https://docs.moodle.org/overview/) | Content pages per wiki: 2,952 (5.3 English), 2,917 (5.1 English), 2,821 (3.11 English), 2,185 (1.9 English), 1,815 (5.3 German). Words not measured, the site refuses scripted reads | Not measured | Developer documentation is a separate site | Not found. The page "Moodle key terms" defines 3 terms |
| Thunderbird and K-9 Mail | Markdown in [`thunderbird/thunderbird-android`](https://github.com/thunderbird/thunderbird-android), commit `2d88129`. Support articles on support.mozilla.org. Old K-9 manual in [`k9mail/k9mail-docs`](https://github.com/k9mail/k9mail-docs) | Repository: 128 Markdown files, 113,336 words. Support articles: 176 for Thunderbird with 149,219 words and 14 for Thunderbird for Android with 12,368 words, from an export of 2026-06-12. K-9 manual: 30 files, 89,577 bytes | The support articles and the K-9 manual | The repository Markdown | None |

### Languages, versions and age

| System | Languages, and German | Versions side by side | Age of the documents |
|---|---|---|---|
| Odoo | 26 translation folders. German has 54,342 of 54,462 strings translated, 1.05 million source words | 34 version branches: 12 major versions (`9.0` to `20.0`) and 22 `saas-` branches | First commit 2015-02-11, last 2026-11-20 |
| Nextcloud | User manual in 40 locales. German (`de_DE`) holds 3,402 translated strings with 50,140 words | 27 branches `stable9` to `stable35` | First commit 2012-10-27, last 2026-11-20, 28,553 commits |
| Shopware | User documentation in English and German, page for page (344 and 343). Developer documentation in English only | Developer repository: 8 version branches (6.3 to 6.8). The user site covers Shopware 6 only. A request for the Shopware 5 pages redirects to docs.safefive.de, a different site | Developer repository first commit 2020-11-24. User pages last changed 2021 to 2026, 226 of 344 in 2026 |
| LedgerSMB | English only | 7 branches (`1.3`, `1.4`, `1.10` to `1.13`, `master`) | First commit 2012-03-28, last 2026-10-28, 440 commits |
| Apache OFBiz | English. 3 German pages in the wiki, among them a German accounting glossary. 4 pages with translation dictionaries for Dutch, French, German and Danish | 5 code branches (`release17.12`, `18.12`, `22.01`, `24.09`, `trunk`), of which I counted the documentation on `trunk` only. The wiki holds four beginner tutorials for different release ranges side by side. 133 page titles carry a version number or a word such as "deprecated", "old" or "archive" | Wiki pages created 2006-05-23 to 2026-11-01. 692 of 1,056 pages (66%) were last changed before 2017. In the end-user space it is 282 of 350 (81%) |
| OpenEMR | English. 14 pages name another language in the title, 2 of them German with 110 and 120 words | 291 pages (25%) carry one of 40 release numbers from 2.9.1 to 8.5.0 in the title | Pages created 2009-11-07 to 2026-11-12. 664 of 1,167 pages (57%) were last changed before 2017 |
| OpenMRS | English in the four spaces measured. Country spaces for Kenya, Mozambique and Uganda were not measured | The "Archives" space holds 3,338 of 5,644 pages (59%) and 44% of the words. 449 page titles carry a version number or an age marker | Pages created 2010-05-07 to 2026-11-18. 2,788 of 5,644 pages (49%) were last changed before 2017 |
| Dolibarr | By a stop-word count per page: English 1,209 pages, French 706, Spanish 304, German 90 (4%, 97,264 words), Italian 29. 110 pages are too short to classify | 62 page titles carry a version number or an age marker | Pages created 2004-07-20 to 2026-11-20. 322 of 2,448 pages (13%) were last changed before 2017 |
| ERPNext | Version 12 manual: English 780 files with 326,190 words, German 226 files with 49,557 words, Spanish 53 files, Lithuanian 9 files. German is 29% of the English files and 15% of the English words | 2 complete copies, `v12` and `v13`, in one tree. Of the 1,126 files of `v12`, 176 are byte-identical in `v13`, 929 differ and 21 are gone | First commit 2019-01-15, last 2021-07-28, then archived |
| iDempiere | English | None in the tree | Repository created 2023-03-28 |
| Moodle | 14 languages on the overview page. English and German each have wikis for 5.3, 5.2, 5.1 and 4.5 | One wiki per release. I opened the wikis for 1.9, 3.11, 5.1 and 5.3 | 159,543 page edits in the English 5.3 wiki since MoodleDocs was set up |
| Thunderbird and K-9 Mail | Repository English. The support article export holds English only. A Mozilla wiki page names German among the languages of the support articles (search snippet). I could not count the German articles | 19 decision records in the repository (earlier note). The K-9 manual has branches `main`, `stable` and `beta`, and its README has called it outdated and abandoned since 2024-08-26 | Repository created 2011-02-04 |

### What the table shows

- Volume. Total documentation runs from 50,641 words (LedgerSMB, 15 files) to 2,879,038 words (OpenMRS, 5,644 pages). The eleven systems with a word count sort as 0.05, 0.24, 0.27, 0.39, 0.45, 0.77, 0.79, 1.37, 1.53, 1.59 and 2.88 million words, with a median of 0.77 million. Thunderbird counts as repository Markdown plus support articles, OFBiz as repository plus wiki, Shopware as developer repository plus both user languages.
- Functional and user part. The seven systems where it can be separated sort as 49,500 (LedgerSMB), 216,632 (OpenEMR, estimate), 275,736 (OFBiz), 326,190 (ERPNext, English), 327,619 (Shopware, English), 1,123,564 (Odoo) and 1,251,266 words (Dolibarr, four languages, estimate). The median is 326,190 words.
- Ratio of functional to developer text. It is 3.1 to 1 in Odoo and 3.9 to 1 in the OFBiz repository, and 0.13 to 1 in Nextcloud and 0.44 to 1 for Shopware's English user site against its developer repository. Business applications sit above 1, platforms below.
- Glossaries are the weakest part of open-source documentation. Four of the twelve systems have none that I could find, Moodle has a page with 3 terms, and three have only a developer glossary. The largest set, OFBiz, has 99 entries in the repository.
- German. Two vendors with German-speaking customers translate everything (Shopware page for page, Odoo 99.8% of strings). Community wikis have little German (Dolibarr 4% of pages, ERPNext 15% of the English words, OFBiz 3 pages).
- Size of one document. The median wiki page has 255 words (OFBiz), 254 (OpenMRS), 269 (OpenEMR) and 318 (Dolibarr). A Shopware user page has 952 words on average. Wiki pages are much shorter than specification documents, which the next section measures at a median of about 7,000 words.

## Published evidence

Two delegated literature passes collected the sources in this section and the next. I re-opened the numbers marked "checked" in the saved full text. The others come from one reading by the pass.

### How much documentation a system has

- PURE, a public dataset of 79 requirements documents ([Zenodo record](https://zenodo.org/records/1414117), Ferrari, Spagnolo and Gnesi, 2017). Counted from the files by the literature pass: 1,314 to 82,103 words per document, median 6,774, mean 10,992, quartiles 3,969 and 15,026, and 868,363 words in total. The 62 PDFs have 8 to 176 pages, median 33.
- Thirteen requirements specifications of financial information systems at one company hold 2,725 requirements and 115,585 words, 331 to 22,057 words each with a median of 7,539 ([Veizaga, Shin and Briand](https://arxiv.org/pdf/2305.07097), Table 2, academics with an industrial partner, checked).
- At NovAtel the document management system held "more than 60,000 documents and their revisions". One medium-size product came to "a few hundred" documents, and a sample of 55 design, test and process documents had 1,630 revisions, which is 30 per document ([Garousi et al., Information and Software Technology 57, 2015](https://pureadmin.qub.ac.uk/ws/portalfiles/portal/180209101/IST2013_Special_Issue_July30.pdf), section 3.4, independent academics, checked).
- One automotive project at Bosch has more than 10,000 requirements ([Langenfeld et al., REFSQ 2016](https://swt.informatik.uni-freiburg.de/staff/langenfeld/resources/Requirements%20Defects%20over%20a%20Project%20Lifetime), checked).
- The VA Software Document Library for VistA, the hospital system of the US Department of Veterans Affairs, lists 398 application entries, 250 of them with documents and 3,931 documents in total. An application with documents has a median of 5 and a mean of 15.7 documents, quartiles 3 and 14, and a maximum of 314. Dates run from 1993-05-01 to 2026-11-18 ([va.gov/vdl](https://www.va.gov/vdl/), counted by the literature pass from the live site). I drew a random sample of 60 of these documents, weighted by documents per application, and counted the PDF text. A document has a mean of 8,378 words (standard error 1,396), a median of 4,133 words and a mean of 39.5 pages. 31 of the 60 are release notes or installation guides, 14 are user manuals or guides with 38% of the words, and 11 are technical manuals. Scaled up, the library holds about 33 million words, which is an estimate from this sample and not a count.
- Two synthetic enterprise benchmarks show what their authors took as realistic. EnterpriseRAG-Bench has 5,189 Confluence pages among 511,962 documents, with a mean Confluence page length of 9,852 characters ([Sun et al.](https://arxiv.org/html/2605.05253), Tables 1 and 9, vendor Onyx, checked). HERB has 400 documents among 39,190 artifacts for 30 products ([Salesforce AI Research](https://arxiv.org/html/2506.23139), checked). These are design choices, not measurements.

I found no study that counts the pages or words of a company wiki or a SharePoint site per project.

### How much of it is outdated or contradictory

- In the 1,000 most popular GitHub projects, 19.2% of documentation files (1,880 of 9,784) hold at least one reference to a code element that no longer exists, and 28.9% of projects hold such a file. Over the history of 800 projects, 40.7% of documents and 82.3% of projects did at some point. A reference that is outdated today has been outdated for 4.7 years on average ([Tan, Wagner and Treude](https://arxiv.org/pdf/2212.01479), sections 5.1 and 5.2, authors of the detector, checked).
- Of 878 documentation issues mined from mailing lists, issue trackers and Stack Overflow, 190 are about up-to-dateness, which is 39% of the issues on content ([Aghajani et al., ICSE 2019](https://emadpres.github.io/pdfs/icse2019.pdf), independent academics).
- In a survey of 48 engineers, the most common answer on how fast documents are updated after a change was "rarely" for requirements (52%), specifications (46%) and detailed design (42%) ([Forward and Lethbridge, DocEng 2002](https://www.eecs.uottawa.ca/~tcl/gradtheses/aforward/papers/aforwarddoceng2002sub.pdf), Table 2, checked). The sample is small and 24 years old.
- In 3.3 million commits of 1,500 Java projects, a code change comes with a comment update in 7% of cases for method comments and 13% for class comments ([Wen et al., ICPC 2019](https://www.inf.usi.ch/faculty/lanza/PUBS/P/Wen2019a.pdf)).
- Of 588 requirements defects in 4.5 years of the Bosch project, 31 (5.3%) are inconsistencies between requirements ([Langenfeld et al.](https://swt.informatik.uni-freiburg.de/staff/langenfeld/resources/Requirements%20Defects%20over%20a%20Project%20Lifetime), Table 1).
- At least 3.3% of the facts in English Wikipedia contradict another fact in it ([Semnani et al.](https://arxiv.org/html/2509.23233), Stanford, a lower bound from manual checks).

I found no measured rate of contradictions between the documents of one company or one project, and no measured share of stale pages in a company wiki. The closest figure is that fewer than 20% of more than 13,000 corporate wikis in one organization showed continuous activity ([Arazy and Croitoru](https://cris.haifa.ac.il/en/publications/the-life-cycle-of-corporate-wikisan-analysis-of-activity-patterns-6/), abstract only).

### Glossary sizes

| Source | Size | Who chose the terms |
|---|---|---|
| Satellite ground station requirements, 380 requirements ([Arora, dissertation 2016](https://orbilu.uni.lu/bitstream/10993/28922/1/thesis_final.pdf), Table 4.5, checked) | 140 glossary terms, 238 domain model classes | Experts at the company SES |
| Safety evidence management system, 110 requirements (same table) | 51 glossary terms, 35 classes | Analysts of an EU project |
| Satellite data network, 138 requirements (same table) | 200 glossary terms, 274 classes | Experts at the company |
| NHS Data Model and Dictionary, March 2025 release ([archive](https://archive.datadictionary.nhs.uk/DD%20Release%20March%202025/), counted by the literature pass) | 796 business definitions, 226 classes, 1,366 attributes, 2,624 data elements | NHS England |
| FIM portal glossary ([fimportal.de/glossar](https://fimportal.de/glossar), counted by the literature pass) | About 143 entries | FIM office |
| FIBO, the financial industry ontology ([FOIS 2021 paper](https://fois2021.inf.unibz.it/wp-content/uploads/2021/08/paper_6.pdf), section 1) | About 1,750 classes | EDM Council |
| ACORD business glossary for insurance ([release note of 2024-07-10](https://acord.org/news-detail/2024/07/10/acord-reference-architecture-update-provides-claims-processing-enhancements), search snippet only, not verified) | 6,232 terms | ACORD |
| Glossary of one German concept document, the Saxon "Fachkonzept zum Elektronischen Staatsarchiv", 94 pages | About 23 entries | The authority |
| Open-source projects in this note | 8 to 99 entries | Volunteers |

The three industrial cases give 0.4 to 1.4 glossary terms per requirement. I found no term counts for a ubiquitous language in the domain-driven design literature.

### Published public sector specifications

Page and word counts are from the PDF text layer and include tables of contents and page headers. The XRepository entries return their version list as JSON when the request carries the header `Accept: application/json`. The literature pass counted them. I recounted the words of the files marked "checked" from its saved text.

| Document | Size | Versions |
|---|---|---|
| [XJustiz](https://xjustiz.justiz.de/XJustiz-Versionen/index.php), the data exchange standard of the German courts | 3.1.1 (2020): 518 pages, 73,793 words. 3.3.1 (2023): 938 pages, 134,870 words. 3.6.2 (2025): 1,165 pages, 179,505 words. 4.0.0 (2026): 1,268 pages, 193,022 words (checked) | 23 versions listed from 2.1.0 (2010) to 4.0.0 (2026), with 18 older ones folded away. One is valid and one is published for 2027 (checked) |
| OSCI-XMeld 26.05, the standard of the resident registers ([XRepository entry](https://www.xrepository.de/api/xrepository/urn:xoev-de:kosit:standard:xinneres.xmeld)) | 2,168 pages, 563,297 words (checked) | 33 versions up to 27.05 (checked) |
| XBau 2.1 ([XRepository entry](https://www.xrepository.de/api/xrepository/urn:xoev-de:bmk:standard:xbau)) | 328 pages, 58,792 words (checked) | 10 versions, 2.0 to 2.7 (checked) |
| XPersonenstand 1.2 (2009) ([XRepository entry](https://www.xrepository.de/api/xrepository/urn:xoev-de:kosit:standard:xinneres.xpersonenstand)) | 384 pages, 87,516 words | 26 versions |
| XFinanz 1.7 ([XRepository entry](https://www.xrepository.de/api/xrepository/urn:xoev-de:agxfinanz:standard:xfinanz)) | 161 pages, 21,315 words | 6 versions |
| XRechnung 3.0.1 (2023) ([PDF](https://xeinkauf.de/app/uploads/2023/09/301-XRechnung-2023-09-22.pdf)) | 128 pages, 40,199 words (checked) | [Version 2.3.1](https://xeinkauf.de/app/uploads/2023/02/XRechnung-v2.3.1.pdf) has 122 pages, 38,235 words |
| Fachkonzept Elektronisches Staatsarchiv Sachsen 1.0.0 (2010) ([PDF](https://www.staatsarchiv.sachsen.de/download/100927_KO_LeA_Fach_ELArch_WEB.pdf)) | 94 pages, 15,131 words (checked) | Not counted |
| Fachkonzept Langzeitspeicherung Sachsen 1.5 (2011) ([PDF](https://www.staatsarchiv.sachsen.de/download/20110909_KO_LeA_FK_LZS.pdf)) | 51 pages, 13,404 words | Not counted |
| OSiP Zielarchitektur, IT-Planungsrat 2026 ([PDF](https://www.it-planungsrat.de/fileadmin/beschluesse/2026/Beschluss_2026_22_Umsetzung_Zielarchitektur_OSiP_Gesamtdokumentation_Zielarchitektur.pdf)) | 140 pages, 30,921 words (checked) | Not counted |
| Tender specification for an e-recruiting system, Rheinland-Pfalz, draft 0.5 ([DOCX](https://mdi.rlp.de/fileadmin/03/Service/Dokumente/eRecruiting_RLP_Teil_B_Leistungsbeschreibung_.docx)) | 20,210 words | Not counted |
| FIM, the federal information management of German public services ([fimportal.de](https://www.fimportal.de/search)) | 15,619 service profiles and 49,417 master texts (checked). 1,371 data schemas, 24,645 data fields and 334 processes (literature pass) | Not counted |

A German functional concept for one system has 13,000 to 31,000 words. A mature exchange standard has 20,000 to 560,000 words and 6 to 33 published versions. XJustiz grew by a factor of 2.6 in words between 2020 and 2026.

I did not find a published Lastenheft or Pflichtenheft of a federal system. The tender platforms need a login. MITA, GOV.UK and EU specifications were not opened.

### Vendor claims

These come from vendors or vendor-paid studies and are not measurements of documentation.

- Lünendonk for the vendor msg, 2025, a survey of 152 companies. 40% name missing knowledge management and missing documentation of existing systems as a strong driver for modernization ([study PDF](https://www.msg.group/dam/jcr%3Abe2dbb5b-b9ef-462d-97f0-2550604780a5/2025_msg_L%C3%BCnendonk-Studie_IT-Modernisierung.pdf), checked).
- GitHub's Open Source Survey 2017. 93% of 5,500 respondents saw incomplete or outdated documentation as a problem ([opensourcesurvey.org](https://opensourcesurvey.org/2017/), run by GitHub). This is an opinion share, not a share of documents.

## What 10 to 20 years do to documentation

### Effects that can be counted in the systems above

- Old versions stay next to new ones. Odoo publishes 34 versions and Nextcloud 27. ERPNext kept two full copies of its manual in one folder tree, 391,274 and 389,418 words, in which 929 of 1,126 files differ by at least one byte. OpenEMR has 291 release-specific pages for 40 releases. XJustiz and XMeld publish every version back to 2010 and earlier.
- Outdated documents are archived, not deleted. OpenMRS moved 3,338 pages into "Archives". In the VistA library, 1,131 of 3,931 documents (29%) sit in entries marked "ARCHIVE". The K-9 manual is online with a note that it is abandoned. The OFBiz wiki keeps a page "Mini-Language Guide (Version 1 - Deprecated)" of 10,120 words.
- Untouched pages are common. 66% (OFBiz), 57% (OpenEMR), 49% (OpenMRS) and 13% (Dolibarr) of pages were last changed before 2017. This says the page is old. It does not say the page is wrong.
- Code keeps citing replaced specifications. `thunderbird-android` cites RFC 822 in 86 places. RFC 2822 replaced it in 2001 and RFC 5322 in 2008. Seven of the 41 RFCs the code cites are obsoleted.
- Formats pile up. Each VistA document is offered as PDF and DOCX (7,776 file links for 3,931 documents). In GovDocs1, a crawl of 986,278 files from US government sites, 23% are PDF, 7.8% old Word files, 6.4% old Excel files and 5.2% old PowerPoint files ([statistical report](https://digitalcorpora.org/corpora/file-corpora/files/govdocs1-simple-statistical-report/), shares divided by the literature pass). It is a public web crawl, not a company archive.

### Effects measured to hurt retrieval or question answering

The earlier note already covers VersionRAG and HoH. Nothing newer replaces them. The table adds what the literature pass found.

| Effect | Measured harm | Source and who measured |
|---|---|---|
| Several dated versions of one text | A retrieval setup over only the current version of the French tax code found the version that applied on the question's date in 0% of cases. A retriever over all 32,436 versions reached 98.3% | [FiscalQA Pro](https://arxiv.org/html/2608.09393), the method's own authors at a company, abstract and introduction read |
| Near-duplicates in the corpus | With injected near-duplicates on HotpotQA, recall at 5 fell from about 79 to 53 and exact match from 34 to about 26.5. Diversified retrieval kept recall at 73 to 75 | [arXiv 2610.09412](https://arxiv.org/html/2610.09412), academics who also propose the fix, values read from figure descriptions |
| Highly similar documents | On 4-hop questions a strong retriever reaches 5.0% (legal), 8.5% (finance) and 27.9% (patent) against 66.4% on Wikipedia | [RARE](https://arxiv.org/html/2604.19047), benchmark authors at a company and a university |
| Conflicting documents | GPT-4o loses 11.6 points of exact match (54.13 to 42.51) when the retrieved contexts conflict. About 25% of plain questions get conflicting contexts from web search | [QACC](https://arxiv.org/html/2410.12311), Table 3, Amazon researchers |
| Conflicting documents | With two contradicting Wikipedia passages, Llama 3 70B answers correctly 10.4% of the time and 43.8% when told to look for conflicts, against 85% to 87% without conflict | [WikiContradict](https://arxiv.org/html/2406.13805), IBM Research |
| Scanned or badly parsed documents | End-to-end F1 falls from 36.1 with clean text to 31.1 with the best OCR and to 14.5 with the worst. Retrieval of evidence in tables falls from 69.6 to 48.5 with a common parser | [OHRBench](https://arxiv.org/html/2412.02592), Table 2, benchmark authors, checked |
| Question and document in different languages | Extractive answering of a German question on an English passage scores 58.5 F1 against 74.9 for English on English | [MLQA](https://ar5iv.labs.arxiv.org/html/1910.07475), Table 6, Facebook AI researchers, checked |
| Question and document in different languages | For German questions against English Wikipedia, BM25 reaches a recall at 100 of 35.4 and dense models 76.2 to 76.9 | [BGE-M3](https://arxiv.org/html/2402.03216), Table 2, the model's own authors |
| A growing corpus around the same questions | With the same 500 questions, BM25 falls from 74.7 at 1,144 documents to 70.1 at 6,980 and 50.5 at 511,959. Dense retrieval falls from 58.1 to 51.0 and 29.9. An agent that reads files leads below about 7,000 to 8,500 documents and loses above | [arXiv 2607.26497](https://arxiv.org/html/2607.26497), Table 1, academics independent of the benchmark vendor, checked |
| A growing corpus around the same questions | Recall at 100 of the E5 model falls from 84.15 at 10,000 passages to 75.39 at 1 million and 50.77 at 100 million. The order of models changes with size | [CoRECT](https://arxiv.org/html/2510.19340), Table 2, University of Passau, checked |

Not measured anywhere I looked: the effect of terms that changed meaning or were renamed inside one organization, the loss from converting old Word, Visio or Lotus Notes files, and the effect of archived wiki pages. The only number near term drift is old. Two people chose the same name for the same thing with a probability below 0.20 ([Furnas et al., 1987](https://dl.acm.org/doi/10.1145/32206.32212), snippet only).

No paper states a corpus size above which a retrieval result carries over to a larger corpus. The two scaling studies say that scores keep falling and that rankings can flip, so a small project memory cannot stand in for a large one.

One warning concerns how the synthetic part is made. When a language model wrote 100 documents from a company overview alone, a judge model flagged over 40% as having a very close sibling ([EnterpriseRAG-Bench](https://arxiv.org/html/2605.05253), section 4.1). Synthetic domain documents need distinct source material per document.

## A real corpus for the mail domain and for Shopware

### IETF mail standards

I selected RFCs from the [RFC index](https://www.rfc-editor.org/rfc-index.xml) by a title rule (IMAP, SMTP, MIME, POP, Sieve, DKIM, DMARC, JMAP, LMTP, "mail", "message format" and similar words, listed in the method section) and downloaded each text file from rfc-editor.org.

| Set | RFCs | Pages | Words |
|---|---|---|---|
| Title rule, all | 569 | 10,809 | 2,902,648 |
| Title rule, not obsoleted | 435 | 7,835 | 2,103,688 |
| Title rule, obsoleted by a later RFC | 134 (24%) | 2,974 | 798,960 (28%) |
| Cited in the `thunderbird-android` source | 41 | 1,272 | 366,735 |
| Cited and not obsoleted | 34 | 894 | 261,195 |

- The RFCs date from 1971 to 2026. By decade there are 32, 37, 156, 182, 119 and 43.
- 130 of the 569 obsolete at least one earlier RFC. The mail core has three-step chains: RFC 822, 2822 and 5322 for the message format, RFC 821, 2821 and 5321 for SMTP, and RFC 1730, 2060, 3501 and 9051 for IMAP.
- The word count includes page headers, footers and tables of contents.
- License. The RFC Editor FAQ says that "All RFCs may be freely reproduced and translated (unmodified)" ([rfc-editor.org/faq](https://www.rfc-editor.org/faq/)). Section 3.c of the [IETF Trust Legal Provisions 5.0](https://trustee.ietf.org/documents/trust-legal-provisions/tlp-5/) allows copying and distributing IETF documents "in full and without modification", translating them, and copying unmodified portions with attribution and all legal notices kept. Section 3.d grants no right to modify.
- Result. The RFC text may be copied unmodified into a public GitHub repository, each file whole with its notices, and may be translated into German. Edited versions may not be published, so a synthetic "outdated" variant of an RFC must not go into the repository. The real obsoleted RFCs supply the outdated versions.
- Limit. The [IETF Trust FAQ](https://trustee.ietf.org/about/faq/) says the Trust "cannot offer definitive advice" for RFCs from before 1986. 53 of the 569 RFCs, with 156,973 words, are from before 1986, and 3 of the 41 cited ones are (RFC 822, 921 and 952). Leaving them out avoids the question and leaves 516 RFCs with 2,745,675 words, 119 of them obsoleted, and 38 cited RFCs with 349,730 words, 6 of them obsoleted.

### Mozilla support articles for Thunderbird

- Count. The repository [`thunderbird/github-action-thunderbird-kb`](https://github.com/thunderbird/github-action-thunderbird-kb) holds an export of all English support articles, last updated 2026-06-12 (commit `9453966`). It has 1,296 articles. 176 carry the product `thunderbird` with 149,219 words, and 14 carry `thunderbird-android` with 12,368 words. A Thunderbird slide deck of 2024 gives about 150 articles in more than 12 languages ([blog.thunderbird.net](https://blog.thunderbird.net/files/2024/07/SUMO-Knowledge-Base.pdf), search snippet, not opened).
- German. A Mozilla wiki page names German among the languages the articles tend to be available in ([wiki.mozilla.org](https://wiki.mozilla.org/Thunderbird/Support/Community_support_based_on_languages), search snippet). I could not count the German articles. The site answers scripts and its own API with a client challenge page, and I did not work around it.
- License. The site footer reads "Content available under a Creative Commons license" and links to Mozilla's website content page ([footer template](https://github.com/mozilla/kitsune/blob/main/kitsune/sumo/jinja2/base.html), line 385). That page names the "Creative Commons Attribution Share-Alike 3.0 Unported license, or any later version" and excludes trademarks and logos ([mozilla.org](https://www.mozilla.org/en-US/foundation/licensing/website-content/)).
- Result. The article text may be copied into a public repository with attribution, and the copied text stays under CC BY-SA. Screenshots with Mozilla logos should be left out. The export repository gives the English text without a scripted read of the site. German text needs a manual export or permission from Mozilla.
- Fit. Support articles describe what a user does. They are the closest public match to a user manual for the mail domain. They are not business rules.

### K-9 Mail and Thunderbird for Android documentation

- The Markdown in `thunderbird/thunderbird-android` is Apache-2.0 and already part of the test dataset. It is engineering documentation.
- The old K-9 manual has 30 Markdown files and 89,577 bytes on the `stable` branch, about 14,000 words by my estimate of 6.5 bytes per word. The repository has no license file, so the text may not be copied. The K-9 website repository is CC BY 4.0, but the manual is not in it.

### Shopware

- Developer documentation, `shopware/docs`: 1,053 files and 744,633 words. The GitHub API reports no license, and the repository has no license file. Without a license the text may be read and fetched but not copied into another public repository.
- User documentation, docs.shopware.com: 344 English and 343 German pages with 327,619 and 298,237 words. Every page ends with "Copyright © shopware AG - Alle Rechte vorbehalten". The site's `robots.txt` allows all crawlers and carries the line `Content-Signal: search=yes, ai-input=yes, ai-train=yes`. That is a signal about automated use, not a license to republish.
- The code repository `shopware/shopware` is MIT and holds 189 ADR files. They may be copied. They are decision records, not domain knowledge.
- Result. Shopware's own documentation can only be fetched at test time. The German user pages are the best public source of German e-commerce domain text for the large project memory: catalogs, orders, payments, taxes, shipping. A fetch script with pinned URLs and dates can live in the repository. The text cannot.

## How far public numbers stand in for client projects

The public numbers stand in for client projects only as a lower bound on volume. They say little about content and form.

- Public repositories and wikis document a product for many installations. A client's domain knowledge describes one organization's rules, exceptions and processes. The public systems above have almost no functional specifications. iDempiere has 6 functional files against 419 files on new features.
- Public text is written to be read by strangers. Client documents assume context, name internal systems and people, and live in Word, Excel, PowerPoint and wiki exports. None of the twelve systems keeps its main documentation in office formats.
- The public numbers still bound the size from below. A project that no rule forces to document produces 0.33 million words of functional text at the median. I assume that a client with audit and procurement duties has no less. That is a judgment, not a measurement.
- The glossary numbers do not carry over. Open-source glossaries are an afterthought. The industrial ones (51 to 200 terms per requirements set) and the organization-wide ones (143 and 796) are the better guide.

Public sources that come closer to a client system:

- The VA Software Document Library. It is a real hospital and billing system with documents from 1993 on, user and technical manuals per application, archive entries and office formats. It is in English. I did not check its terms of use.
- The XÖV standards and FIM. They are German, functional, versioned over 15 years and openly published. They specify data exchange, not a whole application.
- The PURE requirements documents and the Saxon concept documents. They show what one specification looks like and how long it is.
- Tender specifications of German states, such as the Rheinland-Pfalz e-recruiting document. They are the nearest public match to a Lastenheft.
- The NovAtel case study. It is the only source found that counts a company's documents: more than 60,000 with revisions, a few hundred for one product, 30 revisions per sampled document.

None of these shows a full client project. The size of a real project memory's domain knowledge part stays an estimate until the first client project is measured.

## Recommended size

### Derivation for the small project memory

1. Unit. A small modernization project covers one application, not a product suite. This is a judgment.
2. Words before documents. Words are the stable unit, because the document count depends on the format. A wiki page has 250 to 950 words and a specification document 4,000 to 15,000.
3. Words for one application. A VistA application at the 80th to 90th percentile has 21 to 41 documents. At the sampled mean of 8,378 words that is 174,000 to 340,000 words. About half of the sampled documents are release notes and installation guides, which are not domain knowledge, so the domain part by this route is roughly 90,000 to 170,000 words. That halving is an estimate. The smallest whole product measured has 49,500 words of functional text and the median whole product 326,190. I take 150,000 to 300,000 words. That is above the VistA estimate, because a project memory also holds what the modernization project itself writes down: target specifications, process descriptions and rule lists. It stays at or below the median whole product. The choice of percentile and the allowance for project-written documents are judgments.
4. Documents. A sampled VistA document has a median of 4,133 and a mean of 8,378 words. Requirements documents have a median of 6,774 (PURE) and 7,539 words (financial systems). German concept and tender documents have 13,000 to 31,000. At about 5,000 words per document, 150,000 to 300,000 words are 30 to 60 documents. Text that is modeled as wiki pages takes 5 to 20 pages per document.
5. Cross-check. The range is 1.3 to 2.6 times the 113,336 words of Markdown the test repository already has. The business applications measured above have 3 to 4 times as much functional text as developer text, so the upper end is still modest.
6. Glossary. The industrial cases have 51, 140 and 200 terms. I take 100 to 200. Every term gets a German and an English label, because the cross-language question type of the earlier note needs both.
7. Superseded and outdated versions. The measured shares are 24% obsoleted RFCs, 29% of VistA documents in archive entries, 19.2% of documentation files with an outdated reference today and 40.7% ever, and 13% to 66% of wiki pages untouched for ten years. I take 25% to 35% of all domain documents, which is 15 to 30 documents on top of the current ones. An older version is shorter. XJustiz 3.1.1 has 38% of the words of 4.0.0. That adds about 40,000 to 100,000 words.
8. Version chains. Mail has real three-step chains, and the standards above have 6 to 33 versions. At least 3 documents should exist in 3 or more versions, so that "which version applied when" can be asked. The number 3 is a judgment tied to the 10 questions per type the earlier note asks for.
9. German share. Public data does not answer this for client documents. German public specifications are 100% German. Vendors that serve German customers translate everything. I take 60% to 80% German for the synthetic documents, with English terms where the code uses them. This is a judgment about German clients.
10. Formats. A fifth to a third of the documents should arrive as PDF or DOCX and not as Markdown. The basis is thin. The VistA library offers both formats for every document, and old office formats are 19% of GovDocs1. This is a judgment.
11. Contradictions. No natural rate exists for project documents. A handful of planted disagreements between a document and a newer one, or between a document and the code, is all the evidence supports. The earlier note's 10 questions per type set the lower bound.

### Derivation for the large project memory

1. Unit. The large project memory stands for a whole business platform with several modules.
2. Documents and words. Shopware's own user documentation is the nearest measured match. It has 344 pages and 327,619 words in English, and 687 pages and 625,856 words in both languages. The functional parts of Odoo and Dolibarr have 1.1 and 1.25 million words in 963 files and 1,953 pages. I take 300 to 700 current documents and 0.6 to 1.5 million words.
3. Glossary. Several modules of 100 to 200 terms each, bounded by the NHS dictionary's 796 business definitions, give 400 to 800 terms. The step from one module to several is a judgment.
4. Superseded versions. The same 25% to 35% gives 100 to 300 documents, with at least 10 chains of 3 or more versions.
5. German share. Shopware publishes its user documentation page for page in both languages, so about 50%.
6. Scale effects. The scaling study saw the ranking of methods flip at about 7,000 to 8,500 documents. Whether a project memory lies above or below that depends on its issues and pull requests, which this note did not count. Results from the small project memory should not be read as results for the large one.

### How the real corpora fit

- Small, mail. The 41 RFCs that the code cites (366,735 words, 7 of them obsoleted) can serve as the functional specifications, copied unmodified. The 176 English Thunderbird support articles (149,219 words) can serve as the user manual. Together that is 217 real documents and about 516,000 English words, more than the range above. A cited RFC has 8,945 words on average and an article 848. A subset of 15 to 20 RFCs and 30 to 60 articles, together with the synthetic layer in the next point, lands inside the range.
- The client-specific layer stays synthetic for mail. It is 10 to 20 German documents of 40,000 to 100,000 words with business rules, process descriptions and a functional concept for the fictional client, plus the glossary. Real text has no client, no German and no contradiction with the test repository's code.
- With this mix the German share of all domain words falls to about 10% to 40%, by my arithmetic from the figures above. Reaching 60% needs German translations of the RFC subset, which the license allows, or the German support articles, which need a manual export.
- Large, Shopware. The German and English user pages match the range by themselves but may only be fetched at test time. Anything committed to the repository has to be synthetic or come from the MIT-licensed code repository.

### What is measured and what is judgment

Measured: every count in the tables, the size of specification documents, the glossary sizes, the shares of old and obsoleted documents and the license texts. Judgment: that a small project equals one application, the choice of the 80th to 90th percentile of documents per application, the allowance for documents the project writes, the average of 5,000 words per document, the German share for client documents, the share of office formats, and the number of version chains and contradictions.

## Method and limits

Counting rule. A word is a run of letters and digits, with hyphens and apostrophes allowed inside. For Markdown, RST, AsciiDoc and LaTeX files I counted the raw file, so directive names and link targets count as words. For MediaWiki pages I removed HTML tags and the brackets and braces of wiki markup first. For Confluence pages and HTML pages I removed tags, and for Shopware pages I counted only the `article` element. The counts are therefore a few percent too high for marked-up text. They are consistent within one system and comparable between systems to about the first digit.

Commands and sources per system:

- Git repositories. `git clone --filter=blob:none --no-tags <url>` into a scratch directory, with `--no-checkout` and `git checkout <branch> -- '*.rst' '*.md'` for large repositories, then a Python script that walks the tree and applies the counting rule per top-level folder. Dates come from `git log --reverse --format=%cs | head -1` and `git log -1`. Branch lists come from `git ls-remote --heads` and `gh api repos/<repo>/branches --paginate`. No code from any clone was run.
- Translations. A script counted `msgid` entries and non-empty `msgstr` entries in the `.po` and `.pot` files under `locale/de` (Odoo) and `user_manual/locale/de_DE` (Nextcloud).
- MediaWiki (OpenEMR, Dolibarr). The API call `action=query&generator=allpages&gapnamespace=0&gapfilterredir=nonredirects&prop=revisions&rvprop=timestamp|content|size`, 50 pages per request, gave the text and last change of every content page. A second call per page with `rvdir=newer&rvlimit=1` gave the first revision. The page counts differ from the wikis' own statistics (957 and 1,928 "articles") because MediaWiki counts only pages with a link.
- Confluence (OFBiz, OpenMRS). `rest/api/content?spaceKey=<key>&type=page&expand=body.storage,version,history`, 50 pages per request, for the spaces `OFBIZ`, `OFBENDUSER`, `docs`, `projects`, `RES` and `Archives`.
- Moodle. The pages `Special:Statistics` of four English wikis and one German wiki, read through the fetch tool. Direct requests returned HTTP 403.
- Shopware user site. The sitemaps `docs.shopware.com/en/sitemap.xml` and `/de/sitemap.xml`, then one request per URL.
- VistA document sample. 60 draws with replacement from the 3,931 documents, weighted by documents per application, with the random seed 20261120. One document was drawn twice. Pages and words come from the PDF text layer through macOS PDFKit.
- Thunderbird support articles. The file `details-allproducts-kb-title-slug-all-articles.csv` of the export repository, counting the `text` column per value of `products_str`.
- RFCs. `rfc-index.xml` for titles, years, page counts, status and the obsoleted-by relation. The title rule is the regular expression `IMAP\w*|Internet Message Access|SMTP|Simple Mail Transfer|MIME|S/MIME|Multipurpose Internet Mail|Internet Message Format|ARPA Internet Text Messages|POP3|POP|Post Office Protocol|Sieve|e-?mail\w*|mail\w*|Message Submission|DKIM|DomainKeys|DMARC|Sender Policy Framework|JMAP|JSON Meta Application Protocol|LMTP|Message Disposition|Delivery Status|Message Header\w*|Authenticated Received Chain|Message Tracking|Internet Message\w*`, matched on word boundaries without case. The citations come from `grep -rhoiE 'RFC[ -]?[0-9]{3,4}'` over the Kotlin, Java, Markdown and XML files of the clone, without numbers that start with 0, which are the project's own RFC documents.
- Language of Dolibarr pages. A count of 17 to 20 stop words per language over each page's text, with the highest count deciding. Pages under 30 words are "short".
- Keyword rules for the split. OpenEMR: a title is "domain" if it matches words such as "meaningful use", "billing", "clinical", "claim", "insurance", "prescri", "patient", "encounter" or "ICD", "developer and admin" if it matches "develop", "API", "install", "upgrade", "release", "database", "server" and similar, and "user" if it matches "user guide", "how to" or "tutorial". Dolibarr: "developer and admin" if the title matches "develop" in four languages, "API", "hook", "install", "upgrade", "release", "FAQ" and similar, otherwise "functional and user".

Limits:

- The split between functional and developer text is exact only where a project uses separate folders or spaces. For OpenEMR and Dolibarr it is a title rule that I did not validate against a hand-labeled sample.
- "Last changed before 2017" counts any edit, including a typo fix, as a change. The true share of outdated content is unknown.
- Version counts are counts of branches or of titles with version numbers. Apart from the byte comparison of the two ERPNext folders, I did not measure how much text differs between two versions.
- The Moodle word count, the German Thunderbird article count, the iDempiere wiki and the current ERPNext manual are missing because the sites refuse scripted reads or were out of reach in the time available.
- Project founding years are not measured here. The table gives the oldest document date I could measure.
- The literature passes read most papers in full text and some as abstracts or snippets. The text marks the weaker readings. Numbers without "checked" come from one reading.
- Five of the sources are 2026 preprints (arXiv 2604.19047, 2605.05253, 2607.26497, 2608.09393 and 2610.09412). Their peer review status was not checked.
- License statements are my reading of the published terms, not legal advice.

One claim was dropped. A search summary attributed "a quarter have large numbers of undocumented applications" to the vendor Kyndryl, and no Kyndryl page that I or the literature pass opened contains it.

Searches that found nothing: page or word counts of company wikis or SharePoint sites, a measured share of stale pages in Confluence, a contradiction rate between documents of one company, term counts of a ubiquitous language in domain-driven design case studies, a measured effect of renamed terms on retrieval, a measured loss from converting old office formats, a published Lastenheft or Pflichtenheft of a German federal system, and a count of German Thunderbird support articles.

## Sources

Measured systems:

- Odoo documentation: https://github.com/odoo/documentation
- Nextcloud documentation: https://github.com/nextcloud/documentation
- Shopware developer documentation: https://github.com/shopware/docs, user documentation: https://docs.shopware.com/en and https://docs.shopware.com/de, code repository: https://github.com/shopware/shopware
- LedgerSMB book: https://github.com/ledgersmb/ledgersmb-book, code repository: https://github.com/ledgersmb/LedgerSMB
- Apache OFBiz: https://github.com/apache/ofbiz-framework, https://cwiki.apache.org/confluence/display/OFBIZ, https://cwiki.apache.org/confluence/display/OFBENDUSER
- OpenEMR wiki: https://www.open-emr.org/wiki/
- OpenMRS wiki: https://openmrs.atlassian.net/wiki
- Dolibarr wiki: https://wiki.dolibarr.org
- ERPNext documentation (archived): https://github.com/frappe/erpnext_documentation
- iDempiere documentation site: https://github.com/idempiere/idempiere.github.io
- MoodleDocs: https://docs.moodle.org/overview/, https://docs.moodle.org/503/en/Special:Statistics, https://docs.moodle.org/501/en/Special:Statistics, https://docs.moodle.org/311/en/Special:Statistics, https://docs.moodle.org/19/en/Special:Statistics, https://docs.moodle.org/de/Spezial:Statistik
- Thunderbird for Android: https://github.com/thunderbird/thunderbird-android, support article export: https://github.com/thunderbird/github-action-thunderbird-kb, K-9 manual: https://github.com/k9mail/k9mail-docs
- Thunderbird support languages and article count (search snippets only): https://wiki.mozilla.org/Thunderbird/Support/Community_support_based_on_languages, https://blog.thunderbird.net/files/2024/07/SUMO-Knowledge-Base.pdf

Licenses and terms:

- RFC Editor FAQ: https://www.rfc-editor.org/faq/
- IETF Trust Legal Provisions 5.0: https://trustee.ietf.org/documents/trust-legal-provisions/tlp-5/
- IETF Trust FAQ: https://trustee.ietf.org/about/faq/
- RFC index: https://www.rfc-editor.org/rfc-index.xml
- Mozilla website content license: https://www.mozilla.org/en-US/foundation/licensing/website-content/
- Mozilla support site footer template: https://github.com/mozilla/kitsune/blob/main/kitsune/sumo/jinja2/base.html
- Shopware user documentation robots file: https://docs.shopware.com/robots.txt

Studies on documentation volume, staleness and glossaries:

- Ferrari, Spagnolo and Gnesi, PURE dataset: https://zenodo.org/records/1414117
- Veizaga, Shin and Briand, Paska: https://arxiv.org/pdf/2305.07097
- Garousi et al., Usage and usefulness of technical software documentation: https://pureadmin.qub.ac.uk/ws/portalfiles/portal/180209101/IST2013_Special_Issue_July30.pdf
- Langenfeld, Post and Podelski, Requirements defects over a project lifetime: https://swt.informatik.uni-freiburg.de/staff/langenfeld/resources/Requirements%20Defects%20over%20a%20Project%20Lifetime
- Tan, Wagner and Treude, Detecting outdated code element references: https://arxiv.org/pdf/2212.01479
- Aghajani et al., Software documentation issues unveiled: https://emadpres.github.io/pdfs/icse2019.pdf
- Forward and Lethbridge, The relevance of software documentation, tools and technologies: https://www.eecs.uottawa.ca/~tcl/gradtheses/aforward/papers/aforwarddoceng2002sub.pdf
- Wen et al., A large-scale empirical study on code-comment inconsistencies: https://www.inf.usi.ch/faculty/lanza/PUBS/P/Wen2019a.pdf
- Semnani et al., CLAIRE and WikiCollide: https://arxiv.org/html/2509.23233
- Arazy and Croitoru, The life cycle of corporate wikis (abstract): https://cris.haifa.ac.il/en/publications/the-life-cycle-of-corporate-wikisan-analysis-of-activity-patterns-6/
- Arora, Automated analysis of natural-language requirements (dissertation): https://orbilu.uni.lu/bitstream/10993/28922/1/thesis_final.pdf
- NHS Data Model and Dictionary, March 2025: https://archive.datadictionary.nhs.uk/DD%20Release%20March%202025/
- FIBO at FOIS 2021: https://fois2021.inf.unibz.it/wp-content/uploads/2021/08/paper_6.pdf
- GovDocs1 statistical report: https://digitalcorpora.org/corpora/file-corpora/files/govdocs1-simple-statistical-report/
- Furnas et al., The vocabulary problem in human-system communication (snippet only): https://dl.acm.org/doi/10.1145/32206.32212

Public sector documents:

- XJustiz versions: https://xjustiz.justiz.de/XJustiz-Versionen/index.php
- XRepository (XMeld, XBau, XPersonenstand, XFinanz): https://www.xrepository.de
- XRechnung and XBestellung: https://xeinkauf.de
- FIM portal: https://www.fimportal.de/search, glossary: https://fimportal.de/glossar
- VA Software Document Library: https://www.va.gov/vdl/
- Rheinland-Pfalz e-recruiting tender specification: https://mdi.rlp.de/fileadmin/03/Service/Dokumente/eRecruiting_RLP_Teil_B_Leistungsbeschreibung_.docx

Retrieval and question answering studies:

- FiscalQA Pro: https://arxiv.org/html/2608.09393
- Near-duplicate injection study: https://arxiv.org/html/2610.09412
- RARE and RedQA: https://arxiv.org/html/2604.19047
- Liu et al., QACC: https://arxiv.org/html/2410.12311
- Hou et al., WikiContradict: https://arxiv.org/html/2406.13805
- Zhang et al., OHRBench: https://arxiv.org/html/2412.02592
- Lewis et al., MLQA: https://ar5iv.labs.arxiv.org/html/1910.07475
- Chen et al., BGE-M3: https://arxiv.org/html/2402.03216
- Corpus scaling study on EnterpriseRAG-Bench: https://arxiv.org/html/2607.26497
- CoRECT: https://arxiv.org/html/2510.19340
- Sun et al., EnterpriseRAG-Bench (vendor Onyx): https://arxiv.org/html/2605.05253
- HERB (Salesforce AI Research): https://arxiv.org/html/2506.23139

Vendor and vendor-paid material:

- Lünendonk and msg, IT-Modernisierung 2025: https://www.msg.group/dam/jcr%3Abe2dbb5b-b9ef-462d-97f0-2550604780a5/2025_msg_L%C3%BCnendonk-Studie_IT-Modernisierung.pdf
- GitHub Open Source Survey 2017: https://opensourcesurvey.org/2017/

Earlier research in this repository: `docs/research/documentation-graph.md` on the branch `research/documentation-graph`.
