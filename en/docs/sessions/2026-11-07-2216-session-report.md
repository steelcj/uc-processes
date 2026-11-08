---
name: session-report-2026-11-07-2216
org: universal-cake
lang: en
type: session-report
date: 2026-11-07
---

# Session report: 2026-11-07, release ceremony for new repositories

- Prompt: [`2026-11-07-2209-session-prompt.md`](2026-11-07-2209-session-prompt.md)
- Process: [Process: Running a Claude Code Session](../processes/running-a-claude-code-session-v0.1.0.md)
- Repositories read: uc-processes, uc-automa (decisions, directives, offerings, layouts, CLI)
- Status: **the task is complete.** Decisions A–C were made, and *Process: Releasing a Repository* 0.1.0 (Draft) was written. What it leaves open is tracked in uc-processes' new `ROADMAP.md` and, for the release-key principal, in uc-automa's `ROADMAP.md`.
- Committed on `main` and pushed at the owner's request (section 8).

## 1. What uc-processes has

uc-processes has no process for a release ceremony, for new repositories or otherwise. The pieces are spread across scripts and guides:

| Where | What it says |
|---|---|
| `cut-release.py` docstring | The cut step: check `VERSION` is clean and `[Unreleased]` has entries; bump through `bump-version.py`; turn `[Unreleased]` into a dated heading; commit only `VERSION` and `CHANGELOG.md`; add an annotated tag; **stop before pushing** |
| `publish-release.py` docstring | After a human push: check the tag is on the remote, build a byte-stable tarball and `SHA256SUMS`, optionally sign with **GPG**, and publish through a provider backend (`gh` or a plain directory) |
| `README.md`, Releases section | One paragraph: run `cut-release.py patch/minor/major`, push `main` and the tag, then run `publish-release.py`. The scripts are unmodified copies from sat-doc-automa at `fc0b72a`, to be replaced by `uc-automa release` commands (uc-automa 0026) "planned for uc-automa 0.2.0" |
| `en/docs/guides/releases/managing-the-vishpala-release-key-v0.1.3.1.md` | Creating, backing up, publishing and rotating the **SSH** release key. It implements uc-automa decisions 0030 and 0032 |
| `en/docs/guides/releases/generating-a-strong-passphrase-v0.1.1.md` | The passphrase for the release key |
| `en/docs/processes/creating-a-new-repository-with-uc-automa-v0.1.0.md` | New repositories get `CHANGELOG.md` and `VERSION` from uc-automa offerings. It ends: "Creating the repository on a forge, pushing and the first release are separate decisions" |

## 2. What uc-automa has

The ceremony is fully decided, but none of it is built yet.

| Source | What it says | Built? |
|---|---|---|
| **0026** Release ceremony | Three commands, `uc-automa release bump`, `cut` and `publish`, replace the copied scripts, and "No repository holds a copy". `cut` refuses a dirty tree, requires `uc-automa check` to pass, bumps, rolls the changelog, makes a surgical commit, adds an annotated tag, verifies, and stops before pushing. `publish` builds a byte-stable archive (twice, compared), release notes, `SHA256SUMS` and a signature. A `retire-copied-scripts` offering removes old copies | **No.** The CLI has no `release` command, and that offering doesn't exist |
| **0028** Wheels with each release | Python tools also ship pinned, reproducible wheels and `requirements-offline.txt`. Publishing requires a passing verification run on the release commit. The cut also updates the README's install tag | No |
| **0030** Where release keys are published | In two places: `uc-automa/release-keys/` (`allowed-signers`, `README.md` with fingerprints) and `vishpala.com/release-keys`. READMEs gain a "Get the release key" section | No, there's no `release-keys/` folder |
| **0032** SSH signatures | Tags are signed with the **SSH** key, replacing 0026's OpenPGP tags. SSH signatures are required on releases with wheels. OpenPGP becomes optional | No |
| **0039** Hand-cut 0.2.0 | uc-automa 0.2.0 was cut by hand, with an annotated tag and no signature. The ceremony moves to **0.3.0** | (applied) |
| Directives `release/annotated-version-tags` 0.2.0 and `release/readme-install-tag` 0.2.0 | In force. Tag `vX.Y.Z`, annotated, with the message "`<repo> X.Y.Z`"; a surgical release commit; README install references match `VERSION` | **Yes**, checked by `uc-automa check` |
| Offerings `changelog`, `version-file` | New repos get `CHANGELOG.md` and `VERSION` (0.1.0) | **Yes** |

So for a new repository today, what actually works is the 0039 pattern: cut by hand, following the two directives, and check with `uc-automa check`.

## 3. Mismatches the process can't settle on its own

1. **How things are signed.** The copied scripts sign `SHA256SUMS` with GPG and don't sign tags. Under uc-automa 0032, tags are SSH-signed and SSH signatures are required.
2. **The allowed-signers principal.** The uc-processes release-key guide uses `vishpala-release namespaces=…`. uc-automa 0028 and 0030 use the owner's email address as the principal. Every `ssh-keygen -Y verify -I <principal>` command depends on which one is right.
3. **The release commit.** The copied scripts commit only `VERSION` and `CHANGELOG.md`. The directive in force, `release/readme-install-tag` (and 0028), also has the commit update the README's install tag. A repo whose README installs its own tag would fail `uc-automa check` after a scripted cut.
4. **Stale wording in uc-processes.** Its README says the scripts are replaced "in uc-automa 0.2.0". Since uc-automa 0039, that's 0.3.0.
5. **Copied scripts and 0026.** uc-processes still holds the copied scripts, which 0026 says no repository should hold.

## 4. Proposed process: *Releasing a Repository* v0.1.0 (Draft)

It would go in `en/docs/processes/`, with uc-automa's decisions as the authority. Each step ends in a Gate:

1. **Set the repository up for releases.** The `version-file` and `changelog` offerings at creation, and keeping `[Unreleased]` up to date while working.
2. **The release key** (done once, by the owner). Point to the two key guides and to where uc-automa 0030 publishes the key.
3. **Before cutting.** Clean tree, `[Unreleased]` has entries, `uc-automa check` passes, licensing audit (*Process: Licensing a Repository with REUSE*).
4. **Cut.** By hand following the directives until uc-automa 0.3.0, with exact commands. After that, `uc-automa release cut`. Never copy the scripts into a new repo.
5. **Verify the release commit** before pushing (the verification run, for Python tools).
6. **Push**, as a deliberate human step: `git push origin main vX.Y.Z`.
7. **Publish.** `uc-automa release publish` from 0.3.0 (archive, notes, `SHA256SUMS`, SSH signatures, wheels). Until then, the tag is the release.
8. **How a reader verifies a release.** Get the key, compare the fingerprint, `git tag -v`, `ssh-keygen -Y verify`.
9. **Failed cuts.** Delete an unpushed failed cut and cut it again. Never reuse a pushed tag.

## 5. Decisions waiting on the owner

- **A. What it documents.** Two options:
  - the decided ceremony, with an "until uc-automa 0.3.0, cut by hand" step (recommended);
  - only what works today.
- **B. Mismatch 2.** Which principal is correct, `vishpala-release` or the owner's email address? Or should the process leave it as an open item? The second option would put the email address in another public document.
- **C. Mismatches 1, 3, 4 and 5.** Should the process record them as open items, with nothing else changed? Recommended, because fixing them means changing scripts or documents in uc-processes and uc-automa.

**Owner's response:**

> A. until uc-automa 0.3.0, cut by hand" step (recommended) B. Release keys goes in the roadmap and doc as decision to make and implement later 3. go with recommendation and add to the approriate roadmap

Read as:

- **A.** Document the decided ceremony, with an "until uc-automa 0.3.0, cut by hand" step.
- **B.** The release-key principal goes in the process document, and in the roadmap, as a decision to make and implement later.
- **C** (written "3"). Record mismatches 1, 3, 4 and 5 as open items, change nothing else, and add them to the appropriate roadmap.

uc-automa's `ROADMAP.md` "Next release" already covers mismatches 1 and 3, because the ceremony it plans signs with SSH and rewrites the README's install tag. uc-processes has no `ROADMAP.md`, although uc-automa's base layout requires one.

After the change table was shown, the owner confirmed it, and clarified where the work belongs:

> go ahead, since we are effectivly moving process "creation & documentation" to uc-processes, where the work happens on processes. that is where we want to document what we have now regarding releases and release processes. So that part is for those roadmaps and changelogs. It appears that this is what you are doing but your working can be unclear when dealin with multiple repos and files

So the release documentation and its open items live in uc-processes, with its own `ROADMAP.md` and `CHANGELOG.md`. uc-automa gets one roadmap bullet only, for the principal decision, because that decision record belongs to uc-automa. Below, every file is named with its repository.

## 5a. Decisions applied

### uc-processes

| File | Change |
|---|---|
| `uc-processes/en/docs/processes/releasing-a-repository-v0.1.0.md` | **New:** *Process: Releasing a Repository*, 0.1.0, Draft. It has nine steps, each with a Gate. The cut step has "Until uc-automa 0.3.0: cut by hand" with exact commands, and "From uc-automa 0.3.0: `uc-automa release cut`" (A). The release-key principal is a "Decision to make, then implement", written `<principal>` in commands (B). An "Open items" section lists the five mismatches (C). The references cite uc-automa 0026, 0028, 0030, 0032, 0039 and the two release directives |
| `uc-processes/ROADMAP.md` | **New**, in uc-automa's roadmap format. "Later" lists the principal decision, moving to `uc-automa release` plus `retire-copied-scripts` at 0.3.0 (which resolves mismatches 1, 3 and 5), the README's 0.2.0 → 0.3.0 wording (mismatch 4), and updating the process once the keys and tooling exist |
| `uc-processes/CHANGELOG.md` | `[Unreleased]` → "Added" gained two bullets: the release process and `ROADMAP.md` |

### uc-automa

| File | Change |
|---|---|
| `uc-automa/ROADMAP.md` | One bullet added under "Next release", after the `release-keys/` item: decide the allowed-signers principal before `release-keys/` is created. A decision record settles it, and the guide and READMEs follow |

Asked whether this bullet should stay in uc-automa or live only in `uc-processes/ROADMAP.md`, the owner answered:

> keep the uc-automa roadmap bullet

Nothing else in uc-automa changed, including its decisions. The owner's email address is not written in any of the new text.

Checked:

- In the process document, every citation anchor is defined once, every in-page link has a target, and every reference has a citation.
- Every relative link in the process document and in `ROADMAP.md` resolves to an existing file.
- The commands were not run. They come from the decisions and directives, and the `uc-automa release` commands don't exist yet.

## 6. Changes made in uc-processes this session

| File | Change |
|---|---|
| `en/docs/processes/running-a-claude-code-session-v0.1.0.md` | New: *Process: Running a Claude Code Session*, version 0.1.0, Draft (drawn from the uc-gradus session of 2026-11-07) |
| `CHANGELOG.md` | `[Unreleased]` gained an "Added" entry for that process |
| `en/docs/sessions/2026-11-07-2209-session-prompt.md` | New: this session's prompt |
| `en/docs/sessions/2026-11-07-2216-session-report.md` | New: this report |
| `en/docs/processes/releasing-a-repository-v0.1.0.md` | New: *Process: Releasing a Repository* (see 5a) |
| `ROADMAP.md` | New (see 5a) |
| `CHANGELOG.md` | Two more "Added" bullets (see 5a) |

In uc-automa, only `uc-automa/ROADMAP.md` changed, by one bullet (see 5a).

## 7. Open items

- [x] Decide A, B and C above. Then *Process: Releasing a Repository* is written, its changelog entry added, and this report updated.
- [ ] Decide the release key's allowed-signers principal: a decision record in uc-automa (tracked in `uc-automa/ROADMAP.md` and `uc-processes/ROADMAP.md`).
- [ ] The remaining items in `uc-processes/ROADMAP.md` "Later", most of which wait on uc-automa 0.3.0.
- [ ] Optionally list both new processes in `uc-processes/README.md`'s Processes table, once the `en-ca/` to `en/` move is settled.

## 8. Commits

The owner asked: "commit the changes in all three repos". The owner confirmed committing to each repository's own `main`: "Do you mean the changes repo a will be in the main of repo a. If so proceed".

| Repository | Commit | What it holds |
|---|---|---|
| uc-processes | `d60b25b` Add release and Claude Code session processes, ROADMAP | Only this session's files: `CHANGELOG.md`, `ROADMAP.md`, `en/docs/processes/releasing-a-repository-v0.1.0.md`, `en/docs/processes/running-a-claude-code-session-v0.1.0.md`, `en/docs/sessions/2026-11-07-2209-session-prompt.md`, and this report |
| uc-automa | `a76b8e9` Roadmap: decide the allowed-signers principal before release-keys/ | Only `ROADMAP.md`, one line, committed with `git commit -- ROADMAP.md` |
| uc-gradus | `26e7566` | That repository's session work, recorded in `uc-gradus/en/docs/sessions/2026-11-07-2113-session-report.md` |

Left uncommitted, because it is the owner's work in progress:

- **uc-processes:** the `en-ca/` → `en/` move. Asked whether to commit only Claude's files or include the move, the owner didn't choose, so only Claude's files were committed. Until the move is committed, the links in `releasing-a-repository-v0.1.0.md` to `en/docs/guides/releases/` and to `licensing-a-repository-with-reuse-v0.1.0.md` and `creating-a-new-repository-with-uc-automa-v0.1.0.md` don't resolve in history. The commit message says so.
- **uc-automa:** the staged `.gitignore`, an overview rename and a new session note stay staged, and were not part of the commit.

The report update recording these commits is `a6b8c10`.

### Pushes

The owner asked: "push all three repos". Both pushes were fast-forwards:

| Repository | Pushed | Notes |
|---|---|---|
| uc-processes | `206f00e..a6b8c10` | |
| uc-automa | `20e7fad..a76b8e9` | It also carried the owner's earlier unpushed commit `218bdec`. A push hook in that repository also pushed "2 checkpoint ref(s)" |

The `en-ca/` → `en/` move stayed local, so on GitHub the links noted above don't resolve until it is committed and pushed.

Then the owner asked: "update the session reports, commit and push them, lol". This update was committed and pushed the same way, and is not recorded again.

### Committing everything

The owner asked: "lets commit everything".

| Repository | Commit | What it holds |
|---|---|---|
| uc-processes | `5a38160` | *Process: Creating an Eleventy Theme* 0.1.0, with `CHANGELOG.md` entries for it and for the move |
| uc-automa | `c9e4ea3` | The `en-ca/` → `en/` move (77 renames), the owner's `.gitignore` change, an overview rename, a session note, and a `ROADMAP.md` item |

In uc-automa, the move makes `test_translations_report_nothing_for_en_ca` fail: 45 tests, 2 failures, against 1 before the move. The code still uses `en-ca` as its default language. The owner's call: "sounds like a roadmap issue for updating our script". So it is committed with the failure stated, and tracked in `uc-automa/ROADMAP.md` "Later": a decision record superseding 0004, then the code and tests. The other failure, `test_check_passes_on_uc_automa`, was already failing before the move.

Then the owner asked: "update the session reports and push all three repos". This update and the pushes are not recorded again.

Then: "add the failing test to the uc-automa roadmap". Before the move, `uc-automa check` on uc-automa itself had 2 findings, including `docs/overviews/` (added in `218bdec`), which the `uc-automa` layout doesn't describe. Since the move, it has 132. `uc-automa/ROADMAP.md` "Later" gained an item: fix the move first, then describe `overviews/` in `conf/layouts/uc-automa.yaml`. That change, these report updates and their pushes are not recorded again.

Then: "fix the en/ move in uc-automa code". The owner chose "option 1": `en` as both the folder and the language tag. They also gave a direction for later: "I think metadata checker should be it's own stand alone repository / executable. It should include all of the resources it needs and it should be configurable". Applying option 1 took `uc-automa check` on uc-automa from 132 findings to 63, and turned up two blockers:

- Layout paths marked `source-language` are checked against uc-automa's own text language, so downstream `en-ca` repositories would break on upgrading.
- The front-matter schema refuses `dc:language: "en"`.

The owner: "do nothing, if you think this issue needs to be resolved now or very soon add it to the roadmap". Both blockers went into `uc-automa/ROADMAP.md`, marked "resolve soon", with the standalone checker as the longer-term item. Then: "keep them, update the session reports, commit and push". So the partial changes were committed and pushed, with 4 failing tests. The details are in `uc-gradus/en/docs/sessions/2026-11-07-2113-session-report.md`.

Then: "create a markdown report on the metadata issue". The report is `uc-automa/en/docs/sessions/2026-11-07--metadata-language-check-v0.1.0.md`.

Then the owner had decision 0040 drafted in uc-automa: `uc-automa/en/docs/decisions/0040--language-root-per-repository-configuration-v0.1.0.md`, Proposed. After "lets drop language checking altogether" and "Just drop the language check. So no, the standalone checker should not check dc:language at all", it sets three things: `en` as uc-automa's root, layout paths checked against the repository's language, and `dc:language` required but never checked.

Then: "accept 0040 and implement the resolve soon items". 0040 was accepted, 0004 and 0007 were superseded, and the code, schema, directives (both to 0.2.0), roadmap and changelog were updated. uc-automa's check went from 64 findings to 4, which are the owner's content, and 44 of 45 tests pass. The details are in `uc-gradus/en/docs/sessions/2026-11-07-2113-session-report.md`.

Open items:

- [x] Commit and push the `en-ca/` → `en/` move. The owner asked: "commit and push the en-ca to en move in uc-processes".
- [x] Push.

### The `en-ca/` → `en/` move

This was committed together with this update to the report:

- **Moved:** the 10 files under `en-ca/docs/`, to the same paths under `en/docs/`, with content unchanged (four release guides and six processes).
- **Link fixed:** `README.md`'s link to *Process: From Question to Practice* now points at `en/docs/processes/`. That follows uc-automa 0002 and 0029: a rename updates every link to the file in the same commit.
- **Now resolve on GitHub:** the release process's links to the release guides, `licensing-a-repository-with-reuse-v0.1.0.md` and `creating-a-new-repository-with-uc-automa-v0.1.0.md`.
- **Not included:** `en/docs/processes/creating-an-eleventy-theme-v0.1.0.md`, a new file of the owner's with no `en-ca/` copy. It stays uncommitted.
- **Committed afterwards:** the owner then asked to "commit everything". The theme process was committed separately, with `CHANGELOG.md` gaining entries for it and for the move.
- **Left alone:** several moved documents still mention `en-ca/` in prose (for example, where new repositories keep their decisions) and in URLs into uc-automa. Those aren't links to the moved files.

## Observations left out of scope

- uc-processes and uc-automa both have uncommitted work in progress, which looks like moving `en-ca/` to `en/`. Nothing in it was touched.
- The uc-processes README's Processes table lists one process, and links to `en-ca/` paths.
- uc-automa 0016 (*Session records*) names session records `{language}/docs/sessions/{date}--{slug}{version}.md`. These files follow the owner's scheme from uc-gradus instead, `<timestamp>-session-prompt.md` and `<timestamp>-session-report.md`, as instructed.
