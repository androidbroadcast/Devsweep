# Architecture

This document describes how Devsweep is structured — package layout, the Plugin API contract,
core services, lifecycle, and concurrency model. It is the reference for contributors and
third-party plugin authors.

For the _philosophy_ behind why these decisions were made, see the individual sections below;
for the _invariants_ a piece of code must hold, see [`safety.md`](safety.md); for runtime
targets, see [`performance.md`](performance.md).

---

## Package layout

Devsweep ships as a single SwiftPM package with four targets:

```
devsweep/
├── Package.swift
└── Sources/
    ├── DevsweepCore/           # Public API — protocols, value types
    ├── DevsweepCorePrivate/    # Destructive implementations, IO services
    ├── DevsweepModules/        # Bundled cleanup modules
    └── devsweep/               # CLI executable and composition root
```

Dependency direction (enforced by `Package.swift` and CI):

```
devsweep (CLI)
 ├─► DevsweepModules    (depends only on Core)
 ├─► DevsweepCorePrivate (depends only on Core)
 └─► DevsweepCore        (no internal deps)

Third-party SPM package:
 └─► DevsweepCore        (only — never CorePrivate)
```

### Why four targets?

- `DevsweepCore` is the **public contract**: protocols, value types, marker enums. No
  destructive APIs reachable from here. Third-party plugin authors depend on this target
  only.
- `DevsweepCorePrivate` holds the **destructive implementations**: `Remover`, `TrashService`,
  `ArchiveService`, `FileSystemImpl`, `CommandRunnerImpl`, `FastWalkerImpl`,
  `ResourceLockServiceImpl`. These reference `Foundation.Process`, `Darwin.posix_spawn`,
  `FileManager.removeItem`, `FileManager.trashItem` — APIs that must not leak into plugin code.
- `DevsweepModules` hosts the **bundled modules** (`xcode.deriveddata`, `ai.claude.sessions`,
  `git.worktrees`, `js.node_modules-lite`, and more in later versions). These import
  `DevsweepCore` only.
- `devsweep` is the **composition root**: the only place where public and private targets are
  wired together, via `internal import DevsweepCorePrivate` (Swift 5.9+, [SE-0409]).

CI enforces the boundary: an import-graph analysis job fails any PR where `DevsweepModules`
references destructive APIs or imports `DevsweepCorePrivate`.

[SE-0409]: https://github.com/apple/swift-evolution/blob/main/proposals/0409-access-level-on-imports.md

---

## The Plugin API

A cleanup module is any type conforming to `CleanupModule`. The protocol is small; most of
the mechanics live in injected context objects.

```swift
public protocol CleanupModule: Sendable {
    var id: String { get }                    // "xcode.deriveddata"
    var category: Category { get }
    var displayName: String { get }
    var description: String { get }
    var riskLevel: RiskLevel { get }          // .safe / .reversible / .irreversible
    var requiredQuiescence: [ProcessPredicate] { get }

    func scan(context: ScanContext) -> AsyncThrowingStream<[ScanItem], Error>
    func shouldConsider(_ item: ScanItem, context: ScanContext) -> Bool
    func plan(
        items: [ScanItem],
        mode: CleanMode,
        context: CleanContext
    ) async throws -> [CleanAction]
}
```

### Key principles

1. **Modules do not delete files.** A module’s `plan()` returns `[CleanAction]` describing
   _what_ should happen. A single `Remover` service executes the actions, applies safety
   checks, and writes the log. See [`safety.md` §S1](safety.md#s1--single-destructive-path).

2. **Streaming scan.** `scan()` yields batches (100–500 items or ~256 KB, whichever comes
   first). Per-item `yield` would add tens of thousands of `await` suspensions on large
   filesystems; batching keeps overhead negligible.

3. **Sum-type outputs.** `ScanItem` covers `.file / .directory / .externalResource /
   .archivable`; `CleanAction` covers `.trash / .archive / .externalCommand / .hint`. This
   lets modules with different shapes (folder-scan, external-CLI-driven, hint-only) share
   one protocol without leaky abstractions.

4. **Extensible categories.** `Category` is a struct newtype (`id`, `displayName`, `order`),
   not a closed enum. Standard categories live in `Category.build`, `.ai`, `.git`, etc.;
   third parties register new categories through `ModuleRegistry.registerCategory`, which
   rejects collisions (same `id` with different `displayName`/`order` is an error).

5. **Cross-cutting concerns injected via context.** `ScanContext` and `CleanContext` carry
   `logger`, `progress`, `config`, `fileSystem`, `commandRunner`, `resourceLock`, `clock`,
   `projectRoots`, `fastWalker`. Modules never touch `Foundation` directly for IO or
   subprocess — they go through these protocols. That gives tested-in-isolation modules and
   deterministic unit tests.

6. **Compile-in plugins.** Devsweep has no runtime plugin loader (no `dyld`, no `.dylib`
   scanning). A third party publishes their own SwiftPM package depending on `DevsweepCore`,
   forks/re-builds the devsweep binary with their module listed, and distributes the
   resulting binary themselves. A future executable-over-stdio protocol may be considered
   post-1.0, but dylib loading is explicitly out of scope.

7. **Concurrency boundary.** Core services (`ModuleRegistry`, `RemovalLogger`,
   `ResourceLockService`, `TrashService`, `ArchiveService`, `Remover`) are either `actor`-
   isolated or `Sendable` with internal synchronisation. Exactly one instance of each is
   created per CLI invocation at the composition root and injected through the context.

---

## Scan / Clean / Restore lifecycle

```
user invokes `devsweep clean --module <id> --apply`
        │
        ▼
  composition root builds contexts and instantiates Remover, RemovalLogger, ArchiveService
        │
        ▼
  module.scan(context) → AsyncThrowingStream<[ScanItem]>
        │
        ▼
  filter via module.shouldConsider(_:context:) + GlobalDenyList
        │
        ▼
  module.plan(items, mode, context) → [CleanAction]
        │
        ▼
  Remover.execute(actions, mode, denyList, logger)
        │                                  │
        ▼                                  ▼
  For each action:                      RemovalLogger (actor):
   1. TOCTOU re-validate                  append JSONL line
   2. deny-list check                    (batched fsync)
   3. resource-lock check                update session manifest
   4. dispatch:                          (.partial → .json on finalize)
      .trash       → TrashService
      .archive     → ArchiveService
      .externalCmd → CommandRunner
      .hint        → user-facing hint only
```

On start-up, any orphan `sessions/*.json.partial` is reconciled to `status=aborted` and
counts are recomputed from the JSONL (the authoritative record).

`restore --session <id>` reads the manifest + JSONL directly (never through any cache), then
applies an inverse action where possible (move back from `~/.Trash/`, extract from archive).
For `.externalCommand` actions the module supplies a hint only — restoration is the user’s
call.

---

## FastWalker

`FastWalker` walks project directories efficiently using macOS native bulk APIs.

- Backed by `open(O_DIRECTORY)` + `getattrlistbulk(2)` — 3–7× faster than
  `FileManager.enumerator` on SSD.
- Parallelism across `projectRoots` via `withTaskGroup` with bounded concurrency 4–8 (higher
  values cause APFS copy-on-write lock contention).
- `ProjectMarker` is a sum-type (`.file`, `.directory`, `.glob`, `.predicate`). The
  `.predicate(id:_:)` case accepts a `@Sendable` closure for non-trivial project-detection
  rules (Python’s any-of-{pyproject.toml, setup.py, requirements.txt}, Elixir’s
  mix.exs + _build sibling, etc.).
- Exclusions split into `excludedBasenames: Set<String>` (exact match) and
  `excludedPathPrefixes: [URL]` (startsWith after canonicalisation). Defaults are published
  as composable constants; modules can take `defaults.subtracting([...])` for their own
  scans.

Performance target: **100 000 files in under 5 seconds** on an Apple Silicon SSD. See
[`performance.md` §P1](performance.md#p1--fastwalker).

---

## Deletion history

Two-layer storage in `~/.local/state/devsweep/`:

- **`removed.jsonl`** — append-only, one JSON object per item action. **Source of truth.**
- **`sessions/<id>.json`** — session manifest, _derived_ from JSONL. Lifecycle:
  `.json.partial` → atomic rename → `.json` on finalize.

Why this layering: JSONL is grep-able, rotates cleanly on month boundary, and — since it is
append-only — never left in a half-written state across fields. The manifest is convenient
for history listings but must never be trusted as the ground truth.

`RemovalLogger` is an `actor`: a single writer serializes all log output. Writes are batched
(10–50 items or 100 ms) with one `fsync` per batch — this keeps per-item IO cost near zero
even when a module deletes hundreds of artifacts.

Crash-consistency is handled by start-up reconcile: any `.partial` is marked `aborted`,
counts are recomputed from JSONL. No data loss; at worst the session is reported aborted.

Manifest fields that touch user input (`invocation.flags`, `allowDangerFlag`) are
**structured**, not raw command-line: a flag allowlist controls what is persisted, and
token-like strings (`^[A-Za-z0-9_-]{20,}$`) are redacted. See
[`safety.md` §S9](safety.md#s9--output-sanitisation).

---

## Project discovery

Devsweep does not scan `$HOME` implicitly. Modules that need user projects (e.g.
`git.worktrees`, `js.node_modules-lite`) consume `ScanContext.projectRoots`. That array is
populated from the config file `~/.config/devsweep/config.toml`:

```toml
[discovery]
projectRoots = ["~/Projects", "~/dev"]
maxDepth = 6
```

The config is bootstrapped by the explicit `devsweep config init` command (interactive
wizard). If `projectRoots` is empty at scan time, the CLI prints a one-line hint — it never
auto-runs the wizard. Automating a wizard on a read-only command surprises users.

The `devsweep discover` command walks `$HOME` with the default exclusions and suggests
candidate roots; the user approves before they land in config. Discovery results are cached
in `~/.cache/devsweep/project-roots.json` with a 7-day TTL.

---

## Backup exclusion

On first run the CLI calls `CSBackupSetItemExcluded(kCFTypeRef)` on
`~/.local/state/devsweep/`, `~/.cache/devsweep/` and `~/.local/share/devsweep/archive/`.
Time Machine skips those directories by default; users who want different behaviour can
override with `tmutil removeexclusion`. Third-party backup/sync tools (iCloud Drive,
Dropbox, Arq, Backblaze) are outside the app’s control — see
[`accepted-risks.md`](accepted-risks.md).

---

## Further reading

- [`safety.md`](safety.md) — the S1–S14 invariants.
- [`performance.md`](performance.md) — P1–P10 runtime targets.
- [`roadmap.md`](roadmap.md) — what ships when.
- [`accepted-risks.md`](accepted-risks.md) — known trade-offs.
- [`adr/`](adr/) — records of significant decisions (dependencies, benchmark framework,
  etc.).
