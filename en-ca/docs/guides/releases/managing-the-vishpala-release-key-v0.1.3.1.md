---
title: Managing the Vishpala release key
description: How to create, test, back up and publish the SSH key that signs Vishpala releases, and how to rotate or revoke it.
version: 0.1.3
date: 2026-10-22
author: Christopher Steel
---

# Managing the Vishpala release key

This guide walks through creating the SSH key that signs Vishpala releases, checking that it works, keeping it safe, and publishing its public half so anyone can verify a release, and later rotating or revoking it. It implements the key requirements in [uc-automa](https://github.com/steelcj/uc-automa) decisions 0030 (where release keys are published) and 0032 (SSH signing), which are kept in uc-automa's [decision records](https://github.com/steelcj/uc-automa/tree/main/en-ca/docs/decisions).

When you finish, you will have:

- a private key that never leaves your trusted machine and its offline backups;
- a public key file, `~/.ssh/vishpala-release.pub`;
- an allowed-signers file, `~/.ssh/vishpala-release.allowed-signers`, holding the single line verifiers use to decide which key may sign;
- a fingerprint, the short value people compare by eye;
- the allowed-signers line and its fingerprint published in two independent places: uc-automa's `release-keys/` directory and [vishpala.com/release-keys](https://vishpala.com/release-keys).

## About the example values

The examples in this guide use one throwaway key, made only for illustration. Its private half has been deleted, so it signs nothing. Its values are:

- fingerprint: `SHA256:Lf7ytXP2LEw4d1v6i0H697B0hj+gwWln9/KKWS5u+yk`
- allowed-signers line: `vishpala-release namespaces="vishpala-release,git",valid-after="20261012Z" ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAIDEH93Wu4xJnkSeSUdCtWtqit0pesQlAlAHnEZpeLDGj`

Your own key will give different values. Never copy these into uc-automa or onto vishpala.com.

## Before you start

Do this on a machine you trust, such as your own laptop. Never generate or store the key inside a test instance, a container, a virtual machine used for testing, or a folder that syncs to a cloud service.

Check that your OpenSSH is recent enough to sign files. Signing and verifying with `ssh-keygen -Y` needs OpenSSH 8.1 or later.

```sh
ssh -V
```

Decide on the kind of key:

- **A software key (recommended to start).** An ed25519 key stored in a file and protected by a passphrase. Simple to back up and to use on any machine.
- **A hardware key.** An `ed25519-sk` key tied to a FIDO2 security key. The secret half cannot be copied off the device, which is stronger, but if the device is lost the key is gone and you must rotate to a new one. Keep a second device or accept that risk.

Every command in this guide uses absolute paths under `~/.ssh/` or `/tmp/`, so it works from any directory, in any terminal, even if you stop and come back later.

## Generate the key

For a software key:

```sh
ssh-keygen -t ed25519 -a 100 -C vishpala-release -f ~/.ssh/vishpala-release
```

- `-t ed25519` chooses the key type.
- `-a 100` raises the number of key derivation rounds that protect the private key file, making a stolen file much slower to attack. The default is 32 [(OpenBSD, 2026)](#ref-openbsd-ssh-keygen)<a id="cite-openbsd-ssh-keygen"></a>.
- `-C vishpala-release` labels the key so you can recognize it later.
- `-f ~/.ssh/vishpala-release` keeps it separate from your everyday SSH key.

When asked, set a strong passphrase. A lost passphrase cannot be recovered, so before you go further, store it somewhere safe and separate from the key, such as a password manager. To make one, follow [Generating a strong passphrase](../security/generating-a-strong-passphrase-v0.1.1.md).

For a hardware key, plug in the device and run:

```sh
ssh-keygen -t ed25519-sk -O verify-required -C vishpala-release -f ~/.ssh/vishpala-release
```

`-O verify-required` makes the device ask for its PIN on every signature, not just a touch.

Either way you now have two files:

- `~/.ssh/vishpala-release`, the private key (or, for a hardware key, the handle that works only with the device). Keep it private.
- `~/.ssh/vishpala-release.pub`, the public key. This is the only part you publish.

## Back up the private key

For a software key, make two copies of the private key file on offline media, such as two USB drives kept in different places. The file is already encrypted with your passphrase, so the passphrase must be stored separately from the drives.

Before relying on a backup, check that it works: copy it to a temporary location and run `ssh-keygen -y -f <copy>`. It should ask for the key's passphrase and print the same public key as `~/.ssh/vishpala-release.pub`. Then delete the temporary copy.

For a hardware key, a copy of the handle file is useless without the device, so the backup is a second device, or a plan to rotate to a new key if the device is lost.

## Get the fingerprint

```sh
ssh-keygen -lf ~/.ssh/vishpala-release.pub
```

The output has this form, shown here for the example key:

```text
256 SHA256:Lf7ytXP2LEw4d1v6i0H697B0hj+gwWln9/KKWS5u+yk vishpala-release (ED25519)
```

The part starting `SHA256:` is the fingerprint. It is what people compare against the website. The comment, `vishpala-release`, is only a label and proves nothing about the key.

## Write the allowed-signers line

An allowed-signers line names who may sign, for what purposes, and with which key. Its fields are the principal, options, the key type and the key itself [(OpenBSD, 2026)](#ref-openbsd-ssh-keygen).

The Vishpala line uses the principal `vishpala-release`, and allows two namespaces:

- `vishpala-release`, for signing release files such as `SHA256SUMS`;
- `git`, because git checks tag signatures under that namespace. Without it, git refuses the key for tags.

Create the file from your public key. It is public material, so it lives beside the `.pub` file:

```sh
printf 'vishpala-release namespaces="vishpala-release,git",valid-after="%s" %s\n' \
  "$(date -u +%Y%m%dZ)" \
  "$(cut -d' ' -f1,2 ~/.ssh/vishpala-release.pub)" > ~/.ssh/vishpala-release.allowed-signers
cat ~/.ssh/vishpala-release.allowed-signers
```

The result is a single line. For the example key:

```text
vishpala-release namespaces="vishpala-release,git",valid-after="20261012Z" ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAIDEH93Wu4xJnkSeSUdCtWtqit0pesQlAlAHnEZpeLDGj
```

`valid-after` records the key's first day of use, in UTC (the trailing `Z`). It matters when the key is eventually replaced; see [Rotating or revoking the key](#rotating-or-revoking-the-key).

## Get the fingerprint from an allowed-signers line

`ssh-keygen -lf` cannot read an allowed-signers file directly: it stops with "is not a public key file", because the line starts with a principal and options instead of a key. Pass it only the last two fields, the key type and the key.

On Linux or macOS, for your own file:

```sh
awk '{print $(NF-1), $NF}' ~/.ssh/vishpala-release.allowed-signers | ssh-keygen -lf -
```

For any other allowed-signers file, such as one downloaded from uc-automa, replace the path with that file's.

It prints the same fingerprint as the `.pub` file, with "no comment" in place of `vishpala-release`, because the line carries no comment. That difference does not affect the fingerprint.

On Windows, in Command Prompt, the equivalent writes the two fields to a file and fingerprints that. Run it in the directory holding the file you are checking, here named `allowed-signers`:

```bat
for /f "tokens=3,4" %a in (allowed-signers) do @echo %a %b> key.pub
ssh-keygen -lf key.pub
```

The Windows form has not yet been confirmed on the clean Windows snapshot. Confirm it there before publishing it anywhere a reader will rely on it.

Use the Linux or macOS command now to check your own file: the fingerprint it prints must match the one from your `.pub` file.

## Test the key before publishing

Test both a good signature and a bad one. Always judge the result by the exit code, never by the message: a signature from the wrong key can print "Good … signature" and still exit 1.

Sign a test file:

```sh
echo "release key test" > /tmp/release-key-test.txt
ssh-keygen -Y sign -f ~/.ssh/vishpala-release -n vishpala-release /tmp/release-key-test.txt
```

This writes `/tmp/release-key-test.txt.sig`. Verify it:

```sh
ssh-keygen -Y verify -f ~/.ssh/vishpala-release.allowed-signers -I vishpala-release -n vishpala-release \
  -s /tmp/release-key-test.txt.sig < /tmp/release-key-test.txt
echo "exit code: $?"
```

Expect `exit code: 0`. The message also names the key's fingerprint, which should match yours. Now change the file and verify again:

```sh
echo "tampered" >> /tmp/release-key-test.txt
ssh-keygen -Y verify -f ~/.ssh/vishpala-release.allowed-signers -I vishpala-release -n vishpala-release \
  -s /tmp/release-key-test.txt.sig < /tmp/release-key-test.txt
echo "exit code: $?"
```

Expect a non-zero exit code. If it is 0, stop and find out why before publishing anything.

Then test tag signing in a scratch repository. Do this on Linux or macOS: on Windows, `git tag -v` currently hangs with SSH-signed tags. Git needs a name and email to make a tag; if you have none set globally, the commands below give the scratch repository its own.

```sh
mkdir /tmp/tag-test && cd /tmp/tag-test && git init -q
git config user.name "Christopher Steel"
git config user.email "release-key-test@invalid"
git commit -q --allow-empty -m "test"
git config gpg.format ssh
git config user.signingKey ~/.ssh/vishpala-release.pub
git config gpg.ssh.allowedSignersFile ~/.ssh/vishpala-release.allowed-signers
git tag -s test-tag -m "test"
git tag -v test-tag
echo "exit code: $?"
```

Expect `exit code: 0`. These settings are stored only in the scratch repository; the settings themselves are described in the git documentation [(Git Project, 2026)](#ref-git-config)<a id="cite-git-config"></a>.

When you are done, leave the scratch repository and delete it and the test files:

```sh
cd ~ && rm -rf /tmp/tag-test /tmp/release-key-test.txt /tmp/release-key-test.txt.sig
```

## Publish the public key

Publish in two independent places, so a change in one can be noticed against the other.

### In uc-automa

Give the agent, or add yourself, the line from `~/.ssh/vishpala-release.allowed-signers`. It goes in uc-automa's `release-keys/allowed-signers`, with the fingerprint in `release-keys/README.md`, as decision 0030 describes. Review that change before it is pushed: compare the line character by character with your own file.

### On vishpala.com

Create the page at [vishpala.com/release-keys](https://vishpala.com/release-keys). It should show:

- the fingerprint;
- the full allowed-signers line, in a code block so it can be copied exactly;
- how to get the fingerprint from that line, as in [Get the fingerprint from an allowed-signers line](#get-the-fingerprint-from-an-allowed-signers-line);
- the key type and the date it came into use;
- a short history table recording any future rotation.

A starting point for the page, filled in with the example key. The banner at the top is part of the template: delete it only after every value has been replaced with your real key's.

````markdown
# Vishpala release keys

> **Example only.** This key is a throwaway made for illustration, and its private half has been deleted. It signs nothing. Replace every value below with the real release key before publishing.

Vishpala releases and release tags are signed with the SSH key below. Before trusting a release, compare this fingerprint with the fingerprint of the key in uc-automa's `release-keys/allowed-signers`. They must match exactly.

On Linux or macOS, from a clone of uc-automa:

```sh
awk '{print $(NF-1), $NF}' release-keys/allowed-signers | ssh-keygen -lf -
```

## Current key

- Fingerprint: `SHA256:Lf7ytXP2LEw4d1v6i0H697B0hj+gwWln9/KKWS5u+yk`
- Type: ED25519
- In use since: 2026-10-12

```text
vishpala-release namespaces="vishpala-release,git",valid-after="20261012Z" ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAIDEH93Wu4xJnkSeSUdCtWtqit0pesQlAlAHnEZpeLDGj
```

## Key history

| Fingerprint | In use from | Retired | Reason |
| --- | --- | --- | --- |
| `SHA256:Lf7ytXP2LEw4d1v6i0H697B0hj+gwWln9/KKWS5u+yk` | 2026-10-12 | | |
````

Add the Windows command to the page once it has been confirmed on Windows.

### Cross-check once both are live

After both are published, check them against each other from a clean copy:

```sh
git clone --depth 1 https://github.com/steelcj/uc-automa /tmp/uc-automa-check
awk '{print $(NF-1), $NF}' /tmp/uc-automa-check/release-keys/allowed-signers | ssh-keygen -lf -
```

The fingerprint printed must match the one on vishpala.com exactly. Delete `/tmp/uc-automa-check` afterwards.

## What may leave your machine

Safe to share, and meant to be public:

- `~/.ssh/vishpala-release.pub`;
- `~/.ssh/vishpala-release.allowed-signers` and the line in it;
- the fingerprint.

Never share, copy into a repository, paste into a chat or hand to an agent:

- `~/.ssh/vishpala-release`, the private key;
- its passphrase.

## Rotating or revoking the key

Replace the key on a schedule you choose, and at once if it may have been exposed.

**Routine rotation.** Generate a new key with this guide, then:

- add the new key's line with its own `valid-after`;
- give the old key's line a `valid-before` set to its retirement date, and keep it in the file so older releases can still be checked;
- record both in the history table on vishpala.com.

One caution: `ssh-keygen -Y verify` checks a key's validity against the current time unless the verifier passes `-O verify-time=` with another date [(OpenBSD, 2026)](#ref-openbsd-ssh-keygen). A plain file signature does not carry its own date, so once an old key has a `valid-before` in the past, checking an old release with it fails unless the steps supply the release date. Decision 0030 should settle how verification steps handle this before the first rotation. Git tags carry their own date; check how your git version uses it before relying on it.

**Suspected compromise.** Treat any signature made after the possible exposure as untrustworthy. Set the old line's `valid-before` to the last date you know the key was safe, publish the change on vishpala.com with the reason, generate a new key, and re-sign any releases made after that date.

## Changes

- **0.1.3.** Renamed Managing the Vishpala release key, for its new home in uc-processes, to cover the key's whole life, including rotation. Gave the allowed-signers file a fixed path, `~/.ssh/vishpala-release.allowed-signers`, used by every command. The tag test no longer relies on `$OLDPWD`, which the shell sets to the previous directory on each `cd` and which pointed at the wrong place, or nowhere, if the reader changed directory or opened a new terminal between steps. Added a cleanup step that leaves the scratch repository before deleting it. Clarified that the passphrase is stored, not checked, in a password manager. Linked uc-automa's decision records, since the guide now lives outside uc-automa, and the new passphrase guide.
- **0.1.2.** Fixed the fingerprint check: `ssh-keygen -lf` cannot read an allowed-signers file, so the guide and the page template now extract the key first, with a new section explaining how and a Windows form still to be confirmed. Made every example use one clearly labelled throwaway key. Added a git identity to the tag test, which fails without one, and backup advice for hardware keys.
- **0.1.1.** Added example values.
- **0.1.0.** First version.

## References

<a id="ref-git-config"></a>Git Project. (2026). *git-config documentation* (Version 2.56.0). https://git-scm.com/docs/git-config
[Return to citation](#cite-git-config)

<a id="ref-openbsd-ssh-keygen"></a>OpenBSD. (2026, October 1). *ssh-keygen(1)* [Manual page]. OpenBSD manual page server. https://man.openbsd.org/ssh-keygen
[Return to citation](#cite-openbsd-ssh-keygen)

## License

This document is licensed under the Creative Commons Attribution 4.0 International License (CC BY 4.0).
