---
status: accepted
date: 2026-11-21
decided-by: Niko4417
ticket: https://github.com/Niko4417/OPT-cc/issues/23
---

# Size the domain knowledge part from measured systems and fill it mostly with real documents

The test dataset had no domain knowledge and no glossary, so the bake-off could not test questions about them. A first guess of 8 documents and 40 glossary terms had no basis. The [domain documentation volume research](https://github.com/Niko4417/OPT-cc/blob/research/domain-documentation-volume/docs/research/domain-documentation-volume.md) measured 11 public business systems. Their documentation has 0.05 to 2.9 million words, and the functional and user part, where it can be separated, has 49,500 to 1.25 million words with a median of 326,190. Glossaries in industrial requirements sets have 51, 140 and 200 terms.

**Targets**, for real and synthetic text together:

| | Thunderbird | Shopware |
|---|---|---|
| Current documents | 150,000 to 300,000 words | 0.6 to 1.5 million words |
| Outdated versions on top | 25% to 35%, with at least 3 chains of 3 versions | 25% to 35%, with at least 10 chains of 3 versions |
| Glossary terms, each with a German and an English name | 100 to 200 | 100 to 200 |

**Real documents supply the volume and the age.**

- Thunderbird gets 15 to 20 of the 41 mail RFCs that its code cites, each copied whole and unmodified with its notices, and 30 to 60 Mozilla support articles under CC BY-SA with attribution and without logos. The RFCs bring real chains of replaced versions, such as RFC 822, 2822 and 5322. The code still cites RFC 822 in 86 places.
- RFCs from before 1986, RFC 822 among them, are fetched at test time and not copied, because the IETF Trust gives no firm advice on copying them.
- Shopware gets its own user documentation, 327,619 words in English and 298,237 in German. Every page says "Alle Rechte vorbehalten". A script with pinned URLs fetches the pages at test time, and the text stays in the private snapshot of ADR-0032. The owner accepts fetching it for internal testing. If that changes, Shopware's part falls back to synthetic text only.

**A synthetic client layer sits on top.** Each project gets 10 to 20 synthetic documents of 40,000 to 100,000 words in total, mostly German, with business rules, process descriptions and a functional concept for the fictional client. A fifth to a third arrive as PDF or Word. Each document is written from its own source material. The glossary, the version chains that real text does not supply, and a handful of contradictions with newer documents or with the code are planted here.

**German share.** German makes up about 10% to 40% of Thunderbird's domain words. The research judges 60% to 80% typical for German clients. The lower share is accepted, because Shopware's real documentation is half German and every result is reported per language (ADR-0033).

## Considered options

- **8 documents and a glossary of 40 terms.** A guess, 4 to 7 times too small in documents for the small project and 40 to 90 times for the large one.
- **Synthetic documents only.** Rejected. When a language model wrote 100 documents from one company overview, over 40% were flagged as having a very close sibling (EnterpriseRAG-Bench). And a person could not review that volume.
- **German translations of the RFCs.** The license allows them. Not chosen, because machine translations of standards are not what a client holds.
- **German Mozilla support articles.** They would raise the German share. Mozilla's site blocks scripted reads, so a person would have to export them by hand. Left open for later.
- **400 to 800 glossary terms for the large project**, as the research suggests. Not chosen, because both projects follow one recipe (ADR-0031). This is a known gap.

## Consequences

- An unmodified RFC cannot carry the synthetic notice of ADR-0030, and an edited RFC may not be published. Outdated RFC versions are therefore the real obsoleted RFCs, and the manifest records origin and license of every real document.
- Shopware's user documentation covers only the current version. Its outdated versions and all of its version chains have to come from the synthetic layer, which makes that layer larger for Shopware than for Thunderbird in this one respect.
- Answer models know the mail RFCs from training. The run without retrieval in ADR-0033 marks the questions they answer from memory.
- Old versions and near-duplicates hurt retrieval. A setup that held only the current version of the French tax code found the version valid on the question's date in 0% of cases, and injected near-duplicates lowered recall at 5 from about 79 to 53. The dataset reproduces both effects on purpose.
- The domain documents and the glossary make it possible to test whether a graph helps with domain knowledge.
- No public source counts a company's wiki or file share. The size of a real client's domain knowledge stays an estimate until the first client project is measured. The word ranges come from measurements. The outdated share and the chain counts are judgment.
- The license statements are a reading of the published terms, not legal advice.
