# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project state

Devsweep is a macOS disk-cleaner CLI for developers, focused on artifacts left behind by AI
agents (Claude Code sessions, Cursor workspace storage, forgotten git worktrees, MCP data)
and classic dev caches (Xcode, Gradle, node_modules, etc.).

**This repository is pre-implementation for v0.1.** The tree currently contains only
`LICENSE`, `.gitignore`, `.github/ISSUE_TEMPLATE/`, and `docs/`. `Package.swift`, `Sources/`,
and `Tests/` do not exist yet. The full work is tracked as **91 GitHub issues on Project #3**:
https://github.com/orgs/androidbroadcast/projects/3. Start there.

Everything important — architecture, safety invariants, performance targets, roadmap — is
in `docs/`. Read those before planning any change.

## Primary references

- **[`docs/architecture.md`](docs/architecture.md)** — package layout, `CleanupModule`
  contract, scan/clean/restore lifecycle, FastWalker, deletion-history storage.
- **[`docs/safety.md`](docs/safety.md)** — invariants **S1–S14**. Every issue that touches
  a destructive path references these; your PR must hold them.
- **[`docs/performance.md`](docs/performance.md)** — invariants **P1–P10**. Runtime targets
  (e.g. FastWalker < 5 s for 100k files) and the profiling commands to verify them.
- **[`docs/roadmap.md`](docs/roadmap.md)** — what ships in 0.1, 0.2, 0.3, 1.0, and later.
  When scope feels fuzzy, consult this first.
- **[`docs/accepted-risks.md`](docs/accepted-risks.md)** — trade-offs we have consciously
  accepted; do not "fix" them without discussion.
- **[`docs/adr/`](docs/adr/)** — records of significant decisions (dependency choices,
  benchmark framework, etc.).

## Architecture in one paragraph

Devsweep ships as a single SwiftPM package with four targets:
`DevsweepCore` (public Plugin API: protocols, value types, marker enums — no destructive
APIs), `DevsweepCorePrivate` (destructive implementations: `Remover`, `TrashService`,
`ArchiveService`, `FastWalkerImpl`, `CommandRunnerImpl`, `FileSystemImpl`,
`ResourceLockServiceImpl`, `RemovalLogger`), `DevsweepModules` (bundled cleanup modules —
`xcode.deriveddata`, `ai.claude.sessions`, `git.worktrees`, `js.node_modules-lite` in
0.1), and `devsweep` (CLI executable and composition root). Dependency direction is one-way
top-down (`devsweep → Modules → Core`, `devsweep → CorePrivate → Core`); a plugin can only
reach into `DevsweepCore`. CI enforces this via import-graph analysis.

## Non-negotiable rules for this project

These are hard architectural invariants, not style preferences:

- **Modules never delete anything directly.** `DevsweepModules` files must not reference
  `FileManager.removeItem`, `FileManager.trashItem`, `Foundation.Process`,
  `Darwin.posix_spawn`, or import `DevsweepCorePrivate`. A module’s `plan()` returns
  `[CleanAction]`; the `Remover` actor executes. (S1, S8)
- **External CLI goes through `CommandRunner` only**, never direct `Process` or
  `/bin/sh -c …`. `CommandRunner` enforces an allowlist of binary names.
- **Filesystem work goes through the injected `FileSystem` protocol**, not `FileManager`
  directly, so modules are testable in isolation and safe-by-construction.
- **Safety over convenience.** Dry-run is the default for `clean`; `--apply` is required
  for real action; `GlobalDenyList` is a hardcoded last-line-of-defence checked by
  `Remover` on every item.
- **No new dependencies without explicit user approval** (applies repo-wide, but doubly so
  here because of the safety perimeter). Propose via an ADR in `docs/adr/` and wait for
  approval before touching `Package.swift`.
- **`swarm-report/` is gitignored and stays that way.** It holds the internal research
  trace (research report, Q&A state files, issue drafts). Don’t commit it and don’t rely
  on it being present for another contributor.

## Workflow

All implementation work is driven by GitHub issues on Project #3. Each task issue includes:

- A **Context** paragraph explaining why the task exists.
- **Acceptance criteria** in Given/When/Then form, each referencing an `AC-NN` ID from the
  project’s internal design record. Honour every AC — they are the test spec.
- **Implementation notes** with the relevant `S`/`P` invariants and pointers to the docs.
- **Dependencies** (`Blocked by #N`).
- A **Definition of Done** checklist.

Typical sequence for a task:

1. `gh issue view <N>` to read the full spec.
2. Start from the default branch (main). User-global rules require a feature/worktree
   branch for anything that will commit — `feature/<short-description>` or
   `fix/<short-description>`.
3. Implement against the AC. Keep the safety checklist from the issue in view.
4. `swift build && swift test` (once the Swift package exists — see below).
5. Open a PR referencing the issue (`Closes #N`).

### Discovery commands for the project board

```sh
gh issue view <N>                                                 # read an issue
gh project item-list 3 --owner androidbroadcast --limit 100      # see the whole board
gh issue list --label "type:task" --state open --milestone v0.1  # all open v0.1 tasks
```

Custom project fields you’ll interact with: `Size` (S/M/L/XL — drives model selection),
`Module` (core/modules/cli/gui-app/tray-app/infra/docs/tests).

### Build / test / lint

`Package.swift` does not exist yet; it lands in issue **#19 (T-01-1)**. Once that task
ships, the expected commands are:

```sh
swift build --configuration debug       # fast local build
swift build --configuration release     # what the release workflow signs/notarizes
swift test                              # full test suite
swift test --filter <TestClassName>     # single test class
swift test --filter <TestName>/<case>   # single test method (Swift 5.10+)
```

Linting is decided in issue **#34 (T-02-2)** — once wired up, the command will be either
`swift-format lint -r Sources Tests` or `swiftlint` per the ADR.

CI (workflow added in issue **#32 (T-02-1)**) runs `swift build && swift test` + lint +
import-graph analysis on every PR.

### First implementable tasks (unblocked, pick these first)

`T-01-1` (Package.swift), `T-03-1..4` (README / CONTRIBUTING / SECURITY / architecture),
`T-04-1` (notarization), `T-04-4` (CHANGELOG), `T-05-1..4` (Plugin API protocols and value
types), `T-07-3` (GlobalDenyList constant), `T-07-4` (displaySafe sanitiser), `T-09-1`
(FastWalker protocol + default exclusions), `T-12-1` (zstd dep spike), `T-18-1` (benchmark
framework spike). After these, the rest of the graph unblocks naturally.

## Minimum context before any change

Before writing or modifying code that lives under `Sources/`:

1. Read `docs/architecture.md` for the target you are touching.
2. If the change is destructive-adjacent (touches FS, spawns processes, writes logs,
   handles archives), read `docs/safety.md` end-to-end.
3. If the change affects scan/walk/archive/log throughput, read `docs/performance.md`.
4. Read the issue body — `Files to touch` is advisory, but `Acceptance criteria` and
   `Implementation notes` are authoritative.

Missing this context will usually produce code that violates an S-invariant and is rejected
by review even if it builds.

## Session operating notes

- `AC-NN` references in issues map to acceptance criteria in
  `swarm-report/mac-dev-cleaner-research.md` — gitignored, local-only. If a session cannot
  find the file and needs to resolve a specific ID, ask the user; the ID glossary is not
  reproduced in `docs/`.
- Direct fast-forward merges (`git merge --ff-only <branch> && git push origin main`) are
  the norm for docs/infra work at this stage — no PR ceremony until Swift code starts
  landing. The branch-guard hook blocks commits on `main`; always branch first
  (`git checkout -b <feature-branch>`).
- `gh project field-list 3 --owner androidbroadcast --format json` — fetch Size/Module
  field and option IDs when scripting project-board updates.
- `gh api graphql` with the `addSubIssue(issueId, subIssueId)` mutation — native
  parent/child linkage between issues (distinct from the Project "Parent issue" field).
- For bulk issue or project-board edits, use a short Python script calling `gh` CLI +
  GraphQL and checkpoint state to `/tmp/*.json` so the operation is resumable.
