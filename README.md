# uc-processes

How Universal Cake's work is done, from the first question to what is in force. Vishpala creates and maintains these processes. Christopher Steel is their author.

## How it relates to uc-radar and uc-automa

uc-processes sits above the other two repositories and says how they fit into the work.

| Repository | What it does |
| --- | --- |
| uc-processes | Says how work is done |
| [uc-radar](https://github.com/steelcj/uc-radar) | Evaluates options. Its [evaluation lifecycle](https://github.com/steelcj/uc-radar/blob/main/en/docs/radar/uc-radar--evaluation-lifecycle-v0-1-0.md) is the evaluate step of the overall process |
| [uc-automa](https://github.com/steelcj/uc-automa) | Records what is in force, and offers help: templates, offerings and checks |

For now, uc-processes stands on its own and is not set up with `uc-automa init`. Its documents follow uc-automa's front matter fields and filename rule by hand.

## Processes

| Process | Version | Status |
| --- | --- | --- |
| [Process: From Question to Practice](en-ca/docs/processes/from-question-to-practice-v0.1.0.md) | 0.1.0 | Draft |

## Releases

Releases are cut with `cut-release.py` and published with `publish-release.py`, which uses `bump-version.py` for the version arithmetic. Run `python3 cut-release.py patch`, `minor` or `major` once `CHANGELOG.md`'s `[Unreleased]` section lists the changes, push `main` and the tag, then run `python3 publish-release.py`. `python3 test_publish_release.py` runs the publishing tests offline.

These scripts are unmodified copies from [sat-doc-automa](https://github.com/steelcj/sat-doc-automa) at commit `fc0b72a`. uc-automa 0026 replaces them with `uc-automa release` commands, planned for uc-automa 0.2.0, and they are kept unmodified so that its `retire-copied-scripts` offering can recognise and remove them then.

## License

The documents in this repository are licensed under the Creative Commons Attribution 4.0 International License, whose text is in [`LICENSE`](LICENSE).

This document, *uc-processes*, by **Christopher Steel**, with AI assistance from **Claude Opus 5.5 (Anthropic)**, is licensed under the [Creative Commons Attribution 4.0 International License](https://creativecommons.org/licenses/by/4.0/).
