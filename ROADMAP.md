# Roadmap

Forward plan only; completed items move to [CHANGELOG.md](CHANGELOG.md).

## Later

Release processes, from writing *Process: Releasing a Repository* on 2026-11-07:

- Decide the release key's allowed-signers principal: the release-key guide uses `vishpala-release`, and uc-automa 0028 and 0030 use the owner's email address. The decision is recorded in uc-automa, where it is also on the roadmap. Then update [Managing the Vishpala release key](en/docs/guides/releases/managing-the-vishpala-release-key-v0.1.3.1.md), and replace `<principal>` in [Process: Releasing a Repository](en/docs/processes/releasing-a-repository-v0.1.0.md).
- When uc-automa 0.3.0 ships `uc-automa release`, release uc-processes with it, and apply the `retire-copied-scripts` offering to remove `bump-version.py`, `cut-release.py`, `publish-release.py` and `test_publish_release.py`. This also resolves the copied scripts' GPG signing, where uc-automa 0032 requires SSH, and their release commit, which leaves out `README.md`'s install tag. Until then, the scripts are not copied into new repositories.
- Correct the README's Releases section: the scripts are replaced in uc-automa 0.3.0, not 0.2.0 (uc-automa 0039).
- Update *Process: Releasing a Repository* once the release keys are published and `uc-automa release` exists: drop the hand-cut path, and confirm the commands against the released tools.
