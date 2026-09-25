# Software requirements specification: multilingual Grafana documentation

| Field    | Value                                                           |
| -------- | --------------------------------------------------------------- |
| Document | SRS-DOCS-I18N-001                                               |
| Status   | Draft for review                                                |
| Scope    | `docs/` directory of the `grafana` repository and its toolchain |
| Date     | 2026-09-25                                                      |

## 1. Introduction

This section states why the specification exists, what it covers, and the terms it uses.

### 1.1 Purpose

This document specifies the requirements for converting the single-language (US English) Grafana documentation in `docs/` into a multilingual structure. The multilingual structure contains complete, translated Markdown files for every language that the Grafana UI supports in `public/locales/`, and builds with Hugo's native multilingual (i18n) features.

The intended readers are documentation maintainers, the docs platform team that owns `grafana/website` and the `grafana/docs-base` image, the localization team that operates Crowdin, and CI and release engineers.

### 1.2 Scope

In scope:

- Restructuring `docs/sources/` into per-language content trees.
- Producing and committing translated content files for all supported languages.
- Hugo configuration (languages, module mounts, i18n string tables) required to build the multilingual site.
- Translation pipeline integration (Crowdin), synchronization, and staleness tracking.
- Changes to local build tooling (`docs/Makefile`, `docs/docs.mk`, `docs/make-docs`, `docs/variables.mk`) and CI workflows that reference `docs/sources`.
- Validation, quality, and SEO requirements for the translated output.

Out of scope:

- Translating the Grafana application UI (already handled by `public/locales/` and Crowdin).
- Right-to-left (RTL) layout work, because no RTL locale is currently in `public/locales/`. The design must not preclude it (refer to NFR-EXT-2).
- Localized screenshots and videos. Media is shared across languages in the first release.
- Documentation for other Grafana products hosted on grafana.com.

### 1.3 Definitions

| Term               | Definition                                                                                                                           |
| ------------------ | ------------------------------------------------------------------------------------------------------------------------------------ |
| Locale             | A BCP 47 language tag as used by the directory names in `public/locales/`, for example `de-DE`, `zh-Hans`.                           |
| Source language    | US English (`en-US`). All translations derive from it.                                                                               |
| Target language    | Any supported locale other than `en-US`.                                                                                             |
| Hugo language key  | The key under `languages` in Hugo configuration. Hugo treats it case-insensitively; this spec uses the lowercase form of the locale. |
| Translation unit   | A single Markdown page (`*.md`) and its translatable front matter.                                                                   |
| Source revision    | The Git blob SHA of the English file that a translation was produced from.                                                           |
| Stale translation  | A translated file whose recorded source revision differs from the current English file's blob SHA.                                   |
| Shared content     | Files under `docs/sources/shared/` that are included in other pages through the `docs/shared` shortcode.                             |
| docs-base          | The `grafana/docs-base` container image that contains Hugo, the website layouts, and shortcodes, used by `make docs` and CI.         |
| Website repository | `grafana/website`, which receives synced docs content and builds grafana.com.                                                        |

### 1.4 References

- `docs/README.md` — Local build, content guidelines, redirects, publishing.
- `docs/AGENTS.md` — Documentation style guide.
- `packages/grafana-i18n/src/constants.ts` and `packages/grafana-i18n/src/languages.ts` — Canonical list of supported locales and their native names.
- `crowdin.yml` and `.github/workflows/i18n-crowdin-*.yml` — Existing UI translation pipeline.
- `.github/workflows/lint-build-docs.yml`, `documentation-ci.yml`, `deploy-pr-preview.yml` — Docs CI.
- Hugo documentation: [Multilingual mode](https://gohugo.io/content-management/multilingual/), [Configure languages](https://gohugo.io/configuration/languages/), [Module mounts](https://gohugo.io/configuration/module/#mounts), [`i18n` function](https://gohugo.io/functions/lang/translate/).

## 2. Overall description

This section summarizes the current state, the target state, and the constraints that shape the requirements.

### 2.1 Current state

A baseline survey of the repository on the date of this document found:

| Item                                   | Current value                                                                                                                               |
| -------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------- |
| Content root                           | `docs/sources/` (single language, English)                                                                                                  |
| Markdown pages                         | 773 files, of which 92 are under `docs/sources/shared/`                                                                                     |
| Volume                                 | Approximately 880,000 words including front matter and code blocks                                                                          |
| Non-Markdown assets in `docs/sources/` | 2 files (one SVG, one JSON). Images are almost entirely hosted on grafana.com.                                                              |
| Most used shortcodes                   | `admonition` (~1,000), `figure` (~730), `docs/shared` (~365), `youtube` (~116)                                                              |
| Version-templated links                | ~3,000 occurrences of `/docs/grafana/<GRAFANA_VERSION>/...`                                                                                 |
| Pages using `refs` front matter        | 154                                                                                                                                         |
| Pages using `aliases`                  | 448                                                                                                                                         |
| Generated pages                        | `visualizations/panels-visualizations/query-transform-data/transform-data/index.md` generated by `scripts/docs/generate-transformations.ts` |
| Publish path                           | `docs/sources/` synced to `grafana/website` at `content/docs/grafana/{next,latest}`                                                         |
| Local build                            | `make docs` mounts `docs/sources` at `/hugo/content/docs/grafana/latest` in docs-base                                                       |

### 2.2 Supported languages

The set of documentation languages must equal the set of UI languages declared in `packages/grafana-i18n/src/languages.ts`, which matches the locale directories in `public/locales/`. The `public/locales/enterprise/` directory holds i18next configuration and is not a locale. The `pseudo` locale is a UI test locale and is excluded.

| Locale    | Hugo language key | Native name (`languageName`) | URL prefix  | Weight |
| --------- | ----------------- | ---------------------------- | ----------- | ------ |
| `en-US`   | `en-us`           | English                      | _(none)_    | 1      |
| `cs-CZ`   | `cs-cz`           | Čeština                      | `/cs-cz/`   | 2      |
| `de-DE`   | `de-de`           | Deutsch                      | `/de-de/`   | 3      |
| `es-ES`   | `es-es`           | Español                      | `/es-es/`   | 4      |
| `fr-FR`   | `fr-fr`           | Français                     | `/fr-fr/`   | 5      |
| `hu-HU`   | `hu-hu`           | Magyar                       | `/hu-hu/`   | 6      |
| `id-ID`   | `id-id`           | Bahasa Indonesia             | `/id-id/`   | 7      |
| `it-IT`   | `it-it`           | Italiano                     | `/it-it/`   | 8      |
| `ja-JP`   | `ja-jp`           | 日本語                       | `/ja-jp/`   | 9      |
| `ko-KR`   | `ko-kr`           | 한국어                       | `/ko-kr/`   | 10     |
| `nl-NL`   | `nl-nl`           | Nederlands                   | `/nl-nl/`   | 11     |
| `pl-PL`   | `pl-pl`           | Polski                       | `/pl-pl/`   | 12     |
| `pt-BR`   | `pt-br`           | Português Brasileiro         | `/pt-br/`   | 13     |
| `pt-PT`   | `pt-pt`           | Português                    | `/pt-pt/`   | 14     |
| `ru-RU`   | `ru-ru`           | Русский                      | `/ru-ru/`   | 15     |
| `sv-SE`   | `sv-se`           | Svenska                      | `/sv-se/`   | 16     |
| `tr-TR`   | `tr-tr`           | Türkçe                       | `/tr-tr/`   | 17     |
| `zh-Hans` | `zh-hans`         | 中文（简体）                 | `/zh-hans/` | 18     |
| `zh-Hant` | `zh-hant`         | 中文（繁體）                 | `/zh-hant/` | 19     |

This yields 19 target languages. At the baseline size, full coverage means 773 × 19 = 14,687 translated files and roughly 16.7 million translated words.

### 2.3 Target state overview

```text
docs/
├── Makefile, docs.mk, make-docs, variables.mk
├── hugo/
│   ├── config/_default/languages.toml      # generated from languages.ts (FR-CFG-1)
│   ├── config/_default/module.toml         # per-language content mounts (FR-CFG-3)
│   └── i18n/<hugo-lang-key>.toml           # UI string tables (FR-I18N-1)
├── i18n/
│   ├── glossary/<locale>.tbx               # terminology (FR-TM-5)
│   └── status.json                         # coverage/staleness report (FR-SYNC-4)
└── sources/
    ├── en-US/                              # former docs/sources/* (source language)
    │   ├── _index.md
    │   ├── alerting/...
    │   └── shared/...
    ├── de-DE/                              # mirrored tree, translated files
    │   ├── _index.md
    │   ├── alerting/...
    │   └── shared/...
    └── <locale>/...                        # one tree per locale in section 2.2
```

Hugo's _translation by content directory_ model is used: each language has its own content root, and pages that share the same path relative to their root are linked as translations of each other.

### 2.4 Constraints

- C-1: The website layouts, shortcodes, and top-level Hugo configuration live in `grafana/website` and `grafana/docs-base`, not in this repository. Requirements marked **[EXT]** need changes there.
- C-2: `docs/make-docs` and `docs/docs.mk` are vendored from `grafana/writers-toolkit` and must be changed upstream first, not edited locally only.
- C-3: The English content must remain publishable at its existing URLs throughout the migration.
- C-4: Crowdin is the established translation management system for Grafana and must be reused rather than introducing a second system.
- C-5: Documentation is versioned per release branch. Translations are versioned with the English content on the same branch.

### 2.5 Assumptions and dependencies

- A-1: The Hugo version in docs-base supports multilingual mode with per-language module mounts (`lang` on a content mount) and `languages.<key>.contentDir`. The implementation must confirm the version and pin a minimum in the docs-base image.
- A-2: Funding and vendor capacity exist for initial translation of ~16.7M words, or the organization accepts machine translation with human post-editing (refer to FR-TM-3).
- A-3: The website repository can accept a multilingual site configuration without breaking other hosted products, or can host Grafana docs languages as a separate build.

## 3. Functional requirements

Requirements use MUST, SHOULD, and MAY as defined in RFC 2119. Each has a unique identifier for traceability to the acceptance criteria in section 6.

### 3.1 Directory structure

- **FR-DIR-1** The repository MUST move all content from `docs/sources/` into `docs/sources/en-US/`, preserving relative paths, using `git mv` so that history is kept.
- **FR-DIR-2** The repository MUST contain one directory `docs/sources/<locale>/` for every locale in section 2.2, named exactly as the corresponding `public/locales/<locale>/` directory.
- **FR-DIR-3** Every Markdown file under `docs/sources/en-US/` MUST have a counterpart at the same relative path in every target-language tree. This includes `_index.md` section files and files under `shared/`.
- **FR-DIR-4** A target-language tree MUST NOT contain Markdown files that have no English counterpart, unless the file sets `translationKey` to link it to an English page at a different path (refer to FR-HUGO-4).
- **FR-DIR-5** Non-Markdown assets (the SVG and JSON under `docs/sources/`) MUST stay in `docs/sources/en-US/` only and MUST be resolvable from translated pages (refer to FR-CONT-7). They MUST NOT be copied into every language tree.
- **FR-DIR-6** File and directory names MUST NOT be translated. URL slugs stay identical across languages, so that the language switcher and redirects map one to one.

### 3.2 Hugo configuration

- **FR-CFG-1** A `languages` configuration MUST define every language in section 2.2 with `languageCode` (the locale), `languageName` (the native name), `weight`, `languageDirection: ltr`, and `contentDir` or an equivalent content mount.
- **FR-CFG-2** The `languages` configuration MUST be generated from `packages/grafana-i18n/src/languages.ts` by a script (for example `scripts/docs/generate-hugo-languages.ts`), not maintained by hand. CI MUST fail if the generated file is out of date.
- **FR-CFG-3** Content MUST be exposed to Hugo through module mounts of the form `source: docs/sources/<locale>`, `target: content/docs/grafana/<version>`, `lang: <hugo-lang-key>`, so the project mount point stays the same for all languages.
- **FR-CFG-4** `defaultContentLanguage` MUST be `en-us` and `defaultContentLanguageInSubdir` MUST be `false`, so English URLs remain unchanged (for example `/docs/grafana/latest/alerting/`). **[EXT]**
- **FR-CFG-5** Target-language URLs MUST be prefixed with the Hugo language key, for example `/de-de/docs/grafana/latest/alerting/`. **[EXT]**
- **FR-CFG-6** The site MUST NOT enable Hugo's automatic fallback to English content for missing pages in production builds. Missing translations are a build error per FR-VAL-1. A development-only flag MAY allow fallback to speed up local previews.

### 3.3 Hugo page linkage and templates

- **FR-HUGO-1** Pages at the same relative path in different language trees MUST be linked as translations by Hugo (available as `.Translations` and `.AllTranslations`).
- **FR-HUGO-2** Every page MUST render a language switcher listing all available translations of that page, using `languageName`. **[EXT]**
- **FR-HUGO-3** Every page MUST emit `<html lang="…">` with the locale, and `<link rel="alternate" hreflang="…">` for each translation plus `hreflang="x-default"` pointing at English. **[EXT]**
- **FR-HUGO-4** A translated page MAY set `translationKey` only when it intentionally diverges in path; otherwise the path-based linkage MUST be used.
- **FR-HUGO-5** The generated sitemap MUST include all languages, either as a sitemap index with one sitemap per language or as a single sitemap with `xhtml:link` alternates. **[EXT]**
- **FR-HUGO-6** Section menus (built from the file structure and `weight`) MUST render in each language using the translated `title` or `menuTitle`, with the same order as English.

### 3.4 Hugo i18n string tables

- **FR-I18N-1** Strings rendered by layouts and shortcodes, not by content, MUST be translatable through Hugo's `i18n` function using string tables at `i18n/<hugo-lang-key>.toml`. This includes, at minimum: admonition titles (Note, Caution, Warning, Tip), "On this page", "Edit this page", "Was this page helpful?", breadcrumbs root, search placeholder, version selector labels, "Previous" and "Next", and the untranslated or outdated notices in FR-SYNC-5. **[EXT]**
- **FR-I18N-2** The English string table MUST be the source for Crowdin and the target tables MUST be produced by Crowdin (refer to FR-TM-1).
- **FR-I18N-3** Every key present in the English table MUST be present in every target table. Hugo's `enableMissingTranslationPlaceholders` MUST be enabled in CI builds so missing keys are detectable.
- **FR-I18N-4** Shortcodes that take visible text as parameters (for example `admonition` title override, `figure` `caption`, `collapse` `title`, `section` labels) MUST render the text from the page as-is. Translators translate those parameters in content, not in string tables.

### 3.5 Content handling rules

The following rules define what translators and machine translation MUST and MUST NOT change inside a translated file.

- **FR-CONT-1** Translatable front matter keys: `title`, `menuTitle` (and legacy `menutitle`), `description`, `keywords`. All other front matter keys MUST be copied verbatim from English, including `aliases`, `weight`, `labels`, `refs`, `canonical`, `review_date`, `headless`, `type`, and any structured data.
- **FR-CONT-2** The H1 heading MUST equal the translated front matter `title`, following the existing style rule in `docs/AGENTS.md`.
- **FR-CONT-3** Fenced and inline code, configuration keys, CLI commands, API paths and payloads, HTTP headers, environment variables, file paths, query languages (PromQL, LogQL, SQL, and so on), and Go template syntax MUST NOT be translated.
- **FR-CONT-4** Hugo shortcode names and non-visible parameters (for example `lookup`, `source`, `version`, `src`, `type`, `id`) MUST NOT be changed. Only visible text parameters (FR-I18N-4) and inner content MAY be translated.
- **FR-CONT-5** Heading anchors used as link targets MUST remain stable across languages. Every translated heading that is the target of an in-page or cross-page anchor MUST carry an explicit anchor ID equal to the English auto-generated ID, for example `## Einrichtung {#set-up}`. A script MUST add these IDs automatically during import from Crowdin.
- **FR-CONT-6** Names of UI elements referenced in docs (menus, buttons, field labels) MUST match the translated UI strings in `public/locales/<locale>/grafana.json` for that language. Where a UI string is not translated, the English label MUST be kept.
- **FR-CONT-7** Image and media references (`figure`, `youtube`, `video-embed`, Markdown images) MUST keep their English source URLs. Alt text and captions MUST be translated.
- **FR-CONT-8** Product names and trademarks (for example Grafana, Grafana Cloud, Grafana Alloy, Loki, Prometheus, OpenTelemetry, Kubernetes) MUST NOT be translated, and MUST follow the naming rules in `docs/AGENTS.md`.
- **FR-CONT-9** `{{< docs/ignore >}}` blocks MUST be preserved with their English content unless the block contains prose meant for readers.

### 3.6 Links, shared content, and redirects

- **FR-LINK-1** Links written as `/docs/grafana/<GRAFANA_VERSION>/...` and `ref:` destinations in `refs` front matter MUST resolve to the same-language page when rendered in a target language, and to English when no same-language page exists. The resolution MUST be implemented in the link render hook and `ref` handling, not by rewriting every link in content. **[EXT]**
- **FR-LINK-2** Links to other products (`/docs/grafana-cloud/`, `/docs/loki/`, external sites) MUST remain unchanged unless the destination product also publishes that language.
- **FR-LINK-3** The `docs/shared` shortcode MUST look up the shared file in the current page's language first (`<locale>/shared/<lookup>`), and fall back to `en-US/shared/<lookup>` only when the language file is missing. **[EXT]**
- **FR-LINK-4** `aliases` in translated pages MUST produce redirects scoped to the language prefix, for example `/de-de/docs/grafana/latest/old-path/` to `/de-de/docs/grafana/latest/new-path/`. The implementation MUST verify Hugo's alias behavior in multilingual mode and add a build step if language-scoped aliases are not generated.
- **FR-LINK-5** A translated page's `canonical` front matter, when present in English, MUST be adjusted at render time to the equivalent URL in that language, or omitted for the translated page so Hugo's default canonical applies. **[EXT]**
- **FR-LINK-6** Hugo's `refLinksErrorLevel` in CI builds MUST report broken references per language; the existing `HUGO_REFLINKSERRORLEVEL` variable MUST continue to control it.

### 3.7 Translation pipeline

- **FR-TM-1** `crowdin.yml` MUST add file entries for `docs/sources/en-US/**/*.md` with `translation: docs/sources/%locale%/**/%original_file_name%` and preserved hierarchy, and for the English Hugo i18n string table. The Crowdin `%locale%` codes MUST map to the directory names in section 2.2 (use `languages_mapping` where Crowdin's codes differ, for example `zh-Hans` and `zh-Hant`).
- **FR-TM-2** Crowdin's Markdown parser MUST be configured to: treat front matter as structured with only FR-CONT-1 keys translatable; exclude code blocks and inline code; and protect Hugo shortcode delimiters and non-visible parameters as non-translatable placeholders.
- **FR-TM-3** Initial population MAY use machine translation with human post-editing. Each translated file MUST record its method in front matter (refer to FR-SYNC-1) so that readers and reviewers can distinguish machine-only output.
- **FR-TM-4** The existing `i18n-crowdin-upload.yml` and `i18n-crowdin-download.yml` workflows MUST be extended, or sibling workflows added, to upload English docs on changes to `docs/sources/en-US/**` and to open a pull request with downloaded translations on the existing daily schedule.
- **FR-TM-5** A per-locale glossary MUST be maintained at `docs/i18n/glossary/<locale>.tbx` and loaded into Crowdin. It MUST be seeded from the translated UI strings in `public/locales/<locale>/grafana.json` so that docs terminology matches the product UI.
- **FR-TM-6** The generated transformations page MUST be localized through its sources: `scripts/docs/generate-transformations.ts` and `public/app/features/transformers/docs/content.ts`. The generator MUST emit one file per locale into each language tree. Translated text for the generator MUST come from Crowdin, not from hand edits in the generated Markdown.

### 3.8 Synchronization and staleness

- **FR-SYNC-1** Every translated file MUST contain the following front matter keys, written by the import tooling and never by hand:

  ```yaml
  translation:
    source_path: alerting/fundamentals/_index.md
    source_revision: <git blob SHA of the English file>
    method: human | machine | machine-post-edited | source-copy
    status: current | outdated
  ```

- **FR-SYNC-2** A script MUST compute, for every translated file, whether `source_revision` equals the current English blob SHA and set `status` accordingly.
- **FR-SYNC-3** When an English file is added, CI MUST ensure a counterpart exists in every language tree before merge. Until Crowdin returns a translation, the counterpart MUST be a copy of the English content with `translation.status: outdated` and `translation.method: source-copy`. This satisfies FR-DIR-3 without blocking English contributors.
- **FR-SYNC-4** When an English file is moved or deleted, the same move or deletion MUST be applied to every language tree in the same pull request. A script (`scripts/docs/i18n-sync.ts`) MUST perform moves and deletions and update `docs/i18n/status.json`.
- **FR-SYNC-5** Pages with `translation.status: outdated` or `translation.method` of `machine` or `source-copy` MUST display a localized notice with a link to the English page. **[EXT]**
- **FR-SYNC-6** `docs/i18n/status.json` MUST report, per locale: total pages, current, outdated, machine-only, and word counts. It MUST be regenerated in CI and committed by the download workflow.

### 3.9 Build, preview, and publish

- **FR-BUILD-1** `make docs` MUST serve all languages locally. A `LANGS` variable (default: all) MUST allow building a subset, for example `make docs LANGS=en-US,de-DE`, to keep local builds fast.
- **FR-BUILD-2** `docs/variables.mk` and `docs/make-docs` MUST mount `docs/sources/<locale>` for each selected language with the Hugo `lang` attribute, and mount `docs/hugo/config` and `docs/hugo/i18n`. The change MUST be made upstream in `grafana/writers-toolkit` first (C-2).
- **FR-BUILD-3** `.github/workflows/lint-build-docs.yml` MUST copy each language tree into the docs-base container at the matching language content root and build all languages.
- **FR-BUILD-4** `.github/workflows/documentation-ci.yml` (Vale) MUST run only on `docs/sources/en-US/**`. Vale's English styles MUST NOT run on target languages.
- **FR-BUILD-5** `.github/workflows/deploy-pr-preview.yml` MUST build previews for English and any language touched in the pull request.
- **FR-BUILD-6** The publish sync to `grafana/website` MUST copy each language tree to the language-specific content directory configured there, for both `next` (from `main`) and `latest` (from the release branch). **[EXT]**
- **FR-BUILD-7** The `docs/Makefile` transformations target and `docs/README.md` MUST be updated for the new paths.
- **FR-BUILD-8** All other references to `docs/sources` in the repository (workflows, `CODEOWNERS`, `.prettierignore`, scripts such as `scripts/docs/*`, `update-schema-types.yml`, `defaults-ini-docs-reminder.yml`) MUST be updated. A search for the string `docs/sources/` excluding `docs/sources/<locale>/` MUST return no stale paths.

### 3.10 Validation

- **FR-VAL-1** A CI check (`scripts/docs/i18n-verify.ts`, analogous to `i18n-verify.yml` for UI strings) MUST fail when any of the following is true:
  - An English page has no counterpart in a target language (FR-DIR-3).
  - A target-language page has no English counterpart and no `translationKey` (FR-DIR-4).
  - A non-translatable front matter key differs from English (FR-CONT-1).
  - The set, order, names, or non-visible parameters of shortcodes differ from English (FR-CONT-4).
  - The number of fenced code blocks differs from English, or any code block content differs (FR-CONT-3).
  - An explicit heading anchor required by FR-CONT-5 is missing.
  - `translation` front matter is missing or malformed (FR-SYNC-1).
  - The generated `languages` configuration is out of date (FR-CFG-2).
- **FR-VAL-2** `yarn prettier:checkDocs` MUST continue to pass for all language trees.
- **FR-VAL-3** The Hugo build MUST complete with zero missing i18n string placeholders and zero `REF_NOT_FOUND` errors attributable to translated content.

## 4. Non-functional requirements

These requirements constrain quality attributes of the solution rather than its features.

### 4.1 Performance

- **NFR-PERF-1** A full CI build of all 20 languages SHOULD complete in under 20 minutes on the current runner class. If not, CI MUST build English plus languages changed in the pull request, and a scheduled job MUST build all languages.
- **NFR-PERF-2** `make docs LANGS=en-US` MUST perform within 10% of the current single-language `make docs`.

### 4.2 Repository size

- **NFR-SIZE-1** The translated trees add roughly 19 times the current 11 MB of Markdown. The implementation MUST measure the resulting clone size and MUST NOT duplicate binary assets (FR-DIR-5).
- **NFR-SIZE-2** Sparse checkout instructions MUST be documented in `docs/README.md` for contributors who only need English.

### 4.3 Quality

- **NFR-QUAL-1** Before a language is listed in the production language switcher, at least 90% of its pages by word count MUST have `translation.status: current`, and 100% of pages in the top-level `introduction`, `setup-grafana`, and `fundamentals` sections MUST be `method: human` or `machine-post-edited`.
- **NFR-QUAL-2** Each locale MUST have a named reviewer or vendor responsible for linguistic review. Pull requests from the Crowdin download workflow MUST be reviewable per locale.

### 4.4 Maintainability

- **NFR-MAINT-1** English contributors MUST NOT be required to edit target-language files by hand. All target-language changes originate from Crowdin or from the sync and import scripts.
- **NFR-MAINT-2** Adding a new UI locale to `languages.ts` and `public/locales/` MUST be sufficient to add a documentation language, after running the generator and the sync script. No other file requires manual edits.

### 4.5 Compatibility

- **NFR-COMPAT-1** Every existing English URL MUST continue to resolve with HTTP 200 without a redirect after migration.
- **NFR-COMPAT-2** Release branches created before the migration MUST keep building with the old single-language layout. The publish workflow MUST detect the layout per branch.

### 4.6 Extensibility

- **NFR-EXT-1** Localized screenshots MAY be added later by placing images at a language-specific path and having the `figure` shortcode prefer it. The design MUST NOT prevent this.
- **NFR-EXT-2** Adding a right-to-left language MUST only require setting `languageDirection: rtl` in the generated configuration and supporting it in layouts. **[EXT]**

### 4.7 Security

- **NFR-SEC-1** Content downloaded from Crowdin MUST be treated as untrusted. The verification in FR-VAL-1 MUST reject raw HTML, `<script>` tags, and shortcodes not present in the English source, to prevent injection through translated content.
- **NFR-SEC-2** Crowdin credentials MUST continue to come from Vault through the existing `get-vault-secrets` action. No new long-lived secrets are stored in the repository.

## 5. Migration plan

The migration is split into phases so that English publishing is never interrupted (C-3). Each phase is a separate pull request, and frontend, backend, and docs changes stay separate per the repository guidelines.

1. **Phase 0: External prerequisites [EXT].** Confirm the Hugo version in docs-base (A-1). Add multilingual configuration, language switcher, `hreflang`, language-aware `docs/shared` and link resolution, and string tables to `grafana/website` and docs-base. Update `make-docs` in `grafana/writers-toolkit`.
2. **Phase 1: Restructure English.** `git mv docs/sources/* docs/sources/en-US/`. Update all path references (FR-BUILD-3 to FR-BUILD-8) and the publish workflow. Verify NFR-COMPAT-1 with a URL diff of the built site before and after.
3. **Phase 2: Tooling.** Add the language config generator (FR-CFG-2), sync script (FR-SYNC-4), verification script (FR-VAL-1), status report (FR-SYNC-6), glossary seeding (FR-TM-5), and localized transformations generator (FR-TM-6).
4. **Phase 3: Crowdin integration.** Extend `crowdin.yml` and workflows (FR-TM-1 to FR-TM-4). Upload the English source and glossaries.
5. **Phase 4: Initial translation.** Populate all 19 language trees with translated files, one pull request per locale. Machine translation with post-editing is permitted per FR-TM-3.
6. **Phase 5: Launch per language.** Enable each language in the production switcher when it meets NFR-QUAL-1. Languages that don't yet qualify are built and published but marked `noindex` and hidden from the switcher.

## 6. Acceptance criteria

The implementation is accepted when all of the following are true on `main`:

| ID    | Criterion                                                                                                                                            | Verifies                       |
| ----- | ---------------------------------------------------------------------------------------------------------------------------------------------------- | ------------------------------ |
| AC-1  | `docs/sources/` contains exactly the 20 locale directories in section 2.2 and nothing else.                                                          | FR-DIR-1, FR-DIR-2             |
| AC-2  | `scripts/docs/i18n-verify.ts` exits 0, reporting 773 (or current) pages for each of 20 languages.                                                    | FR-DIR-3, FR-DIR-4, FR-VAL-1   |
| AC-3  | No target-language page has `translation.method: source-copy`, that is, every file contains real translated text.                                    | FR-DIR-3, FR-TM-3              |
| AC-4  | The Hugo build in CI completes for all languages with no missing i18n placeholders and no broken refs.                                               | FR-VAL-3, FR-I18N-3, FR-LINK-6 |
| AC-5  | A crawl of the pre-migration English sitemap returns 200 for every URL on the post-migration build.                                                  | FR-CFG-4, NFR-COMPAT-1         |
| AC-6  | `/de-de/docs/grafana/latest/alerting/` renders German content, `lang="de-DE"`, a switcher with 20 entries, and `hreflang` links for all languages.   | FR-CFG-5, FR-HUGO-2, FR-HUGO-3 |
| AC-7  | A German page that includes `docs/shared` content shows the German shared file, and falls back to English when that file is removed in a test build. | FR-LINK-3                      |
| AC-8  | A link to `/docs/grafana/<GRAFANA_VERSION>/alerting/` inside a Japanese page resolves to `/ja-jp/docs/grafana/latest/alerting/`.                     | FR-LINK-1                      |
| AC-9  | Changing one English page and running the status script marks exactly that page `outdated` in all 19 languages.                                      | FR-SYNC-2, FR-SYNC-6           |
| AC-10 | Moving one English page with the sync script moves it in all languages and keeps working aliases in each language.                                   | FR-SYNC-4, FR-LINK-4           |
| AC-11 | Adding a test locale to `languages.ts` and running the generator and sync script produces a buildable language with no other manual edits.           | NFR-MAINT-2                    |
| AC-12 | A Crowdin download containing an injected `<script>` tag or an unknown shortcode fails verification.                                                 | NFR-SEC-1                      |

## 7. Open questions

These questions need decisions from the owners named before implementation starts.

1. **URL scheme** (docs platform team): language prefix at site root (`/de-de/docs/grafana/...`, as specified) or inside the project (`/docs/grafana/de-de/latest/...`). The first matches Hugo's native behavior. The second avoids site-wide multilingual configuration in `grafana/website`.
2. **Short language keys** (docs platform, SEO): whether to use `de` instead of `de-de` for locales with a single regional variant. The spec keeps full locales to match `public/locales` and to keep `pt-BR`/`pt-PT` and `zh-Hans`/`zh-Hant` unambiguous.
3. **Version coverage** (localization team): translate only `next` and `latest`, or also older release branches.
4. **Translation budget and vendors** (localization team): human, machine with post-editing, or a mix per section.
5. **Repository placement** (docs maintainers): keep translations in this repository as specified, or keep them in a separate repository synced at publish time to limit clone size (NFR-SIZE-1).
