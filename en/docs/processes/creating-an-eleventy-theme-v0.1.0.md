---
title: "Process: Creating an Eleventy Theme"
dc:title: "Process: Creating an Eleventy Theme"
dc:creator: "Christopher Steel"
dc:contributor: "Claude Opus 5.5 (Anthropic), AI assistance"
dc:subject: "process, Eleventy, theme, self-contained theme, eleventy-site layout, site overrides, checks, fixtures, releases"
dc:description: "How a Vishpala Eleventy theme is built so that it lives entirely in its own folder: a site connects to it with one configuration file, keeps its overrides in its own theme folder, and conforms to the eleventy-site layout without exceptions."
dc:date: "2026-10-31"
dc:type: "Text"
dc:format: "text/markdown"
dc:identifier: "urn:uuid:1138506f-56ae-4485-80db-f6b55b92b3dd"
dc:language: "en-CA"
dc:rights: "CC-BY-4.0"
sat:version: "0.1.0"
sat:status: "Draft"
---

# Process: Creating an Eleventy Theme

Version: 0.1.0
Status: Draft

## Purpose

This process builds an Eleventy theme that a website can adopt by copying one folder. Everything the theme needs, from layouts to fonts to the scripts that test it, lives in `themes/<name>/`. A site connects to it through `eleventy.config.js` alone, so a site built on the theme holds only what the `eleventy-site` layout describes: `eleventy.config.js`, `package.json`, `package-lock.json`, `src/` and `themes/` <a id="cite-steel-2026b"></a>[(Steel, 2026b)](#ref-steel-2026b).

The vishpala.com theme, in `themes/vishpala/` of eleventy-website-vishpala-com, follows this pattern. Katagami 0.5.1 kept its site overrides, scripts and tool configuration at the root of its repository, so a site copying it had to copy those too, and each of them is a path the `eleventy-site` layout does not describe. The steps below are drawn from that difference. The examples use a theme named `katagami` and a site named `chrissteel`; replace them with your own names throughout.

## Steps

### Choose the name

The theme's name is used for its folder, `themes/<name>/`, and its repository, `eleventy-theme-<name>`. It may use only lowercase letters, digits and single hyphens.

Gate: the name matches `^[a-z0-9]+(-[a-z0-9]+)*$`.

### Create the repository

Create the theme's repository with Process: Creating a New Repository with uc-automa <a id="cite-steel-2026a"></a>[(Steel, 2026a)](#ref-steel-2026a), with the `eleventy-site` layout. A theme repository is an Eleventy site whose `src/` is the theme's demo and fixture site, so the same layout describes both the theme's repository and every site that uses the theme.

The founding decision for a theme records its purpose and style, the languages its fixture site covers, including a right-to-left language, its accessibility targets, and its licensing: code, documentation and content, and every font and vendored library under its own licence.

Gate: the repository passes `uc-automa check` with the `eleventy-site` layout.

### Keep the theme in one folder

Everything the theme provides lives under `themes/<name>/`:

- `index.js`, the Eleventy plugin that registers the theme
- `theme.yaml`, the theme's manifest, with its name and version
- layouts, parts, partials, blocks, views and defaults
- interface strings for each language
- assets: CSS, JavaScript, images, and self-hosted fonts and vendored libraries, each with its licence file
- `scripts/`, the theme's own tools (see Ship the tools with the theme)
- `README.md`, `CHANGELOG.md` and `LICENSE` for the theme itself

Nothing a site needs from the theme lives outside this folder.

Gate: deleting everything at the repository root except `eleventy.config.js`, `package.json`, `package-lock.json`, `src/` and `themes/` leaves the demo site building and its checks passing.

### Connect a site with one file

A site loads the theme from `eleventy.config.js` and does nothing else there but set its directories and pass the theme its options. The theme's README gives the exact lines its version needs; Katagami needs the includes and layouts directories set, as below:

```js
import katagami from "./themes/katagami/index.js";

export { config } from "./themes/katagami/index.js";

export default function (eleventyConfig) {
  eleventyConfig.setInputDirectory("src");
  eleventyConfig.setIncludesDirectory("_includes");
  eleventyConfig.setLayoutsDirectory("_layouts");
  eleventyConfig.setDataDirectory("_data");
  eleventyConfig.setOutputDirectory("_site");
  eleventyConfig.addPlugin(katagami);
}
```

Site settings that change with the site, such as section colours, the licence shown in the footer, or whether search is built, are plugin options or data in `src/_data/`, never edits to the theme's files.

Gate: the theme's README shows the complete `eleventy.config.js` a site needs, and the demo site uses exactly that pattern.

### Make site overrides an option

A site customizes the theme by shadowing its files: a file at the same path in the site's overrides folder replaces the theme's file. The folder is a plugin option, never a fixed name at the repository root. Each site keeps its overrides in its own theme folder, `themes/<site>/`, beside the theme it overrides, and passes it as the option, such as `overrides: "themes/chrissteel"`. Any site-only build code goes there too, such as an `index.js` the theme loads after itself. A site with no overrides passes no option and has no such folder.

When the option is not given, the theme uses no overrides folder.

Gate: the demo site's overrides live under `themes/`, and no path in the theme's code names a folder at the repository root other than those the `eleventy-site` layout describes.

### Ship the tools with the theme

Scripts that vendor fonts and libraries, generate fixture images, or check the built site are part of the theme and live in `themes/<name>/scripts/`. The site's `package.json` runs them from there:

```json
{
  "scripts": {
    "build": "eleventy",
    "check": "node themes/katagami/scripts/check-pages.mjs",
    "check:browser": "node themes/katagami/scripts/check-browser.mjs",
    "vendor-fonts": "node themes/katagami/scripts/vendor-fonts.mjs"
  }
}
```

Gate: no script the theme documents lives outside `themes/<name>/`.

### Drive the checks from the site

The theme's checks test rules that hold on every site: markup, the Content Security Policy, fonts, vertical rhythm, accessibility in every contrast mode, right-to-left pages and the page without JavaScript. They read the pages to test from the site, through a data file in its data directory, such as Katagami's `src/_data/checks.yaml`, and never name a page themselves.

Checks that only make sense on the theme's fixture pages, such as a typography specimen, are listed in the demo site's configuration as fixture checks, so a site that has no such page skips them rather than failing.

Gate: the checks pass on the demo site, and on a second site with different URLs and no fixture pages, without editing the checks.

### Keep tool configuration out of the root

A site should not have to copy tool configuration files to use the theme.

- **Node version.** State it in `package.json` under `engines`, which npm enforces. A version manager file such as `.nvmrc` is optional and never required by the theme.
- **Lint rules.** Keep the theme's stylelint configuration in `themes/<name>/`, and extend it from the `stylelint` key in `package.json`.
- **Editor and assistant settings.** Claude Code settings and similar files stay out of the repository, ignored in `.gitignore`, unless the layout describes them.

Gate: the repository root holds no tool configuration file the `eleventy-site` layout does not describe.

### Keep the demo site as the test suite

The demo and fixture site in `src/` is how the theme is tested. It covers every part, block and view, every font set, and at least three languages, one of them right-to-left, marked as a fixture locale so it stays out of a production language switcher until a reader of that language has reviewed it. The theme project's own documentation lives in `en-ca/docs/`, and none of it is needed by a site.

Gate: every part, block, view and font set appears on at least one fixture page, and the right-to-left locale builds.

### Test with a copy

Before each release, create a scratch site that copies only `themes/<name>/` and writes its own `eleventy.config.js`, `package.json` and a few pages under `src/`, with locale codes and URLs that differ from the demo site's.

Gate: the scratch site builds, its checks pass, and `uc-automa check` reports it conforms to the `eleventy-site` layout.

### Release so sites can pin

Release the theme with an annotated version tag, and keep the theme's version in `theme.yaml` and its changes in `themes/<name>/CHANGELOG.md`, so they travel with the folder.

A site adopts the theme by copying `themes/<name>/` from a tagged release, never by forking the theme's repository: a fork would carry the demo site and its fixtures into the site, and every update would conflict with them. The site records the release it copied in a decision record, and updates by replacing the folder with a newer release and recording that in a new decision record.

Gate: the release is tagged, and `theme.yaml`, the theme's changelog and the tag agree on the version.

## Principles

- A theme is one folder. Copying the folder is the whole installation.
- A site connects to a theme through one file and customizes it through options and its own overrides folder, never by editing the theme.
- The theme ships the tools that test it, and the site decides what they test.
- A theme that needs a path the `eleventy-site` layout does not describe is changed, rather than the layout widened to fit it.

## References

<a id="ref-steel-2026a"></a>Steel, C. (2026a). *Process: Creating a new repository with uc-automa* (Version 0.1.0) [Draft]. uc-processes. https://github.com/steelcj/uc-processes/blob/main/en-ca/docs/processes/creating-a-new-repository-with-uc-automa-v0.1.0.md
[Return to citation](#cite-steel-2026a)

<a id="ref-steel-2026b"></a>Steel, C. (2026b). *Decision: An eleventy-site layout* (Version 0.1.0). uc-automa. https://github.com/steelcj/uc-automa/blob/main/en-ca/docs/decisions/0037--eleventy-site-layout-v0.1.0.md
[Return to citation](#cite-steel-2026b)

## Changelog

| Version | Date | Status | Notes |
| --- | --- | --- | --- |
| 0.1.0 | 2026-10-31 | Draft | Initial version, drawn from comparing the vishpala.com theme with Katagami 0.5.1 |

## License

This document, *Process: Creating an Eleventy Theme*, by **Christopher Steel**, with AI assistance from **Claude Opus 5.5 (Anthropic)**, is licensed under the [Creative Commons Attribution 4.0 International License](https://creativecommons.org/licenses/by/4.0/).
