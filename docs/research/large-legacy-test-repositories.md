# A second, large legacy test repository

Date: 2026-11-20. Follow-up to [#8](https://github.com/Niko4417/OPT-cc/issues/8). Map: [#1](https://github.com/Niko4417/OPT-cc/issues/1).

## Question

Which public GitHub repositories with 1 million or more lines of legacy code can be a second, large test repository?

The research for #8 picked [`thunderbird/thunderbird-android`](https://github.com/thunderbird/thunderbird-android). It has about 12.9 MB of Kotlin and Java, which is about 370,000 lines. That is too small to test retrieval at 1 to 2 million lines.

## Answer

Use [`shopware/shopware`](https://github.com/shopware/shopware) as the large second repository, pinned at commit [`470e9fe`](https://github.com/shopware/shopware/commit/470e9fe474ed51d3dad5a48328b89237e74ed2a6) (2026-11-20). Keep Thunderbird for fast evaluation loops and for the German workshop content.

- Shopware has about 1.16 million lines of production code and about 1.3 million lines of tests, counted in a shallow clone. Most of it is PHP, with a JavaScript and TypeScript administration and Twig templates.
- 181 ADRs in [`adr/`](https://github.com/shopware/shopware/tree/470e9fe474ed51d3dad5a48328b89237e74ed2a6/adr) date from 2020 to 2026, 46 of them from 2026. Seven more sit in `adr/_superseded/`.
- Several migrations are visible in the code: Vuex to Pinia (done), XML to PHP container configuration (done), the old `sw-` components to the Meteor component library (in progress) and administration JavaScript to TypeScript (in progress). The first three have an ADR.
- The license is MIT. 4,556 pull requests were merged in the last 12 months.

Shopware's ADRs have no status field. The status is the folder (`adr/` or `adr/_superseded/`) and a few inline "supersedes" notes. Before 2024 the team tracked work in Jira, so older changelog entries point to Jira IDs that do not resolve on GitHub.

If a third repository is needed, take one of these:

- [`openemr/openemr`](https://github.com/openemr/openemr) for the common client case: about 1.2 million hand-written lines of PHP, a written rule that freezes the legacy folders, and no decision records. It needs synthetic decision records, as Thunderbird does.
- [`kubernetes/kubernetes`](https://github.com/kubernetes/kubernetes) with [`kubernetes/enhancements`](https://github.com/kubernetes/enhancements) for decision status at scale: 686 KEPs with a `status:` field, among them withdrawn, replaced and rejected ones.

No other candidate in Tier 1 combines a stated legacy migration, decision records inside the code repository and an active GitHub tracker the way Shopware does. Kubernetes and CPython keep their decision records in a separate repository. Gradle has only 12 ADRs.

## Selection criteria

The criteria of #8, with the size raised:

1. A legacy system is modernized, and the repo says so.
2. Decision records are Markdown files in the repo, with status fields.
3. Issues and pull requests are on GitHub, not in Jira.
4. The license allows internal commercial use as test data.
5. The code has at least 1 million lines.
6. The project is active, so the issues and pull requests reflect current work.

Projects that track issues outside GitHub fail criterion 3 and were not surveyed. Examples are LibreOffice, Moodle, GitLab, Drupal, MediaWiki, Liferay and most Apache Java projects.

## Ranked candidates

Lines are estimates from GitHub's language bytes (see "Method and limits"). Issues and pull requests are totals. "Merged" counts pull requests merged since 2025-11-20.

### Tier 1: meets the criteria at 1 million lines or more, with gaps named per candidate

| Repository | License | Modernization story | Decision records | Issues / PRs / merged | Lines |
|---|---|---|---|---|---|
| [shopware/shopware](https://github.com/shopware/shopware) | MIT | Vuex to Pinia, XML to PHP container configuration, `sw-` to Meteor components, JavaScript to TypeScript | 181 ADRs + 7 superseded, no status field | 10,790 / 12,146 / 4,556 | 2.5 M counted (1.16 M production) |
| [kubernetes/kubernetes](https://github.com/kubernetes/kubernetes) + [enhancements](https://github.com/kubernetes/enhancements) | Apache-2.0 | In-tree cloud providers removed, in-tree volume plugins migrated to CSI, dockershim removed | 686 KEPs with a status field, in the separate repo | 50,063 / 93,534 / 4,032 | 3.1 M |
| [python/cpython](https://github.com/python/cpython) + [peps](https://github.com/python/peps) | PSF-2.0 (PEPs: public domain or CC0) | Optional GIL, JIT compiler, removal of dead standard library modules | 748 PEPs with a `Status:` header, in the separate repo | 79,355 / 80,555 / 11,311 | 2.0 M |
| [gradle/gradle](https://github.com/gradle/gradle) | Apache-2.0 | Monolith split into platforms, Groovy types out of the public API, lazy provider APIs | 12 ADRs with a status line | 15,846 / 23,347 / 2,494 | 2.4 M |

### Tier 2: one criterion is weak

| Repository | License | Decision records | Issues / PRs / merged | Lines | Weak point |
|---|---|---|---|---|---|
| [RocketChat/Rocket.Chat](https://github.com/RocketChat/Rocket.Chat) | MIT outside the `ee/` folders, a separate enterprise license inside ([LICENSE](https://github.com/RocketChat/Rocket.Chat/blob/06942c84e7b0dc886e07f446e22953a6cfc1adaf/LICENSE)) | 13 ADRs in `docs/adr`, 16 in the XMPP server package, 7 proposals | 17,351 / 24,913 / 2,866 | 0.9 M | Below 1 million lines. Moving off Meteor is the story ([ADR 0009](https://github.com/RocketChat/Rocket.Chat/blob/06942c84e7b0dc886e07f446e22953a6cfc1adaf/docs/adr/0009-web-client-built-by-vite-served-by-the-server.md), [ADR 0010](https://github.com/RocketChat/Rocket.Chat/blob/06942c84e7b0dc886e07f446e22953a6cfc1adaf/docs/adr/0010-own-http-server-in-front-of-meteor.md)) |
| [camunda/camunda](https://github.com/camunda/camunda) | Split between Apache-2.0 and Camunda License 1.0 ([licenses/](https://github.com/camunda/camunda/tree/6c2a1d7f9ad2bf39179a360a6a87ae0eebfab4ac/licenses)) | 59 ADRs with a "Status" section | 20,419 / 49,436 / 18,986 | 3.3 M | ADRs record feature decisions for 8.10 and 8.11, not a modernization. License needs a legal check |
| [openssl/openssl](https://github.com/openssl/openssl) | Apache-2.0 | 54 design documents in `doc/designs`, no status | 11,959 / 19,958 / 8 | 1.3 M | Merges happen outside the GitHub merge button, so pull requests show as closed, not merged |
| [home-assistant/core](https://github.com/home-assistant/core) + [architecture](https://github.com/home-assistant/architecture) | Apache-2.0; the architecture repo has no license file | 22 ADRs, for example [0021](https://github.com/home-assistant/architecture/blob/5438568da4ed57c9c88b08e0c4c5782500d0be41/adr/0021-YAML-integration-configuration-deprecation-policy.md) (YAML configuration deprecation) | 76,643 / 115,897 / 20,543 | 4.1 M | Few ADRs for the size, unlicensed ADR repo |
| [FreeCAD/FreeCAD](https://github.com/FreeCAD/FreeCAD) | LGPL-2.1 | 6 ADRs, only in `src/Mod/CAM/Roadmap/ADR` | 13,632 / 20,244 / 3,827 | 2.4 M | Records cover one module |
| [saltstack/salt](https://github.com/saltstack/salt) | Apache-2.0 | 9 RFCs in `rfcs/`; the proposals repo is archived | 27,548 / 42,580 / 1,175 | 1.1 M | Few records, no active decision process |

### Tier 3: large legacy code without decision records

These repositories match the common client case: a lot of old code and no written decisions. Each needs synthetic decision records in a companion repository, as Thunderbird does in #8.

| Repository | License | Evidence of legacy or modernization in the repo | Issues / PRs / merged | Lines |
|---|---|---|---|---|
| [openemr/openemr](https://github.com/openemr/openemr) | GPL-3.0 | Legacy folders frozen, new code goes to `src/` (see below) | 4,818 / 10,601 / 3,887 | 2.3 M by bytes, about 1.2 M hand-written |
| [Dolibarr/dolibarr](https://github.com/Dolibarr/dolibarr) | GPL-3.0 | No modernization document found. The [ChangeLog](https://github.com/Dolibarr/dolibarr/blob/41e3047b94059a4c5a49a4aa0434c14032b378ec/ChangeLog) records small cleanups, for example 41 per-module note pages folded into one in 25.0 | 10,964 / 32,241 / 4,466 | 2.6 M |
| [keycloak/keycloak](https://github.com/keycloak/keycloak) | Apache-2.0 | Moved from WildFly to Quarkus. "Starting with Keycloak 17 the default distribution is now powered by Quarkus" ([migration guide](https://www.keycloak.org/migration/migrating-to-quarkus)) | 21,420 / 27,369 / 4,082 | 1.7 M |
| [E3SM-Project/E3SM](https://github.com/E3SM-Project/E3SM) | BSD-3-Clause except separable pieces ([LICENSE](https://github.com/E3SM-Project/E3SM/blob/9b057515aabbcbf241481b3b088b11d9836e1ef4/LICENSE)) | Fortran atmosphere physics ported to C++. The [EAMxx README](https://github.com/E3SM-Project/E3SM/blob/9b057515aabbcbf241481b3b088b11d9836e1ef4/components/eamxx/README.md) calls the Fortran modules "legacy" and says they are used "mostly for comparisons with the newer C++ implementations". Project docs are on Confluence | 2,738 / 6,254 / 715 | 3.0 M |
| [eclipse-ee4j/glassfish](https://github.com/eclipse-ee4j/glassfish) | EPL-2.0 or GPL-2.0 with Classpath Exception | The [README](https://github.com/eclipse-ee4j/glassfish/blob/cc501faf8dc003831f136d8e179dcb20285ab87a/README.md) version table runs from 5.1 (Java EE 8) to 9.0 (Jakarta EE 12). Jakarta EE 9 is the release for "migration to the new namespace" ([Jakarta EE 9](https://jakarta.ee/release/9/)) | 23,422 / 2,841 / 363 | 2.1 M |
| [nextcloud/server](https://github.com/nextcloud/server) | AGPL-3.0 | Fork of ownCloud: the README carries "2013-2016 ownCloud, Inc." and [`lib/private/legacy`](https://github.com/nextcloud/server/tree/7a89690a1ca17aae3113da13070487c191eb1f4b/lib/private/legacy) still holds 7 `OC_*` classes | 21,901 / 42,926 / 7,108 | 1.4 M |
| [magento/magento2](https://github.com/magento/magento2) | OSL-3.0 | Not checked | 25,899 / 15,675 / 126 | 3.5 M; low activity |
| [ADempiere/adempiere](https://github.com/ADempiere/adempiere) | GPL-2.0 | Not checked | 2,820 / 1,666 / 43 | 1.6 M; low activity |
| [odoo/odoo](https://github.com/odoo/odoo) | Not classified by GitHub | Not checked | 19,413 / 283,048 / 2 | 3.4 M; merges happen outside GitHub, so the pull request history is unusable |

### 1. shopware/shopware

The repository describes Shopware 6 as "an open commerce platform based on Symfony Framework and Vue". The GitHub repository was created in 2018. At the pinned commit the tree has 29,575 files. Counted lines:

| Language | Production | Tests |
|---|---|---|
| PHP | 623,000 | 925,000 |
| JavaScript | 217,000 | 303,000 |
| TypeScript | 130,000 | 72,000 |
| Twig | 137,000 | - |
| SCSS | 49,000 | - |

The ADRs carry front matter with `title`, `date`, `area` and `tags`, but no status. Migrations found in the code and their ADRs:

| Migration | ADR | State in the code |
|---|---|---|
| Vuex to Pinia | [2024-06-17-replace-vuex-with-pinia](https://github.com/shopware/shopware/blob/470e9fe474ed51d3dad5a48328b89237e74ed2a6/adr/2024-06-17-replace-vuex-with-pinia.md) | Done: 0 files use `Shopware.State`, 366 use `Shopware.Store` |
| XML to PHP container configuration | [2026-07-30-migrate-container-configuration-from-xml-to-php](https://github.com/shopware/shopware/blob/470e9fe474ed51d3dad5a48328b89237e74ed2a6/adr/2026-07-30-migrate-container-configuration-from-xml-to-php.md) | Done: 0 XML files, 164 PHP files |
| `sw-` components to Meteor `mt-` components | [2024-03-21-implementation-of-meteor-component-library](https://github.com/shopware/shopware/blob/470e9fe474ed51d3dad5a48328b89237e74ed2a6/adr/2024-03-21-implementation-of-meteor-component-library.md) | In progress: 103 `<sw-button>` against 752 `<mt-button>` |
| Vue 2 Options API to Vue 3 Composition API | [2024-10-02-vue-2-options-api-to-vue-3-composition-api-conversion-codemod](https://github.com/shopware/shopware/blob/470e9fe474ed51d3dad5a48328b89237e74ed2a6/adr/2024-10-02-vue-2-options-api-to-vue-3-composition-api-conversion-codemod.md) | Not measured |
| Administration JavaScript to TypeScript | None found. [2022-06-24-add-typescript-support-for-storefront-js](https://github.com/shopware/shopware/blob/470e9fe474ed51d3dad5a48328b89237e74ed2a6/adr/2022-06-24-add-typescript-support-for-storefront-js.md) covers the storefront only | In progress: 989 components with `index.js`, 268 with `index.ts` |
| Deprecations removed in major versions | [2022-01-20-feature-flags-for-major-versions](https://github.com/shopware/shopware/blob/470e9fe474ed51d3dad5a48328b89237e74ed2a6/adr/2022-01-20-feature-flags-for-major-versions.md), which supersedes [2021-01-21-deprecation-strategy](https://github.com/shopware/shopware/blob/470e9fe474ed51d3dad5a48328b89237e74ed2a6/adr/_superseded/2021-01-21-deprecation-strategy.md) | 806 `@deprecated` tags in production PHP before the 6.8 major |

These pairs give ground truth for questions such as "How far is the Meteor migration?" or "Which ADR replaced the deprecation strategy?". The superseded ADR names its successor in a heading.

Issues and pull requests: 10,790 issues and 12,146 pull requests in total. Since 2024 there are 9,232 issues and 10,200 pull requests, 99 of them from Dependabot. In the last 12 months 3,768 issues were opened and 4,556 pull requests merged. The repository has 5,839 changelog files. About 5,000 of their `issue:` fields hold Jira `NEXT-` IDs and about 150 hold GitHub numbers. Questions about work before 2024 will hit references that the project memory cannot follow. That is realistic for clients who moved trackers.

Weaknesses:

- No status field. A status test needs the `_superseded` folder as the status, or synthetic records with a status.
- Only 3 `de-DE` files in the tree. German content must come from the synthetic workshops.
- Indexing the 2024-onward issues and pull requests means about 19,300 items, five times Thunderbird's subset.

### 2. kubernetes/kubernetes with kubernetes/enhancements

The code repository has about 107 MB of Go. Its [`.gitattributes`](https://github.com/kubernetes/kubernetes/blob/539fda3eec68e131a4732e03d200bc2e9f96d7d0/.gitattributes) marks generated files as `linguist-generated`, so GitHub leaves them out of the byte count. The license is [Apache-2.0](https://github.com/kubernetes/kubernetes/blob/539fda3eec68e131a4732e03d200bc2e9f96d7d0/LICENSE).

Decision records are KEPs in [`kubernetes/enhancements`](https://github.com/kubernetes/enhancements/tree/cac930cb9d7c70192016865425f943c339414011/keps) (pinned at `cac930c`, 2026-11-20). Each KEP is a folder with a `README.md` and a `kep.yaml`. The [template](https://github.com/kubernetes/enhancements/blob/cac930cb9d7c70192016865425f943c339414011/keps/NNNN-kep-template/kep.yaml) allows `provisional|implementable|implemented|deferred|rejected|withdrawn|replaced`. The 686 KEPs have these statuses: 304 implemented, 296 implementable, 63 provisional, 11 withdrawn, 6 replaced, 3 rejected, 1 superseded, 1 removed and 1 deferred.

Three KEPs remove legacy code from the core:

- [KEP-2395](https://github.com/kubernetes/enhancements/blob/cac930cb9d7c70192016865425f943c339414011/keps/sig-cloud-provider/2395-removing-in-tree-cloud-providers/kep.yaml) "Removing In-Tree Cloud Providers", created 2018-12-18, status `implemented`, stable in v1.31.
- [KEP-625](https://github.com/kubernetes/enhancements/blob/cac930cb9d7c70192016865425f943c339414011/keps/sig-storage/625-csi-migration/kep.yaml) "In-tree Storage Plugin to CSI Migration", status `implementable`, stable in v1.25. Nine more KEPs in `sig-storage` cover the migration of single plugins.
- [KEP-2221](https://github.com/kubernetes/enhancements/blob/cac930cb9d7c70192016865425f943c339414011/keps/sig-node/2221-remove-dockershim/kep.yaml) "Removing dockershim from kubelet", status `implemented`, stable in v1.24.

KEP-625 shows a problem for status questions: the migration shipped as stable, but the record still says `implementable`.

Weaknesses: at 3.1 million lines and 143,000 issues and pull requests it is larger than the target size. The decision records sit in a second repository, which needs the two-repository setup that #8 left open.

### 3. python/cpython with python/peps

CPython's [LICENSE](https://github.com/python/cpython/blob/e0d0f2e153450cf79b84e2ec5ffbcbad8aea7922/LICENSE) is the Python Software Foundation License Version 2, with the Zero-Clause BSD license for code in the documentation. GitHub does not classify it. 715 of the 748 PEP files say they are in the public domain or under CC0-1.0.

The PEPs in [`python/peps`](https://github.com/python/peps/tree/4434e728a7db24540d581f52e96570ca1230356e/peps) (pinned at `4434e72`, 2026-11-19) have a `Status:` header: 375 Final, 131 Rejected, 74 Withdrawn, 57 Draft, 38 Active, 35 Deferred, 25 Superseded, 11 Accepted, 1 Provisional and 1 April Fool. This is the widest status mix of all candidates.

PEPs that change legacy parts of CPython:

- [PEP 703](https://github.com/python/peps/blob/4434e728a7db24540d581f52e96570ca1230356e/peps/pep-0703.rst) "Making the Global Interpreter Lock Optional in CPython", Final, Python 3.13. [PEP 779](https://github.com/python/peps/blob/4434e728a7db24540d581f52e96570ca1230356e/peps/pep-0779.rst) sets the criteria for supported status, Final, Python 3.14.
- [PEP 744](https://github.com/python/peps/blob/4434e728a7db24540d581f52e96570ca1230356e/peps/pep-0744.rst) "JIT Compilation", Informational, still Draft.
- [PEP 594](https://github.com/python/peps/blob/4434e728a7db24540d581f52e96570ca1230356e/peps/pep-0594.rst) "Removing dead batteries from the standard library", Final, Python 3.11.

Issues moved from bugs.python.org to GitHub. [PEP 581](https://github.com/python/peps/blob/4434e728a7db24540d581f52e96570ca1230356e/peps/pep-0581.rst) gives the reasons and [PEP 588](https://github.com/python/peps/blob/4434e728a7db24540d581f52e96570ca1230356e/peps/pep-0588.rst) says "The migration was carried out in April 2022". The 79,355 GitHub issues include the migrated history.

Weaknesses: it is a language runtime, not a business system. The decision records are in a second repository and are reStructuredText, not Markdown. Many PEPs are language features, not modernization.

### 4. gradle/gradle

The 12 ADRs in [`architecture/standards`](https://github.com/gradle/gradle/tree/5956d9736e56e4050fabb6b3566f0946d6613a9f/architecture/standards) (pinned at `5956d97`, 2026-11-20) each have a "Status" section with a date. 11 are accepted and [ADR-0007](https://github.com/gradle/gradle/blob/5956d9736e56e4050fabb6b3566f0946d6613a9f/architecture/standards/0007-java-pre-requisite.md) is proposed. [ADR-0004](https://github.com/gradle/gradle/blob/5956d9736e56e4050fabb6b3566f0946d6613a9f/architecture/standards/0004-use-a-platform-architecture.md) states the legacy problem: "The Gradle code base is essentially a large monolith, without strong internal boundaries." Other ADRs keep Groovy types out of the public API ([0003](https://github.com/gradle/gradle/blob/5956d9736e56e4050fabb6b3566f0946d6613a9f/architecture/standards/0003-avoid-introducing-Groovy-types-to-public-api.md)) and move to lazy provider APIs ([0006](https://github.com/gradle/gradle/blob/5956d9736e56e4050fabb6b3566f0946d6613a9f/architecture/standards/0006-use-of-provider-apis-in-gradle.md)).

The survey's folder match missed these ADRs because the folder is named `standards`. They were found by hand.

Weaknesses: 12 ADRs is few for 2.4 million lines, and none is superseded or rejected. Groovy is the largest language by bytes (40 MB). The share of tests in it was not measured.

### OpenEMR, the recommended third repository

OpenEMR is an electronic health record and practice management system. The GitHub repository was created in 2010. Its [CONTRIBUTING.md](https://github.com/openemr/openemr/blob/48c71737f40ae13f808336d41d63d6a8cda20c54/CONTRIBUTING.md) (pinned at `48c7173`, 2026-11-20) freezes the legacy code: "The `interface/`, `portal/` and `library/` directories, including `interface/modules/`, are frozen." New logic goes into classes under `src/`. A bug fix in a legacy file moves the changed code into `src/`. Pull requests that grow the frozen folders without moving at least as much code out may be closed. Issue [#14764](https://github.com/openemr/openemr/issues/14764) "ratchet legacy directories so new code lands in src/" (opened 2026-10-21) gives the reasons. This is a strangler pattern written down as a rule, which is close to what consultants propose to clients.

PHP in the tree at the pinned commit:

| Folder | Files | Size | About |
|---|---|---|---|
| `interface/`, `library/`, `portal/` (frozen) | 1,505 | 16.5 MB | 470,000 lines |
| `src/` without `src/FHIR` | 1,621 | 11.4 MB | 330,000 lines |
| `src/FHIR`, mostly classes in `src/FHIR/R4` generated with PHPFHIR | 920 | 13.5 MB | not hand-written |
| `.phpstan/baseline/` (PHPStan baseline) | 172 | 16.7 MB | not hand-written |
| `tests/` | 1,278 | 11.1 MB | 320,000 lines |

GitHub counts 70.9 MB of PHP, which gives 2.3 million lines by the byte method. The whole tree holds 71.3 MB of PHP, so GitHub's count includes the baseline, although the [`.gitattributes`](https://github.com/openemr/openemr/blob/48c71737f40ae13f808336d41d63d6a8cda20c54/.gitattributes) marks `.phpstan/baseline/` as `linguist-generated`. Without the generated FHIR classes and the baseline, the hand-written PHP is about 41 MB, which is about 1.2 million lines including tests and about 0.85 million without.

The tree has no decision records. Synthetic ones in a companion repository can build on the freeze rule, issue #14764 and the FHIR API. The issues contain health care questions from users, so the GDPR note of #8 applies here too.

## Excluded

- [elastic/kibana](https://github.com/elastic/kibana) (about 15 million lines) and [apache/netbeans](https://github.com/apache/netbeans) (about 11 million lines): too large.
- [camunda/camunda-bpm-platform](https://github.com/camunda/camunda-bpm-platform): archived.
- [SuiteCRM/SuiteCRM-Core](https://github.com/SuiteCRM/SuiteCRM-Core) (1.5 million lines, 57 merged pull requests in 12 months) and [salesagility/SuiteCRM](https://github.com/salesagility/SuiteCRM) (1.2 million lines, none merged): low activity.
- Above 1 million lines, but no decision-record folder and no modernization story checked: [getsentry/sentry](https://github.com/getsentry/sentry) (3.7 M), [opensearch-project/OpenSearch](https://github.com/opensearch-project/OpenSearch) (3.6 M), [grafana/grafana](https://github.com/grafana/grafana) (3.4 M), [metabase/metabase](https://github.com/metabase/metabase) (3.2 M), [mattermost/mattermost](https://github.com/mattermost/mattermost) (2.3 M), [apache/superset](https://github.com/apache/superset) (2.1 M), [neovim/neovim](https://github.com/neovim/neovim) (1.2 M) and [asterisk/asterisk](https://github.com/asterisk/asterisk) (1.1 M).
- Below 1 million lines: thunderbird-android (0.37 M), [backstage/backstage](https://github.com/backstage/backstage) (0.97 M), [matomo-org/matomo](https://github.com/matomo-org/matomo) (0.82 M), [owncloud/core](https://github.com/owncloud/core) (0.80 M), [JabRef/jabref](https://github.com/JabRef/jabref) (0.55 M), [joomla/joomla-cms](https://github.com/joomla/joomla-cms) (0.52 M), [owncloud/ocis](https://github.com/owncloud/ocis) (0.51 M), [jenkinsci/jenkins](https://github.com/jenkinsci/jenkins) (0.44 M), [microsoft/terminal](https://github.com/microsoft/terminal) (0.39 M), [CMSgov/beneficiary-fhir-data](https://github.com/CMSgov/beneficiary-fhir-data) (0.38 M), [eclipse-sw360/sw360](https://github.com/eclipse-sw360/sw360) (0.27 M), [ManageIQ/manageiq](https://github.com/ManageIQ/manageiq) (0.23 M) and [LedgerSMB/LedgerSMB](https://github.com/ledgersmb/LedgerSMB) (0.17 M). LedgerSMB is the best small fit: 43 ADRs in `doc/adr` and a clear legacy story.

No large open-source COBOL project was found. [aws-samples/aws-mainframe-modernization-carddemo](https://github.com/aws-samples/aws-mainframe-modernization-carddemo) is small (see #8). A web search found no curated list of large legacy open-source projects. It found benchmarks instead: the [X-COBOL](https://arxiv.org/abs/2306.04892) dataset of 84 COBOL repositories, [Legacy-Bench](https://www.factory.ai/news/legacy-bench) and [LegacyCodeBench](https://kalmantic.github.io/legacycodebench).

## Method and limits

A script queried 50 repositories through the GitHub GraphQL API on 2026-11-20: license, archive flag, last push, the six largest languages by bytes, and issue and pull request totals. It counted merged pull requests with the search query `is:pr is:merged merged:>=2025-11-20`. It counted decision records as `.md`, `.rst`, `.adoc` and `.txt` files in folders named `adr`, `decision`, `rfc`, `kep`, `pep`, `design`, `design-doc` or `proposal` (singular or plural), from the recursive git tree.

Limits:

- Lines are GitHub's language bytes divided by 35 bytes per line, and include tests. GitHub leaves out vendored files and files marked `linguist-generated`, but counts generated code that is not marked. For Shopware the estimate (2.56 million) is within 4% of the counted lines (2.46 million). For OpenEMR it is almost twice the hand-written size.
- The six largest languages include markup such as HTML, which adds a little to some estimates.
- The folder match misses records in other folders, as with Gradle's `architecture/standards`. The trees of netbeans, gradle, kibana and magento were truncated by the API.
- A low merge count can mean that a project merges outside the GitHub button (OpenSSL, Odoo), not that it is inactive.
- Kubernetes KEP and PEP statuses were counted in shallow clones of the two record repositories. The Shopware line counts and migration counts come from a shallow clone at the pinned commit.
- Licenses were read from the license files, not checked by a lawyer. Camunda's split license and the missing license in `home-assistant/architecture` need a check before use.

## Sources

Repository metadata (GitHub GraphQL and REST API, 2026-11-20): `https://api.github.com/repos/<owner>/<repo>` for each repository linked above. Merged pull requests: `https://api.github.com/search/issues?q=repo:<owner>/<repo>+is:pr+is:merged+merged:>=2025-11-20`.

Decision records, license files and documentation are linked inline, pinned to the commits named in each section.
