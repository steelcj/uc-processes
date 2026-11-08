---
title: "Process: Releasing a Repository"
dc:title: "Process: Releasing a Repository"
dc:creator: "Christopher Steel"
dc:contributor: "Claude Opus 5.5 (Anthropic), AI assistance"
dc:subject: "process, releases, release ceremony, VERSION, CHANGELOG, annotated tags, signed tags, release keys, uc-automa"
dc:description: "How a Vishpala repository is released: set it up for releases, check it before cutting, cut by hand until uc-automa 0.3.0 and with uc-automa release from then on, verify the release commit, push deliberately, publish, and let readers verify what they download."
dc:date: "2026-11-07"
dc:type: "Text"
dc:format: "text/markdown"
dc:identifier: "urn:uuid:4e707204-c42e-4943-91cc-4d43e9118eca"
dc:language: "en-CA"
dc:rights: "CC-BY-4.0"
sat:version: "0.1.0"
sat:status: "Draft"
---

# Process: Releasing a Repository

Version: 0.1.0
Status: Draft

## Purpose

This process makes every Vishpala release the same: a tag that matches exactly what the repository contained, a changelog that says what changed, and, once the release keys exist, signatures that let anyone confirm what they downloaded came from Vishpala.

The ceremony is decided in uc-automa <a id="cite-steel-2026a"></a>[(Steel, 2026a)](#ref-steel-2026a). It will be the `uc-automa release bump`, `cut` and `publish` commands, so that no repository holds a copy of release scripts. Those commands are planned for uc-automa 0.3.0 <a id="cite-steel-2026e"></a>[(Steel, 2026e)](#ref-steel-2026e). Until then, a release is cut by hand to the two release directives in force, as uc-automa 0.1.0 and 0.2.0 were. This process gives both ways, and marks each step that changes at 0.3.0.

Don't copy `bump-version.py`, `cut-release.py`, `publish-release.py` or `test_publish_release.py` into a new repository. uc-automa 0026 replaces them, and its `retire-copied-scripts` offering will remove existing copies. The copies in uc-processes are kept, unmodified, until then.

## Steps

### Set the repository up for releases

A repository created with uc-automa <a id="cite-steel-2026h"></a>[(Steel, 2026h)](#ref-steel-2026h) gets `VERSION`, planted at `0.1.0`, from the `version-file` offering, and `CHANGELOG.md`, in Keep a Changelog form, from the `changelog` offering. The repository owns both from then on.

Write changelog entries by hand as work happens, under `## [Unreleased]`. The ceremony only ever writes headings, never changelog prose.

Gate: `VERSION` holds the current version, and `CHANGELOG.md` has an `## [Unreleased]` section listing every change since the last release.

### Have the release key ready

One SSH release key signs every Vishpala release and every release tag <a id="cite-steel-2026d"></a>[(Steel, 2026d)](#ref-steel-2026d). Christopher Steel creates and holds it. The ceremony never creates, imports or stores a key. How to make it, back it up and protect it is in [Managing the Vishpala release key](../guides/releases/managing-the-vishpala-release-key-v0.1.3.1.md) and [Generating a strong passphrase](../guides/releases/generating-a-strong-passphrase-v0.1.1.md).

Its public half is published as an allowed-signers line in two places on different hosts: `release-keys/` in uc-automa, and `https://vishpala.com/release-keys` <a id="cite-steel-2026c"></a>[(Steel, 2026c)](#ref-steel-2026c). Neither exists yet.

**Decision to make, then implement.** An allowed-signers line starts with a principal, the name that `ssh-keygen -Y verify -I <principal>` checks. The release-key guide uses `vishpala-release`. uc-automa 0028 and 0030 use the owner's email address. A decision record in uc-automa settles which one it is. Then `release-keys/`, the READMEs' "Get the release key" sections and the guide follow it. Until then, the commands below write `<principal>`. The item is tracked in uc-processes' and uc-automa's `ROADMAP.md`.

Until the key is published, releases are unsigned, as uc-automa 0.2.0 was [(Steel, 2026e)](#ref-steel-2026e).

Gate: either the release key is published in both places and its principal is decided, or the release is knowingly unsigned.

### Check before cutting

A release is a conforming, committed state of the repository. Before cutting:

```sh
git status --porcelain   # must print nothing
uc-automa check          # must pass
git tag -l "v$(cat VERSION)"   # check the tags: the new version must not be among them
```

- The working tree is clean: nothing uncommitted, staged or not, and no untracked file that git does not ignore.
- `## [Unreleased]` has content.
- No tag exists for the version about to be released. Tags are never reused.
- If the release adds or changes third-party material, the licensing audit in [Process: Licensing a Repository with REUSE](licensing-a-repository-with-reuse-v0.1.0.md) has been done.

Gate: the tree is clean, `uc-automa check` passes, `[Unreleased]` has entries, and the target tag does not exist.

### Cut the release

The cut follows two directives that `uc-automa check` enforces: `release/annotated-version-tags` <a id="cite-steel-2026f"></a>[(Steel, 2026f)](#ref-steel-2026f) and `release/readme-install-tag` <a id="cite-steel-2026g"></a>[(Steel, 2026g)](#ref-steel-2026g).

#### Until uc-automa 0.3.0: cut by hand

For a release of `example-repo` from 0.1.0 to 0.2.0:

1. **Bump `VERSION`.**

   ```sh
   printf '0.2.0\n' > VERSION
   ```

2. **Roll the changelog.** Rename `## [Unreleased]` to the version, dated today in local time (`date +%F`), and leave a fresh, empty `## [Unreleased]` above it. Don't add Keep a Changelog's link references until the repository has a remote.

   ```markdown
   ## [Unreleased]

   ## [0.2.0] - 2026-11-07

   ### Added

   - …
   ```

3. **Update the README's own install references.** Wherever `README.md` installs this repository from a tag, such as `git+https://github.com/steelcj/example-repo@v0.1.0`, change the tag to `v0.2.0`. A dependency's tag is changed by hand with its version range, not as part of the cut.

4. **Commit surgically.** Commit those three files only, never everything.

   ```sh
   git add VERSION CHANGELOG.md README.md
   git commit -m "Release 0.2.0"
   ```

5. **Tag.** Create an annotated tag `v` + `VERSION`, with the message naming the repository and the version:

   ```sh
   git tag -a v0.2.0 -m "example-repo 0.2.0"
   ```

   Once the release key is published, sign the tag with it instead [(Steel, 2026d)](#ref-steel-2026d):

   ```sh
   git -c gpg.format=ssh -c user.signingkey=~/.ssh/vishpala-release.pub tag -s v0.2.0 -m "example-repo 0.2.0"
   ```

6. **Check the cut.**

   ```sh
   uc-automa check
   git show --stat v0.2.0   # the release commit changes only VERSION, CHANGELOG.md and README.md
   ```

7. **Stop. Don't push yet.**

#### From uc-automa 0.3.0: `uc-automa release cut`

```sh
uc-automa release cut patch|minor|major|X.Y.Z
```

It does the same steps, and refuses at each one: it refuses a dirty tree, checks the release, requires `uc-automa check` to pass, bumps, rolls the changelog, rewrites the README's install tag, commits surgically, makes a signed annotated tag, checks the tag, and stops before pushing [(Steel, 2026a)](#ref-steel-2026a); <a id="cite-steel-2026b"></a>[(Steel, 2026b)](#ref-steel-2026b).

Gate: the tag `vX.Y.Z` is annotated, its message names the repository and the version, it points at a release commit that changes only `VERSION`, `CHANGELOG.md` and `README.md`, and `uc-automa check` passes.

### Verify the release commit before pushing

The tag stays local until the release has been verified. Then a fault can be fixed without a public tag ever pointing at it.

- **Until uc-automa 0.3.0:** install the release the way the README says, from a local clone holding the tag, and run it.
- **From uc-automa 0.3.0, for Python tools:** a verification run on the release commit, on at least one platform below 1.0.0, and on Linux, Windows and macOS from 1.0.0. Its wheels must match the record that `release publish` will check [(Steel, 2026b)](#ref-steel-2026b).

Gate: the release commit has been installed and run as a reader would, and passed.

### Push deliberately

Pushing is a person's act, never part of the ceremony:

```sh
git push origin main v0.2.0
```

Gate: `main` and the tag are on the remote, and point at the same commit as they do locally.

### Publish

- **Until uc-automa 0.3.0:** the pushed tag is the release. There is no archive, no wheel and no signature, as for uc-automa 0.2.0 [(Steel, 2026e)](#ref-steel-2026e).
- **From uc-automa 0.3.0:** `uc-automa release publish` confirms the tag is on the remote. It builds the archive twice from the tag's tree and refuses any byte difference, and takes the release notes from the changelog. It writes `SHA256SUMS`, signs it with the SSH release key as `SHA256SUMS.sig`, and publishes through the `gh` or plain-directory backend. For Python tools it also adds pinned, reproducible wheels and `requirements-offline.txt` with its signature [(Steel, 2026a)](#ref-steel-2026a); [(Steel, 2026b)](#ref-steel-2026b). The ceremony never reads, stores, logs or echoes a credential.

Gate: the release is published, and from uc-automa 0.3.0, every file it carries is listed in a signed `SHA256SUMS`.

### Let readers verify the release

Once the release key is published, a reader checks a release like this. `<principal>` is waiting on the decision above.

```sh
git clone --depth 1 https://github.com/steelcj/uc-automa uc-automa-keys
ssh-keygen -lf uc-automa-keys/release-keys/allowed-signers
```

Compare the fingerprint that `ssh-keygen` prints with the one at `https://vishpala.com/release-keys`. If they differ, stop. Then:

```sh
git -c gpg.ssh.allowedSignersFile=uc-automa-keys/release-keys/allowed-signers tag -v v0.2.0
ssh-keygen -Y verify -f uc-automa-keys/release-keys/allowed-signers -I <principal> -n vishpala-release -s SHA256SUMS.sig < SHA256SUMS
```

The result of `git tag -v` is its exit status, not its message: 0 means a key in the file signed the tag. On Windows, `git tag -v` does not finish for SSH-signed tags, but the `ssh-keygen -Y verify` check of files works there [(Steel, 2026d)](#ref-steel-2026d).

Gate: the repository's README tells readers how to get the key and verify a release, from the first signed release on.

### When a cut fails

- **Not yet pushed:** the tag and the release commit were never public. Remove them, fix the fault, and cut the same version again:

  ```sh
  git tag -d v0.2.0
  git reset --keep HEAD~1
  ```

- **Already pushed:** a pushed tag is never reused or moved. Fix the fault and release the next version.

Gate: no pushed tag has ever been deleted, moved or reused.

## Open items

These were found while writing this process, on 2026-11-07. They are tracked in uc-processes' `ROADMAP.md`, and left unchanged here:

1. **Signing in the copied scripts.** `publish-release.py` signs `SHA256SUMS` with GPG, and `cut-release.py` does not sign tags. uc-automa 0032 requires SSH signatures and SSH-signed tags. `uc-automa release` replaces the scripts at 0.3.0.
2. **The release key's principal.** It is a decision to make, as described in "Have the release key ready".
3. **The release commit in the copied scripts.** `cut-release.py` commits only `VERSION` and `CHANGELOG.md`. The `release/readme-install-tag` directive also needs `README.md` updated in the release commit, so a scripted cut of a repository whose README installs its own tag fails `uc-automa check`.
4. **The uc-processes README** says the scripts are replaced "in uc-automa 0.2.0". Since uc-automa 0039, it is 0.3.0.
5. **uc-processes holds the copied scripts.** uc-automa 0026 says no repository should. The `retire-copied-scripts` offering removes them at uc-automa 0.3.0.

## Principles

- A tag marks exactly one release: what was committed, nothing more.
- A release is a conforming state: `uc-automa check` passes before and after the cut.
- Pushing is a person's decision, and comes only after the release is verified.
- A pushed tag is never reused.
- The ceremony writes headings, never changelog prose, and never handles a key or a credential.
- Release tooling is installed, never copied into a repository.

## References

<a id="ref-steel-2026a"></a>Steel, C. (2026a). *Decision: Release ceremony* (Version 0.1.0). uc-automa 0026. https://github.com/steelcj/uc-automa/blob/main/en-ca/docs/decisions/0026--release-ceremony-v0.1.0.md
[Return to citation](#cite-steel-2026a)

<a id="ref-steel-2026b"></a>Steel, C. (2026b). *Decision: Wheels with each release* (Version 0.1.0). uc-automa 0028. https://github.com/steelcj/uc-automa/blob/main/en-ca/docs/decisions/0028--wheels-with-each-release-v0.1.0.md
[Return to citation](#cite-steel-2026b)

<a id="ref-steel-2026c"></a>Steel, C. (2026c). *Decision: Where the release keys are published* (Version 0.1.0). uc-automa 0030. https://github.com/steelcj/uc-automa/blob/main/en-ca/docs/decisions/0030--where-the-release-keys-are-published-v0.1.0.md
[Return to citation](#cite-steel-2026c)

<a id="ref-steel-2026d"></a>Steel, C. (2026d). *Decision: SSH signatures required with wheels, and SSH-signed tags* (Version 0.1.0). uc-automa 0032. https://github.com/steelcj/uc-automa/blob/main/en-ca/docs/decisions/0032--ssh-signatures-required-with-wheels-and-ssh-signed-tags-v0.1.0.md
[Return to citation](#cite-steel-2026d)

<a id="ref-steel-2026e"></a>Steel, C. (2026e). *Decision: Hand-cut 0.2.0* (Version 0.1.0). uc-automa 0039. https://github.com/steelcj/uc-automa/blob/main/en-ca/docs/decisions/0039--hand-cut-0-2-0-v0.1.0.md
[Return to citation](#cite-steel-2026e)

<a id="ref-steel-2026f"></a>Steel, C. (2026f). *Release: Annotated version tags* (Version 0.2.0) [Directive]. uc-automa. https://github.com/steelcj/uc-automa/blob/main/en-ca/docs/directives/release/annotated-version-tags-v0.2.0.md
[Return to citation](#cite-steel-2026f)

<a id="ref-steel-2026g"></a>Steel, C. (2026g). *Release: README install tag* (Version 0.2.0) [Directive]. uc-automa. https://github.com/steelcj/uc-automa/blob/main/en-ca/docs/directives/release/readme-install-tag-v0.2.0.md
[Return to citation](#cite-steel-2026g)

<a id="ref-steel-2026h"></a>Steel, C. (2026h). *Process: Creating a new repository with uc-automa* (Version 0.1.0) [Draft]. uc-processes. [creating-a-new-repository-with-uc-automa-v0.1.0.md](creating-a-new-repository-with-uc-automa-v0.1.0.md)
[Return to citation](#cite-steel-2026h)

## Changelog

| Version | Date | Status | Notes |
| --- | --- | --- | --- |
| 0.1.0 | 2026-11-07 | Draft | Initial version, drawn from uc-automa 0026, 0028, 0030, 0032 and 0039, the two release directives, and the copied release scripts |

## License

This document, *Process: Releasing a Repository*, by **Christopher Steel**, with AI assistance from **Claude Opus 5.5 (Anthropic)**, is licensed under the [Creative Commons Attribution 4.0 International License](https://creativecommons.org/licenses/by/4.0/).
