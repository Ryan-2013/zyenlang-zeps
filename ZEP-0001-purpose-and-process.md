# ZEP-0001 — Purpose and Process

**English** | [繁體中文](ZEP-0001-purpose-and-process.zh-TW.md)

| Field | Value |
|---|---|
| **ZEP** | 0001 |
| **Title** | Purpose and Process |
| **Author** | zuenchen, Claude Opus 4.7 |
| **Status** | Active |
| **Type** | Process |
| **Created** | 2026-06-20 |
| **Post-History** | 2026-06-20 |

## Abstract

This ZEP defines what a ZEP is, what it is *not*, the lifecycle of a
proposal, and the responsibilities of the author and the reviewer.
ZEPs that govern syntax or stdlib behaviour have authority; the
compiler and stdlib must follow them.

## Motivation

ZyenLang is a small experimental language with one human author and a
rotating cast of AI assistants. Decisions about syntax and the standard
library are scattered across commit messages, READMEs, CLAUDE.md, and
chat transcripts. That works for the first few months but rots fast:
the next contributor (or the same author six months later) cannot tell
which decisions are load-bearing versus which were tactical.

ZEPs fix this by giving each long-lived decision a stable, dated, named
document. New contributors read the ZEP index instead of mining git
log. Future-you can disagree with a ZEP, but at least disagreement is
explicit instead of accidental.

## Specification

### Types

| Type | Use it for |
|---|---|
| **Process** | Governance, lifecycle, who decides what. |
| **Informational** | Conventions, style guides, recommended patterns. Compiler does not enforce. |
| **Standards** | A rule the compiler or stdlib must enforce. Violating it is a bug. |

A `Standards` ZEP is the strongest form. Once accepted as `Final`, the
compiler is bound by it. An `Informational` ZEP is a strong suggestion;
the compiler doesn't reject violations but the stdlib must follow them
internally.

### Status

```
Draft  →  Review  →  Active   (Process / Informational, ongoing)
                  →  Final    (Standards, locked)
                  →  Rejected (closed without merge)
                  →  Withdrawn (author pulled it back)

Any of the above (except Draft) →  Superseded  (replaced by a newer ZEP)
                                →  Deprecated  (still valid but no longer
                                                recommended)
```

- **Draft**: actively being written. Not authoritative yet.
- **Review**: author considers it complete and is gathering objections.
- **Active**: in force, may be revised in place if substance does not change.
- **Final**: in force, locked. Substantive changes require a new ZEP that
  Supersedes this one.
- **Rejected** / **Withdrawn**: kept in the repo for the record. Do not
  apply.

### Numbering

ZEPs are numbered sequentially starting at 1, zero-padded to four digits
in the filename: `ZEP-0001-purpose-and-process.md`.

Numbers, once assigned, are never reused or reassigned. A withdrawn
or rejected ZEP keeps its number.

### Authorship

Any author may submit a ZEP. The author may be `zuenchen` (the human
maintainer), an AI assistant by name (`Claude Opus 4.7`, `ChatGPT`), or
a future external contributor. Multiple authors are written
comma-separated.

The maintainer (`zuenchen`) is the final decision-maker on Status
transitions.

### Workflow

1. **Draft**: Copy `ZEP-0002-zep-template.md`, pick the next free
   number, write the proposal. Status: `Draft`.
2. **Review**: When you think it's done, change Status to `Review` and
   open a PR or post it for feedback.
3. **Decision**: Maintainer marks it `Active` / `Final` / `Rejected` /
   `Withdrawn`.
4. **Index**: Add the ZEP to the table in `README.md`.

### What is not a ZEP

A document does not need to be a ZEP if it is:

- A bug report (lives in compiler repo `docs/`).
- Per-module API reference (lives in `docs/std_<module>.md`).
- Release notes or changelogs (lives in `SPEC_v0_1.md`).
- A one-sentence rule (lives in the README).
- A short style preference (extend an existing Style Guide ZEP).

If you are unsure, draft it as a ZEP. The cost of an unnecessary ZEP is
low; the cost of an undocumented decision is high.

## Why a stripped-down process

PEP-style governance evolved for a multi-author language with thousands
of contributors. ZyenLang has one human and a few AI assistants. The
stripped-down rules here remove vote counting, BDFL delegation, and
PEP-Editor roles, leaving just:

- A canonical number (so we can cite "ZEP-0004" forever).
- A status (so readers know if the doc still applies).
- A template (so ZEPs are skimmable).

When the project grows, this ZEP itself can be Superseded by a fuller
governance document.

## References

- [PEP 1 — PEP Purpose and Guidelines](https://peps.python.org/pep-0001/)
  (the model this ZEP is shrunk from)

## Copyright

This document is placed in the public domain.
