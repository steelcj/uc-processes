---
title: Generating a strong passphrase
description: How to set up a trusted wordlist and generate a random, memorable seven-word passphrase, such as the one protecting the Vishpala release key.
version: 0.1.0
date: 2026-10-22
author: Christopher Steel
---

# Generating a strong passphrase

This guide sets up what you need once, then shows how to generate a passphrase of seven random words that is strong enough to protect a signing key and simple enough to remember. It was written for the passphrase on the Vishpala release key, described in [Managing the Vishpala release key](managing-the-vishpala-release-key-v0.1.3.md), and works for any secret you must keep in your head, such as the master password of a password manager.

When you finish, you will have:

- a copy of the EFF long wordlist on your trusted machine, checked and with its hash recorded;
- a passphrase of seven words, chosen at random from that list;
- the passphrase stored safely and memorized.

## How it works

The method is Diceware: each word is picked at random from a published list of 7,776 words, so every word adds about 12.9 bits of randomness, and seven words give about 90 bits. The words are common and easy to type, which makes the result memorable, but they are chosen by dice or by the computer's secure random source, never by you. Using a wordlist of the same length gives the same strength, whichever list you choose [(Electronic Frontier Foundation [EFF], n.d.)](#ref-eff-dice)<a id="cite-eff-dice"></a>.

EFF suggests six words for most uses [(EFF, 2025)](#ref-eff-diceware-pdf)<a id="cite-eff-diceware-pdf"></a>. Seven gives extra margin for a key that signs releases for years.

The strength comes only from the random choice. Spaces or other separators between words, capital letters, or added symbols make a passphrase harder to remember without making a randomly chosen one meaningfully harder to guess.

## Before you start

Do this on the same trusted machine you use for the release key. Do it with nobody watching your screen, and with no screen sharing or recording running.

Never take a passphrase from a website, from a chat, or from an AI assistant. A passphrase that has been shown anywhere outside your machine is no longer secret.

## Set up once

### Get the wordlist

Download EFF's Long Wordlist, `eff_large_wordlist.txt`, from [eff.org/dice](https://www.eff.org/dice), and keep it in its own folder:

```sh
mkdir -p ~/passphrase
mv ~/Downloads/eff_large_wordlist.txt ~/passphrase/
```

The wordlist is public, so it is not secret, but it must be the real list.

### Check it

Each line holds a five-digit dice number, a tab, and a word. Check that there are 7,776 lines, that every line has that form, and that no word repeats:

```sh
wc -l < ~/passphrase/eff_large_wordlist.txt
awk -F'\t' '$1 !~ /^[1-6][1-6][1-6][1-6][1-6]$/ || NF != 2' ~/passphrase/eff_large_wordlist.txt | head
cut -f2 ~/passphrase/eff_large_wordlist.txt | sort | uniq -d | head
```

Expect `7776` from the first command and no output from the other two.

### Record its hash

```sh
sha256sum ~/passphrase/eff_large_wordlist.txt
```

Write the hash down, for example in your password manager next to where the passphrase will go. The next time you use the list, run the command again: if the hash has changed, find out why before using it.

### Install KeePassXC (optional)

If you will use KeePassXC to generate or store the passphrase, install it from your distribution. On Linux Mint, Debian or Ubuntu:

```sh
sudo apt install keepassxc
```

## Generate the passphrase

Use one of these three methods. Generate once and accept the result. Rerolling once because a word is hard to spell is fine; rerolling until you like the words narrows the choice toward words people favour.

### With dice (strongest)

No software is involved, so nothing on the machine can see or influence the result. You need five six-sided dice, or one die rolled five times.

1. Decide before you roll how you will read the dice, for example left to right.
2. Roll five dice and read them as a five-digit number, such as `43463`.
3. Find that number in `eff_large_wordlist.txt` and write the word down on paper.
4. Repeat until you have seven words.

This matches EFF's own instructions [(EFF, n.d.)](#ref-eff-dice). Destroy the paper once the passphrase is stored and memorized.

### With a script

This uses Python's `secrets` module, which draws from the operating system's cryptographic random source, unlike the `random` module [(Python Software Foundation, 2026)](#ref-python-secrets)<a id="cite-python-secrets"></a>. It also checks the wordlist before using it.

```sh
python3 - ~/passphrase/eff_large_wordlist.txt <<'EOF'
import secrets, sys
with open(sys.argv[1]) as f:
    words = [line.split()[1] for line in f if line.strip()]
assert len(words) == 7776 and len(set(words)) == 7776, "unexpected wordlist"
print(" ".join(secrets.choice(words) for _ in range(7)))
EOF
```

The passphrase is printed only on screen. Do not redirect it to a file. When you have stored it, clear the terminal and close it, so it does not stay in the scrollback.

### With KeePassXC

1. Open KeePassXC and go to Tools, then Password Generator.
2. Choose the Passphrase tab.
3. Set the word count to 7.
4. Check that the wordlist selector shows the EFF long list. If it does not, add `~/passphrase/eff_large_wordlist.txt` with the button beside the selector.
5. Choose a space as the word separator and lower case for the words.

These options are described in the KeePassXC user guide [(KeePassXC Team, n.d.)](#ref-keepassxc-guide)<a id="cite-keepassxc-guide"></a>. The generator can copy the passphrase to the clipboard. If you use that, clear the clipboard afterwards, or rely on KeePassXC's automatic clipboard clearing.

## Use it

For a new release key, enter the passphrase when `ssh-keygen` asks for one, as in [Managing the Vishpala release key](../release/managing-the-vishpala-release-key-v0.1.3.md).

To put a new passphrase on a key that already exists, change it in place:

```sh
ssh-keygen -p -f ~/.ssh/vishpala-release
```

It asks for the old passphrase, then the new one twice. Existing backups of the key file still use the old passphrase: replace them with copies of the updated file, and test them as the release key guide describes.

## Store it

Store the passphrase somewhere safe and separate from the key and its backups, such as an entry in your password manager named after the key, for example `vishpala-release SSH key passphrase`.

The master password of the password manager cannot be stored inside it. If you generate one with this guide, memorize it, and keep a written copy only where you would keep other irreplaceable documents.

## Memorize it

- Make the seven words, in their order, into one absurd scene or a short story you can picture.
- Type it several times today, then again tomorrow and in a few days, until it comes without thinking.
- Check your recall a week later against the stored copy.

## What to avoid

- Choosing, swapping or reordering words yourself.
- Song lines, quotations, names or anything meaningful to you.
- Passphrases generated by a website, a chat or an AI assistant.
- Reusing this passphrase for anything else.
- Saving it in a plain text file, a note, a commit or a message.

## Changes

- **0.1.0.** First version.

## References

<a id="ref-eff-diceware-pdf"></a>Electronic Frontier Foundation. (2025). *Diceware* [Fact sheet]. https://www.eff.org/files/2025/08/19/diceware.pdf
[Return to citation](#cite-eff-diceware-pdf)

<a id="ref-eff-dice"></a>Electronic Frontier Foundation. (n.d.). *EFF dice-generated passphrases*. Retrieved October 22, 2026, from https://www.eff.org/dice
[Return to citation](#cite-eff-dice)

<a id="ref-keepassxc-guide"></a>KeePassXC Team. (n.d.). *KeePassXC user guide: Password generator*. Retrieved October 22, 2026, from https://keepassxc.org/docs/KeePassXC_UserGuide
[Return to citation](#cite-keepassxc-guide)

<a id="ref-python-secrets"></a>Python Software Foundation. (2026). *secrets: Generate secure random numbers for managing secrets*. Python documentation. https://docs.python.org/3/library/secrets.html
[Return to citation](#cite-python-secrets)

## License

This document is licensed under the Creative Commons Attribution 4.0 International License (CC BY 4.0).
