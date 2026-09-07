# COMMIT.md — Commit Message Conventions

This document follows the [Conventional Commits](https://www.conventionalcommits.org/) specification — a structured, machine- and human-readable commit format used widely across the industry. It applies to any project in this family, regardless of language or framework.

## Full structure

```text
<type>[(<scope>)][!]: <subject>

[optional body]

[optional footer(s)]
```

- `type` is one of the categories below, lowercase.
- `scope` is optional, in parentheses, no space before it.
- `!` immediately before the colon marks a breaking change (see below) — optional.
- One blank line between subject and body, and another blank line between body and footer(s), when those sections are present.

Default to a subject-line-only commit — no body, no footer. Mark a breaking change with the `!` marker on the subject line rather than a `BREAKING CHANGE:` footer. Only add a body or footer when explicitly asked to.

## Types

| Type | Use for |
|---|---|
| `feat` | A new capability, page, field, button, or behavior |
| `fix` | Correcting incorrect behavior (a bug, a wrong condition, a missing case) |
| `refactor` | Restructuring existing code with no behavior change (renames, extracting a component, removing dead code) |
| `style` | Visual/formatting-only changes — spacing, colors, font sizes, code formatting — with no logic change |
| `chore` | Tooling, config, or non-product-code housekeeping |
| `docs` | Documentation only (guideline docs, `README`, code comments) |
| `test` | Adding or correcting tests only |
| `perf` | A change whose entire point is performance, not new behavior |
| `build` | Build tooling/dependencies (bundler config, build scripts, package manager files) |
| `ci` | CI/CD pipeline configuration |
| `revert` | Reverting a previous commit, when not just using Git's own revert format below |

Pick the type that actually matches the change — don't force a test-only commit into `chore` or a performance-only commit into `refactor` just because those feel more familiar.

A revert keeps Git's own default format instead of the table above: `Revert "<original subject>"`, body naming the reverted commit. Don't rewrite it into a `type:` subject.

## Scope

Use a scope when the change is centered on one identifiable place; omit it when the change spans many unrelated files.

- **One component/module carries the change**: scope is that file's name, matching its own naming convention exactly — `feat(UserProfileForm): add avatar upload`, `fix(PaymentSummaryCard): correct total calculation`.
- **The change is about a layer/folder rather than one file**: scope is that layer's name — `fix(enums): remove duplicate value`, `feat(ci): cache dependencies between jobs`, `fix(i18n): correct pluralization for count`. Match the case already established for that particular scope name if it has prior commits; don't introduce a second casing for the same scope.

## Subject line

- Lowercase immediately after the colon.
- Imperative mood — "add", "fix", "remove", "implement" — never "added"/"adds"/"fixed".
- No trailing period.
- Keep it short enough to read on one line in `git log --oneline` — aim for ~50 characters, treat ~72 as a hard ceiling. If the summary needs more than that, the extra detail belongs in the body, not a run-on subject.
- The subject alone must be a complete summary; don't write a vague subject and rely on the body to say what actually changed.

## Body

Optional — many commits are a single subject line with no body. Add a body when the subject can't carry enough context on its own (a multi-file feature, a non-obvious "why"): one short paragraph, wrapped at ~72 characters per line, explaining what changed and why, not restating the subject in more words or narrating how (the diff already shows how).

```text
feat: add saved-search alerts to the search page

Lets a signed-in user save a search and opt into email alerts when new
matching results appear, with a management list under account settings.
```

## Breaking changes

Flag a commit that breaks an existing contract other code relies on — a shared function's signature, a store's public getters/actions, an API method's params/return shape, anything another file already imports and calls. Mark it either way:

- `!` right after the type/scope: `feat(SearchForm)!: rename validate() to validateAndSubmit()`
- a `BREAKING CHANGE:` footer (see below), which can also carry the migration detail the one-line subject can't

The whole point of marking it is that a reviewer (or a future agent) can spot an incompatible change without reading the full diff; silently shipping one as a plain `feat:`/`fix:` defeats that.

## Footers

Optional, after the body, separated from it by a blank line. One footer per line, `Token: value` (or `Token #value`), the same shape as a Git trailer:

```text
fix(SearchResultsTable): correct pagination offset

BREAKING CHANGE: `SearchResultsTable` no longer accepts a `pageSize` prop;
read it from the shared pagination store instead.
Refs: #482
```

- `BREAKING CHANGE:` (this exact casing) is the one the spec singles out — at most one per commit.
- Other footers (`Refs:`, `Closes:`, `Co-authored-by:`) follow the same one-per-line trailer shape when there's an external reference worth recording; don't invent a footer for information that just belongs in the body.

## Anti-patterns to flag, not silently fix

- A branch name committed as the subject line (`Feature/username/some-task`) — not a `type:` prefix, not lowercase, not a real description of the change.
- A single commit bundling several unrelated fixes/features joined by "and" because they happened to land together — prefer one commit per logical change.
- `feature:`/`Feature:` as a type — the correct type is `feat`.
- A commit that changes a shared function/component's public shape without a `!` or `BREAKING CHANGE:` footer — the incompatibility should be visible from the log, not discovered later by whoever's code broke.
