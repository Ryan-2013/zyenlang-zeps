# ZEP-0002 — ZEP Template

**English** | [繁體中文](ZEP-0002-zep-template.zh-TW.md)

| Field | Value |
|---|---|
| **ZEP** | 0002 |
| **Title** | ZEP Template |
| **Author** | zuenchen, Claude Opus 4.7 |
| **Status** | Active |
| **Type** | Process |
| **Created** | 2026-06-20 |
| **Post-History** | 2026-06-20 |

## Abstract

A boilerplate to copy when writing a new ZEP. Keep the section order
below; drop sections that don't apply, but don't reshuffle.

## How to use this template

1. Copy this file.
2. Rename to `ZEP-NNNN-short-kebab-title.md`. Use the next free integer
   for `NNNN`, zero-padded to four digits.
3. Replace the header fields and section bodies.
4. Add an entry to `README.md`'s ZEP index table.

---

# ZEP-NNNN — Short Title

| Field | Value |
|---|---|
| **ZEP** | NNNN |
| **Title** | Short Title |
| **Author** | Your Name(s) |
| **Status** | Draft |
| **Type** | Standards / Informational / Process |
| **Created** | YYYY-MM-DD |
| **Post-History** | YYYY-MM-DD |
| **Supersedes** | (optional) ZEP-MMMM |
| **Superseded-By** | (optional, set later) ZEP-OOOO |

## Abstract

One paragraph. What this ZEP decides. A reader should be able to
skim the abstract and know whether the rest applies to them.

## Motivation

Why does this need a ZEP? What problem are you solving? What goes
wrong if we don't have this rule? Cite concrete cases (a bug, a
confusing API, a contributor onboarding pain point).

## Specification

The actual rules. Be concrete. Use code samples where they clarify.
Headings encouraged when the spec covers multiple sub-rules.

If the ZEP is `Standards`, every rule must be either enforced by the
compiler or implementable as a lint. State which.

## Rationale

For each non-obvious decision in the Specification, say why this option
was picked over the alternatives. This is the section future-you (or
the next ZEP that wants to Supersede this one) reads first.

## Backwards Compatibility

What existing code breaks? If anything: how to migrate. If nothing:
say so explicitly so reviewers don't have to guess.

## Reference Implementation

Link to the commit / file where this ZEP is implemented. For
`Informational` ZEPs that just codify existing practice, link to a
representative example in the stdlib.

## Open Questions

Things this ZEP intentionally does *not* answer. Useful for future
contributors deciding whether to write a follow-up ZEP.

## References

External links: prior art (other languages), discussions, bug reports.

## Copyright

This document is placed in the public domain.
