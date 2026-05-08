# ComplyTime Enhancement Proposals (CEPs)

## Purpose

CEPs capture architectural decisions and requirements for significant changes to the ComplyTime project. They are the unit of design review for work that crosses component boundaries, introduces new subsystems, or changes core interfaces.

CEPs are **not** feature specs. They define the problem, constraints, architectural approach, and acceptance criteria at a level that enables independent feature implementation.

## When to Write a CEP

Write a CEP when a change:

- Introduces a new subsystem or component
- Changes a public interface or plugin contract
- Adds a new dependency path (OCI, Wasm runtime, etc.)
- Requires coordination across multiple repositories
- Has meaningful trade-offs that need community input

Do **not** write a CEP for:

- Bug fixes or minor enhancements
- Documentation changes
- Changes confined to a single internal package
- Component-level features that don't alter cross-component interfaces or introduce new subsystems

## Lifecycle

| **Status** | **Meaning** |
|:---|:---|
| `draft` | Under active authorship; not yet ready for review |
| `proposed` | Submitted as PR for community review |
| `accepted` | Approved by maintainers; implementation may begin |
| `implemented` | Core work complete and merged |
| `withdrawn` | Author withdrew; superseded or abandoned |
| `rejected` | Maintainers declined after review |

## Process

1. **Author** copies `cep-template.md` to `cep-NNNN-short-title.md`
2. **Author** fills in all required sections, opens PR against `main`
3. **Community** reviews via PR comments (minimum two maintainer approvals)
4. **Maintainers** merge with status `accepted` or close with `rejected`
5. **Implementers** reference the CEP in feature PRs (`Implements: CEP-NNNN`)
6. **Author** updates status to `implemented` when core work merges

## Numbering

CEPs use four-digit sequential numbers starting at `0001`. The filename encodes the number and a short kebab-case title: `cep-0001-complypack-architecture.md`.

## Review Criteria

Reviewers evaluate CEPs against:

- **Problem clarity** — Is the problem well-defined with evidence?
- **Scope** — Is the boundary between in-scope and out-of-scope explicit?
- **Alternatives** — Were alternatives considered and rejected with rationale?
- **Compatibility** — Are backward-compatibility and migration addressed?
- **Testability** — Can acceptance criteria be verified?

## Index

| **CEP**                                     | **Title**               | **Status** | **Author(s)**               |
|:--------------------------------------------|:------------------------|:-----------|:----------------------------|
| [0001](cep-0001-complypack-architecture.md) | ComplyPack Architecture | proposed | @jpower432, @trevor-vaughan |
