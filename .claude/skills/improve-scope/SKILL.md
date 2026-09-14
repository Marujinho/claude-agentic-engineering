---
description: Turn a raw task file into a complete, implementation-ready scope
argument-hint: <path/to/task.md>
---

# Scope

**Input**: $ARGUMENTS (a task file, or a document already in the conversation)

Fill in everything an implementing agent would otherwise have to guess. Answer *what*, *why*, *for whom*, *under which rules* — not *how*. That's `/plan`'s job.

Runs before `/plan`.

## Process

1. **Read the task file.** Separate what it states from what it assumes.

2. **Look at the codebase.** `ls`, `git log --oneline -10`, `CLAUDE.md` / `README.md`, plus the closest existing feature to the one being asked for. Pull out: stack and versions, where this kind of code lives, naming conventions, error handling, test patterns — each with a `file:line` source.

   Empty repo? Then propose those conventions instead. One concrete choice per decision, not a menu.

   **Subagents**: if the recon splits into independent threads — conventions, data layer, tests, external dependencies — dispatch them in parallel and have each return findings with `file:line` refs. Skip this on a small or familiar codebase; one agent reading four files directly beats four agents reporting back.

   **No git repo?** `git init` and commit the current state as a baseline before anything gets written — otherwise there's no diff later showing what the implementation touched. Add a `.gitignore` first if there isn't one, so the baseline commit doesn't swallow `node_modules`, `vendor`, build output, or `.env`. Note in the scope that the repo was initialized.

3. **Find the gaps.** Go through the scope template below and mark each item known, inferred, or missing. Never ask about something the codebase already answers.

4. **Run the security pass.** Walk all ten OWASP 2025 categories against what's being built — one at a time, no skipping. For each: does this feature touch it? If yes, what's the requirement? The output is a requirement the implementer must satisfy and a test that proves it, not a note saying "be careful with auth". Categories the feature genuinely doesn't touch get marked N/A with a reason.

   Check the code around it too, not just the new code — a feature can be written correctly and still break an existing control by widening a query, reusing a permissive handler, or logging a field that wasn't logged before.

5. **Ask, once.** Batch every question whose wrong answer means rework. Give 2–3 options with a recommendation. Everything else becomes a listed assumption — never a silent one.

6. **Write the scope** next to the task file, named for the feature itself in kebab-case — `user-invite-flow.scope.md`, not `task.scope.md` or `notes-final-v2.scope.md`. The name should tell someone what's inside without opening it.

## Output

```markdown
# Scope: {Name}

**Source**: `{task file}` · **Type**: {feature/bugfix/refactor} · **Status**: {READY / BLOCKED}

## Problem
{What's broken or missing today, and what it costs.}

## User
{Who they are, and what they can do after this ships that they can't now.}

## Definition of Done
- [ ] {Observable outcome someone else can verify}

## Out of Scope
- {Thing} — {why not / where it belongs}

## Stack & Areas Affected
{Tech + versions. Which parts of the codebase this touches, directly and indirectly.}

## Conventions
{Naming, structure, error handling, tests — each with `file:line`, or marked PROPOSED for greenfield. Include a real snippet where it's clearer than prose.}

## Data & Contracts
{Entities, schema changes, API/event shapes, migrations, backwards-compat. Or "None".}

## Constraints
{Architecture rules that must hold. Performance, scale, accessibility — whichever apply. Security goes in its own section below.}

## Security (OWASP Top 10:2025)

| # | Category | Applies? | Requirement | How it's tested |
|---|---|---|---|---|
| A01 | Broken Access Control | | | |
| A02 | Security Misconfiguration | | | |
| A03 | Software Supply Chain Failures | | | |
| A04 | Cryptographic Failures | | | |
| A05 | Injection | | | |
| A06 | Insecure Design | | | |
| A07 | Authentication Failures | | | |
| A08 | Software or Data Integrity Failures | | | |
| A09 | Security Logging and Alerting Failures | | | |
| A10 | Mishandling of Exceptional Conditions | | | |

**Existing controls at risk**: {controls this change could weaken, or "None identified"}
**Secrets / PII touched**: {what data, where it flows, how it's stored — or "None"}

## Do / Don't
- Do: {instruction}
- Don't: {instruction} — {consequence}

## Caveats
{Sharp edges in the affected code, external dependencies, failure modes.}

## Assumptions
| Assumption | Basis | If wrong |
|---|---|---|

## Open Questions
{Blocking, unanswered — or "None".}

## Tests
{What must be covered: happy path, error cases, boundaries, and every "How it's tested" cell from the security table. Note anything that needs a fixture, seed, or mock that doesn't exist yet.}

## Verification
{Build, lint, test commands. Plus end-to-end checks precise enough to execute.}

## Delivery
- **Branch**: {new branch + name, or existing branch + which}
- **Migrations / ordering**: {anything that must run before or after deploy}
- **Config / secrets**: {new env vars, where they're set, who sets them}
- **Feature flag**: {name and default, or "none"}
- **Rollout**: {all at once, staged, behind a flag}
- **Rollback**: {how to undo, and whether data changes make it one-way}
```

## Rules

- Explaining *how* to build it means you've drifted into `/plan`.
- A convention without a `file:line` is either unsourced or PROPOSED. Say which.
- Anything from your head, not the task file, goes in the assumptions table.
- Every OWASP row gets an answer. "Applies: no" is an answer; a blank cell is not.
- A security requirement with no test in the same row is incomplete.
- Open questions remaining means Status is BLOCKED. Don't hand it to `/plan`.
- If the task is really several tasks, or the codebase contradicts it, say so before writing.