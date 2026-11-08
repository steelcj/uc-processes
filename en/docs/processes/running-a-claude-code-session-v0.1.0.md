---
title: "Process: Running a Claude Code Session"
dc:title: "Process: Running a Claude Code Session"
dc:creator: "Christopher Steel"
dc:contributor: "Claude Opus 5.5 (Anthropic), AI assistance"
dc:subject: "process, Claude Code, AI assistance, sessions, session prompt, session report, findings, decisions, audit trail"
dc:description: "How a Claude Code CLI session is run and recorded in a repository: file the session prompt, present findings before any change, ask the owner to decide, apply only what was confirmed and in scope, verify, keep the session report current after every step, and close by saying plainly whether the task is complete."
dc:date: "2026-11-07"
dc:type: "Text"
dc:format: "text/markdown"
dc:identifier: "urn:uuid:e1c5a2ea-d4da-4d69-b68d-9ab8cbce5b84"
dc:language: "en-CA"
dc:rights: "CC-BY-4.0"
sat:version: "0.1.0"
sat:status: "Draft"
---

# Process: Running a Claude Code Session

Version: 0.1.0
Status: Draft

## Purpose

This process keeps a person in charge of work done with Claude Code, and leaves a record of it that someone can read later. Claude can investigate, propose and carry out changes, but the owner decides what changes. Every session leaves two files behind: the prompt that started the work, and a report of what was found, decided, changed and left open.

The repository carries the process in its `CLAUDE.md`. Claude Code loads that file at the start of every session, so the rules apply without being pasted in each time.

The examples come from uc-gradus. Its session on 2026-11-07 covered a secrets audit before the repository was first published, then an audit of the repository against its README.

## Steps

### Put the process in the repository

Add a `CLAUDE.md` at the repository root that sets out the session files, how changes are made, and the repository's conventions. Link back to this process. If the README has a repository structure, add `CLAUDE.md` to it.

Gate: `CLAUDE.md` is at the repository root, and the README's structure lists it.

### File the session prompt

All session records live in `en/docs/sessions/`. Timestamps are local time, written `YYYY-MM-DD-HHMM`.

The task prompt is saved as `<timestamp>-session-prompt.md`, where the timestamp is when work on the prompt began. If the prompt arrives as a file somewhere else, move it there. If it arrives in the chat, save it there. When the start time is not known, Claude Code's own session transcript records when each message was sent.

In uc-gradus, the task prompt `task-normalize-uc-gradus-repository-to-match-readme.md` came in at the repository root. It was moved to:

```text
en/docs/sessions/2026-11-07-2109-session-prompt.md
```

The transcript gave the time: 02:09 UTC, which is 21:09 EST.

Gate: the prompt is in `en/docs/sessions/`, its timestamp is when work on it began, and anything that linked to its old path has been updated.

### Present findings before any change

Investigate first, then present what was found. Don't change anything at this step. Proposed changes are shown as a table:

```markdown
| File | Current | Required | Action |
|---|---|---|---|
```

An audit gets its own report in `en/docs/audits/<YYYY-MM-DD>-<audit-title>.md`. A report never reproduces a secret: it points to the file and line, and at most a few leading characters, enough to tell two values apart.

Gate: the owner has seen every finding and every proposed change, and no file has changed yet.

### Ask the owner to decide

Some findings need a person's decision. For example, the source of truth contradicts itself, or following it would break working code. Put each one to the owner as a clear question with the options laid out. Don't pick a side. If an instruction can be read two ways, ask which one is meant before acting on it.

In uc-gradus, the README listed robertdebock roles for jobs the working playbook already did with its own tasks. The owner decided that each conflicting file should be marked by renaming it `RESOLVE-<code-file-name>`, and the choice between role and README would be made later.

Gate: every decision the change depends on has been made by the owner, and is recorded word for word.

### Apply only what was confirmed, and only that

Apply a change only after the owner confirms it. Change only what the task or decision covers. Anything else noticed along the way goes in the report under "Observations left out of scope", unchanged.

A historical file is never rewritten to match the present. It moves into an `artifacts/` folder next to where it was, such as `en/docs/design/artifacts/`, and its first line becomes:

```markdown
> Historical. Superseded by README v0.3.0.
```

Don't commit or push unless the owner asks.

Gate: every applied change matches a confirmed table row, and nothing outside the task was changed.

### Verify the change

Check that the change did what it was meant to, with the strongest check that fits: a search showing no old values remain, a syntax check, a test run, or a rehash of a placeholder password hash. Say what the check covered and what it did not.

In uc-gradus, after renaming playbook files, `ansible-playbook --syntax-check` passed for both playbooks. The report says they were not run against a server.

Gate: each change has a stated check, and its result is in the report.

### Keep the session report current

The report is `en/docs/sessions/<timestamp>-session-report.md`. Start it once the first findings exist. Update it after every decision and every applied change, without waiting to be asked. For each decision, the report records:

- the question as it was put;
- the owner's response, word for word;
- what was applied, as a table of old and new;
- how it was checked;
- what was deliberately left unchanged, and why.

When a later step moves or renames a file the report already mentions, add a short note where it is mentioned. Don't rewrite the earlier entry.

Gate: the report matches the repository as it stands now, and every decision in it carries the owner's own words.

### Close the session

End by saying plainly whether the task is complete. If it is not, the report gets a section listing every remaining item, with its status, what blocks it, and the plan for doing it. The report also keeps a running "Open items" checklist.

In uc-gradus, the README-normalization task was not complete at the end of the session. The report's section 5 lists the twelve remaining changes, and the owner's plan to make them during an end-to-end run of the dev process.

Gate: the report states whether the task is complete, and every open item is listed with what it is waiting on.

## Principles

- A person decides. Claude proposes, applies what was confirmed, and records it.
- Findings come before changes, and every change can be traced to a confirmed table row.
- Scope is what was asked. Everything else is noted, not fixed.
- The owner's words are recorded as given, not paraphrased.
- History is kept, not rewritten: superseded files move to `artifacts/`.
- A report says what was checked and what was not. "Done" means verified.
- Secrets never appear in a record. Records point to where a secret is.

## Changelog

| Version | Date | Status | Notes |
| --- | --- | --- | --- |
| 0.1.0 | 2026-11-07 | Draft | Initial version, drawn from the uc-gradus session of 2026-11-07 |

## License

This document, *Process: Running a Claude Code Session*, by **Christopher Steel**, with AI assistance from **Claude Opus 5.5 (Anthropic)**, is licensed under the [Creative Commons Attribution 4.0 International License](https://creativecommons.org/licenses/by/4.0/).
