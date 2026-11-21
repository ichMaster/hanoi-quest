# Specification format: VISION, ARCHITECTURE, ROADMAP

The format of the three specification files, taken from the matrix-agora project. `examples/` holds its first
version (commit 23f9a5a), copied verbatim; this file names the rules those examples follow, plus the
conventions matrix-agora added later (marked "later"). Both `/generate-spec` and `/plan-phase` write to this
format.

## The three files

All three live in `specification/`, in English.

| File | Answers | Holds | Never holds |
|---|---|---|---|
| `VISION.md` | why, and for whom | purpose, audience, direction, principles, non-goals, glossary | mechanisms, file formats, schedules |
| `ARCHITECTURE.md` | how it works | components, mechanisms and their invariants, contracts, configuration, security, errors, layout, testing | the order of work, phase plans |
| `ROADMAP.md` | in what order | versions and phases, each with Goal, Tasks, DoD and Tests | re-explained mechanisms (it refers to ARCHITECTURE §Section) |

Instructions addressed to Claude Code about *how to work* (when to stop, how to commit, which commands to run)
belong in `CLAUDE.md`, not in these files.

## Rules for all three files

- **Title line:** `# Vision — <project>`, `# Architecture — <project>`, `# Roadmap — <project>`.
- **Concrete names.** File paths, settings, env vars, commands, class and method names in backticks; numbers
  with units and example defaults (`HISTORY_N` (e.g. 30), `0.15 m`).
- **One home per fact.** A mechanism is described once, in ARCHITECTURE; ROADMAP points to it
  (`ARCHITECTURE §Message flow`) and repeats only the values a DoD needs. A non-goal lives in VISION.
- **Phase tags.** Where a part appears later than the start, ARCHITECTURE tags it: `(from v2.1)`,
  `**Panel (from v3.2):**`, a `Phase` column in the configuration table.
- **Literal strings in another language** (UI text, prompts) are quoted as they are, followed by an English
  gloss in parentheses: «Котра година?» ("What time is it?").
- **The owner** is the person the project is built for and who confirms manual checks. Refer to them as "the
  owner"; no pronouns that assume gender.
- **Plain sentences, no emoji.** matrix-agora later used emoji as status marks; this project forbids emoji in
  documentation, so statuses are words (`planned`, `completed`).
- **No secrets and no private data**, since the specification is committed.

## VISION.md

About 50–80 lines. Sections in this order:

1. `## In one sentence` — one sentence: what it is, where it runs, for whom.
2. `## What we are building` — one paragraph: the shape of the system and the logic of the build order
   (what is proven first, what complexity comes later).
3. `## For whom` — who uses it, how access is limited, and what it is not.
4. `## The longer-term direction` *(optional)* — what may follow, and the statement that it is not part of
   this roadmap. Later matrix-agora renamed it to fit the direction ("The direction: the panel is the
   product").
5. `## Principles` — 6–12 bullets, each `- **A short rule.** One to three sentences: what it means in
   practice and why.` A principle is a decision that constrains the design (for example "Closed by
   default", "Rules live in code, identity lives in the canon"), not a slogan.
6. `## Non-goals` — bullets of what is deliberately left out. This list is the scope fence that issue
   generation and reviews read, so each item is specific enough to reject a task against.
7. `## Glossary` — `- **Term** — definition, with the concrete name used in code or files.` Every term that
   ARCHITECTURE and ROADMAP rely on.

## ARCHITECTURE.md

About 150–300 lines at the start; it grows with every phase. Sections in this order (omit what the project
does not have):

1. `## Overview` — one paragraph naming the axes along which the system grows, then an ASCII diagram in a
   code block: the runtime components, where they run, and how they connect.
2. `## Components` — bullets `- **Name:** what it is, the technology and version, where it runs, from which
   phase.`
3. **One section per mechanism** (`## Message flow and the allowlist`, `## Memory`, …). Each states its
   **invariants** as a numbered list ("The invariants every reply path must honor:"), the rules with their
   defaults, the failure behavior, and which decisions are kept pure so tests need no framework objects.
   Later matrix-agora added `## Network, ports and protocols`, `## Tech stack`, `## Data and state files`,
   `## Deployment and CI/CD` and `## Observability` sections when the project grew them.
4. `## Contracts` — starts with "Changing any of these updates this document and the test that pins it, in
   the same commit:" and lists the seams other parts or later phases rely on: names of settings, file
   formats, public interfaces, ordering rules, protocol semantics.
5. `## Configuration and secrets` — a table `| Variable | Phase | Meaning (example default) |`, then where
   secrets live and where they never appear (logs, argv, commits).
6. `## Security and access` — a table `| Threat | Protection |`, then any trade-off accepted on purpose and
   what must change before it stops being acceptable.
7. `## Error handling and resilience` — bullets: each failure and what the system does (log, stay silent,
   retry, keep the previous state); the rule that one failure never takes the process down.
8. `## Stack and repository layout` (later split into `## Tech stack` and `## Repository layout`) — the
   repository tree in a code block, with a comment per entry and the phase in which it appears.
9. `## Testing and CI` — what the automated gates need (no network, mocks, injected clock), a table
   `| Gate | Command | When |`, the rule that a gate that cannot run yet is `n/a`, not passed; then
   **Unit tests** (what pure logic is covered), **Contract tests** (they pin §Contracts), and
   **Manual (owner) checks** (what needs the live system, and that they count as passed only once the owner
   confirms).

## ROADMAP.md

The header, before the first version:

1. One paragraph: "N self-contained versions, built in order: **v0** Name (what) → **v1** Name (what) → …".
   Then: phases inside a version are numbered `vA.B`; each phase lists a **Goal**, a short description,
   **Tasks**, a **Definition of Done (DoD)** and the **Tests** that encode the DoD (see ARCHITECTURE.md
   §Testing and CI); phases are built strictly in file order, each on the previous one's real, released code.
2. **Versioning (`A.B.C`).** `A` = roadmap version, `B` = phase within it, `C` = a post-release fix on that
   phase. Phase `vA.B` → release `A.B.0`, tag `vA.B.0`; a fix bumps `C`. Releases are cut per phase. Never
   bump a version without explicit confirmation.
3. One paragraph on **Manual (owner)** checks: which kinds of DoD items need the owner, and what the automated
   tests mock instead.
4. `---`
5. `## Status` *(later)* — one paragraph on where the project stands, then a table
   `| Phase | Title | Status | Release |` with one row per phase. Status is `planned`, `in progress` or
   `completed`; Release is `` `vA.B.0` (YYYY-MM-DD) `` plus any patches, or `--`.

Each version:

```
## vA — Name: what it delivers

One paragraph: what the version achieves as a whole and why it comes at this point. Depends on: vA-1 (or
"nothing — this is the foundation").
```

Each phase, in this field order:

```
### vA.B — Title

**Status:** planned                     (later; "completed — released `vA.B.0` on YYYY-MM-DD." once released)

**Goal:** one sentence: the outcome the owner can see.

One to three sentences: what the phase builds, pointing to ARCHITECTURE §Section for the mechanism, and how
it is run.

**Owner decisions** (YYYY-MM-DD): decisions the owner made for this phase.          (optional, later)

**Tasks:**
- concrete work items naming files, settings and commands; a code block when an exact config or call is the
  deliverable.

**DoD:**
- (Manual, owner) a check that needs the owner or the live system.
- an automated check.

**Tests:** unit — what the tests cover, what is mocked; which gates apply. Or "none (owner steps only)".

**Owner steps:** what the owner does outside the repo.                              (optional, later)

**Not in this phase:** nearby work that belongs to a later phase or to no phase.    (optional, later)
```

- When every DoD item is manual, write `**DoD** (Manual, owner):` and drop the per-item prefix.
- **A phase** is one releasable slice that can be tested on its own, usually 3–8 tasks. An owner-only phase
  (setup steps with no code) is fine.
- **A version** is 1–7 phases toward one goal. The first version proves the setup end to end with the least
  logic; complexity is added by version, never all at once.
- **Order** is dependency order. Never plan a phase whose work needs a later phase.
