---
title: "Process: Licensing a Repository with REUSE"
dc:title: "Process: Licensing a Repository with REUSE"
dc:creator: "Christopher Steel"
dc:contributor: "Claude Opus 5.5 (Anthropic), AI assistance"
dc:subject: "process, licensing, REUSE, REUSE.toml, SPDX, LICENSES, fonts, icons, trademarks, audit"
dc:description: "How a Vishpala repository records the licence of every file with REUSE: inventory what it holds, identify each third-party item's licence from its own source, place the licence texts, write REUSE.toml general-first, verify, and audit whenever what the repository ships changes."
dc:date: "2026-10-23"
dc:type: "Text"
dc:format: "text/markdown"
dc:identifier: "urn:uuid:31436ab2-84b7-49a8-911d-beb8d205d886"
dc:language: "en-CA"
dc:rights: "CC-BY-4.0"
sat:version: "0.1.0"
sat:status: "Draft"
---

# Process: Licensing a Repository with REUSE

Version: 0.1.0
Status: Draft

## Purpose

This process makes sure every file in a repository has a known licence and a known copyright holder, recorded where both people and tools can read it. It covers Vishpala's own work, the third-party material a repository carries, such as fonts and icons, and the marks that are not licensed at all.

uc-automa marks licences with REUSE <a id="cite-fsfe-2024"></a>[(Free Software Foundation Europe, 2024)](#ref-fsfe-2024): the licence texts live in `LICENSES/`, and `REUSE.toml` says which licence covers which paths. Vishpala's defaults are MIT for code and CC BY 4.0 for documentation and content <a id="cite-steel-2026a"></a>[(Steel, 2026a)](#ref-steel-2026a). The `reuse` offering plants a starting `REUSE.toml` once, and the repository owns it from then on, so each repository completes it for what it actually holds.

The examples come from eleventy-website-vishpala-com, an Eleventy website.

## Steps

### Record the licensing decision

Before any annotation is written, the repository's founding decision says how its material is licensed: code, documentation and content, anything excluded from those licences, such as logos and wordmarks, and anything that keeps its own licence, such as fonts. `REUSE.toml` carries out that decision; it does not make it.

Gate: the founding decision, or a later decision, states the licensing for every category the repository holds.

An example of a founding decision from the eleventy-website-vishpala-com Github repository:

```bash
https://github.com/steelcj/eleventy-website-vishpala-com/en-ca/docs/decisions/0001--build-website-vishpala-com-v0.1.0.md
```

### Inventory what the repository holds

Once the repository has been setup using uc-automa, ( see [creating-a-new-repository-with-uc-automa-v0.1.0.md](creating-a-new-repository-with-uc-automa-v0.1.0.md) ) and examples each licensed resource has been collected in the repository you can query the repository for licensed materials.

List every file git tracks or would track, and sort them into categories. Build the list from the real files, never from what a layout or a theme usually contains: a path written from memory, such as `themes/vishpala/fonts/` when the fonts are in `themes/vishpala/assets/fonts/`, silently matches nothing.

```sh
git ls-files --cached --others --exclude-standard
```

The categories:

- **Code**: configuration, scripts, templates, styles and data files, such as `eleventy.config.js`, `package.json`, `*.njk`, `*.js`, `*.css` and `*.json`.
- **Documentation**: everything under `en-ca/`, and `README.md`, `CHANGELOG.md`, `ROADMAP.md`, `CONTRIBUTING.md` and `CLAUDE.md`.
- **Content**: what the site publishes for people to read, such as the Markdown pages under `src/`.
- **Third-party files**: anything Vishpala did not write or create, such as fonts, icon sets, images and vendored libraries.
- **Third-party material inside Vishpala files**: code or artwork copied into a file Vishpala otherwise wrote, such as icons drawn inline as SVG in a template.
- **Marks**: logos and wordmarks, which are not licensed.

These three searches find the files most often missed:

```sh
find . -path ./.git -prune -o -path ./node_modules -prune -o \( -iname '*.woff*' -o -iname '*.ttf' -o -iname '*.otf' -o -iname '*.svg' -o -iname '*.png' -o -iname '*.jpg' -o -iname '*.webp' -o -iname '*.ico' -o -iname '*licen*' -o -iname '*ofl*' -o -iname '*logo*' -o -iname '*wordmark*' \) -print

grep -rn -E "font-family|@font-face|url\(https?:" --include='*.css' .

grep -rln "<svg" --include='*.njk' --include='*.html' .
```

A font named only in a `font-family` stack and not shipped as a file is a system font, and needs no entry. A font or script loaded from another site at run time is not in the repository either, but it is shipped to every visitor, so it is listed for the audit.

Gate: every file belongs to exactly one category, and every font, image, icon and vendored library is listed with where it came from.

#### VERY IMPORTANT

Vishpala prefers to download fonts and other scripts and resources local rather than loading them from third party servers.

Storing them locally ensures for a significantly smaller network footprint, reduces bandwidth requirements and helps to ensure that websites, publications and applications can continue to operate rapidly and reduces our carbon footprint significantly in many cases.

In addition to helping to support our speed, sustainability and the reliance on remote dependencies these features are also leveraged by Vishpala in order to create applications built to support emergency teams and customers and users in areas or regions where bandwidth is limited, unavailable or unreliable.

### Identify each third-party licence from its source

For each third-party item, find its licence and its exact copyright line in its own upstream source, such as the project's `LICENSE` or `OFL.txt`, not in a summary, a package page or an AI collaborator's recollection. Keep a copy of the text you relied on.

Two common cases:

- **A font family.** Atkinson Hyperlegible is published under the SIL Open Font License 1.1, and its `OFL.txt` gives the copyright line `Copyright 2020 Braille Institute of America, Inc.` <a id="cite-braille-2020"></a>[(Braille Institute of America, 2020)](#ref-braille-2020).
- **Icons copied inline.** The globe, search and close icons in `base.njk` match Feather Icons, which is MIT-licensed. MIT asks that the copyright notice travel with the work, so the template records Feather's copyright holder beside Vishpala's.

Gate: every third-party item has an SPDX identifier and a copyright line taken from its own source.

### Place the licence texts

Every licence named in `REUSE.toml` has its text in `LICENSES/`, named by its SPDX identifier, and no licence is placed there that no file uses. The REUSE specification requires both [(Free Software Foundation Europe, 2024)](#ref-fsfe-2024).

```sh
mkdir -p LICENSES
curl -fsSL https://raw.githubusercontent.com/googlefonts/atkinson-hyperlegible/main/OFL.txt -o LICENSES/OFL-1.1.txt
head -3 LICENSES/OFL-1.1.txt
```

`MIT.txt` and `CC-BY-4.0.txt` arrive from the `license-mit` and `license-cc-by-4-0` offerings. A licence with no SPDX identifier, such as the statement that the marks are not licensed, takes a `LicenseRef-` name and a text written by hand, such as `LICENSES/LicenseRef-Vishpala-Trademark.txt`:

```text
The Vishpala name, logos and wordmarks, including vishpala-mark.svg, are trademarks and copyright of Vishpala. All rights reserved.

They are not covered by the MIT License or the Creative Commons Attribution 4.0 International License that apply to the rest of this repository. You may not use them except to refer to Vishpala and its work, without written permission from Vishpala.
```

The name in `REUSE.toml` and the filename match exactly, capitals included.

Gate: `LICENSES/` holds one text for each licence `REUSE.toml` names, and nothing else.

### Write REUSE.toml general first, exceptions last

When several annotations match the same file, only the last matching one is used [(Free Software Foundation Europe, 2024)](#ref-fsfe-2024). So keep the planted annotations as they are, then add the repository's general annotations, then each narrower exception after the annotation it overrides.

For eleventy-website-vishpala-com, after the two planted annotations:

```toml
[[annotations]]
path = ["eleventy.config.js", "package.json", "package-lock.json", "src/**", "themes/**"]
SPDX-FileCopyrightText = "Christopher Steel"
SPDX-License-Identifier = "MIT"

[[annotations]]
path = ["src/**/*.md"]
SPDX-FileCopyrightText = "Christopher Steel"
SPDX-License-Identifier = "CC-BY-4.0"

[[annotations]]
path = ["themes/vishpala/assets/fonts/**"]
SPDX-FileCopyrightText = "2020 Braille Institute of America, Inc."
SPDX-License-Identifier = "OFL-1.1"

[[annotations]]
path = ["themes/vishpala/assets/img/vishpala-mark.svg"]
SPDX-FileCopyrightText = "Vishpala"
SPDX-License-Identifier = "LicenseRef-Vishpala-Trademark"

[[annotations]]
path = ["themes/vishpala/layouts/base.njk"]
SPDX-FileCopyrightText = ["Christopher Steel", "2013-2017 Cole Bemis"]
SPDX-License-Identifier = "MIT"
```

Rules that keep it correct:

- **Annotate files, not parts of files.** REUSE records licensing per file. A mark drawn inline in a template would take the template's licence, so a mark lives in its own file, and the templates refer to it.
- **Name every copyright holder.** A file holding someone else's work, such as the inline icons, lists both holders.
- **A licence header inside a file wins over `REUSE.toml`.** That is REUSE's default precedence, `closest`, so a vendored file that carries its own SPDX header keeps it.
- **Prefer exact paths for exceptions.** A glob such as `**/*.svg` for the mark would also catch every icon added later.

Gate: every category and every third-party item from the inventory is covered by an annotation, and each exception comes after the annotation it overrides.

### Verify

```sh
uc-automa check
pipx run reuse lint
```

`uc-automa check` enforces `licensing/rights-match-reuse`: each document's `dc:rights` must name the licence `REUSE.toml` says covers it. `reuse lint` checks the rest of the REUSE specification, such as files with no licensing information and licence texts that are missing or unused. Then read the inventory once more against `REUSE.toml`, since neither tool knows that a font came from somewhere else.

Gate: both commands pass, and the inventory and `REUSE.toml` agree.

### Commit with the licences together

Commit `REUSE.toml`, the new texts in `LICENSES/` and the files they cover in the same commit, so that no commit holds a file without its licence. Read `git status` before committing.

Gate: the commit that adds a third-party file also adds its licence text and its annotation.

## When to audit

Run the inventory, identification and verification steps again:

- whenever a font, icon, image, theme or vendored library is added, replaced or updated;
- whenever a template is changed to inline third-party artwork or code;
- before every release, as part of what a release means;
- whenever a dependency or service that ships to visitors changes, even though it is not in the repository.

An audit also looks beyond the repository, at what is actually shipped. A built site, such as `_site/`, can carry third-party code that the repository holds only as a dependency, such as a search library's script. Its licence, and any notice it requires, has to travel with the built site even though, for example,  `node_modules/` is never committed. Each audit is recorded in `CHANGELOG.md`, with what was found and changed.

Gate: the audit's findings are in `CHANGELOG.md`, and every finding is either fixed or recorded as an open item in `ROADMAP.md`.

## Principles

- Every file has a licence and a copyright holder, recorded, not assumed.
- A third-party licence is taken from its own source, and the source is kept.
- `REUSE.toml` is built from the files the repository holds, never from what it usually holds.
- Marks are excluded from the licences explicitly, never by omission.
- What a repository ships to people, including its built output, is audited, not only what it stores.

## References

<a id="ref-braille-2020"></a>Braille Institute of America. (2020). *Atkinson Hyperlegible* [Typeface]. Google Fonts. https://github.com/googlefonts/atkinson-hyperlegible
[Return to citation](#cite-braille-2020)

<a id="ref-fsfe-2024"></a>Free Software Foundation Europe. (2024). *REUSE specification* (Version 3.2). https://reuse.software/spec-3.2/
[Return to citation](#cite-fsfe-2024)

<a id="ref-steel-2026a"></a>Steel, C. (2026a). *Decision: MIT for code, CC BY 4.0 for documentation* (Version 0.1.0). uc-automa. https://github.com/steelcj/uc-automa/blob/main/en-ca/docs/decisions/0025--mit-for-code-cc-by-for-documentation-v0.1.0.md
[Return to citation](#cite-steel-2026a)

## Changelog

| Version | Date | Status | Notes |
| --- | --- | --- | --- |
| 0.1.0 | 2026-10-23 | Draft | Initial version, drawn from licensing eleventy-website-vishpala-com |

## License

This document, *Process: Licensing a Repository with REUSE*, by **Christopher Steel**, with AI assistance from **Claude Opus 5.5 (Anthropic)**, is licensed under the [Creative Commons Attribution 4.0 International License](https://creativecommons.org/licenses/by/4.0/).
