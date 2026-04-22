# Roadmap

Public version plan for Devsweep.

---

## 0.1 — Core + 4 modules

Proof of architecture. Establishes the Plugin API, the safety layer, and the performance
baseline. Four modules cover the three primary API shapes so the contract is validated
empirically before more modules are added.

**Included:**
- Core Plugin API (`CleanupModule`, `ScanItem`, `CleanAction`, `Category`, `SymlinkPolicy`,
  `RiskLevel`, `ArchiveIndexEntry`, `DiscoveredRoot`, `ProjectMarker`).
- Core services — `FileSystem`, `CommandRunner`, `ResourceLockService`, `Logger`,
  `ProgressReporter`, `Clock`, `ConfigLoader`.
- Safety layer — S1–S14 ([details](safety.md)).
- Performance invariants — P1–P10 ([details](performance.md)).
- `FastWalker` via `getattrlistbulk` + parallel.
- `RemovalLogger` actor with batched fsync, crash-consistent session manifests.
- `ArchiveService` with streaming hash (archive-verify-fsync-then-trash).
- `GlobalDenyList` comprehensive.

**Modules:**
- `xcode.deriveddata` — folder-scan + Trash.
- `ai.claude.sessions` — folder-scan + archive-mode (opt-in `--archive`).
- `git.worktrees` — `FastWalker` + external `git` CLI + multi-criteria heuristic.
- `js.node_modules-lite` — `FastWalker` for `package.json`, report-only (no clean in
  0.1).

**CLI:**
- `scan`, `clean`, `restore`, `history`, `history show`, `discover`,
  `config init`, `config show`.
- Flags: `--json`, `--interactive`, `--archive`, `--apply`, `--allow-danger`, `--redact`
  (global, two-level), `--since`, `--module`, `--overwrite`, `--suffix`, `--all`, `-n`,
  `--i-understand-this-is-ci-and-accept-data-loss`.

**Retention / ambient:**
- _None in 0.1._ Retention lands in 0.3.

**Distribution:**
- GitHub Releases + Homebrew tap. Notarized. Direct, open-source, donate-backed.

**Out of scope for 0.1:**
- `history stats` + SQLite history index → 0.2.
- GUI, tray, App Store.

---

## 0.2 — Plugin expansion

More modules. The Plugin API validated in 0.1 should absorb these without breaking
changes; anything that does break is a bug in the API, not in the module.

**Modules added:**
- `gradle.caches` — `~/.gradle/caches/`, daemon, stale `wrapper/dists`.
- `js.node_modules` extended — adds clean plan over the 0.1 lite scanner.
- `ai.cursor.workspaceStorage` — orphaned Cursor workspace storage (commonly 50+ GB).
- `jetbrains.caches` — `~/Library/Caches/JetBrains/*`.
- `sdk.xcode-versions` — duplicate Xcode.app / CommandLineTools / incompatible
  DeviceSupport.
- Package-manager runners-up: Rust `target/`, Cargo registry, Homebrew cleanup, Python
  `__pycache__/venv`, pnpm/yarn/bun stores, Maven, CocoaPods, pub-cache.

**Infra added:**
- `devsweep history stats` — aggregated totals (freed, by module, by month).
- `history-index.sqlite` in `~/.cache/devsweep/` — **non-authoritative** projection of
  JSONL for fast queries. Incremental index (see [`performance.md` §P7](performance.md#p7--sqlite-history-index-02)).
  Raw `libsqlite3` only; prepared statements; not a runtime dependency.
- Hardlink dedup (`Set<(dev_t, ino_t)>`) for scanners touching Ollama/HuggingFace-shaped
  trees.

---

## 0.3 — Retention + TUI (optional)

The CLI becomes feature-complete: it schedules itself, notifies the user, and (optionally)
provides a rich in-terminal UI.

**Added:**
- `launchd` scheduled scan — weekly `devsweep scan` via a bundled plist. Background QoS
  (`LowPriorityIO`, `ProcessType=Background`), `setiopolicy_np(IOPOL_THROTTLE)`,
  `Task.priority=.background`. See [`performance.md` §P8](performance.md#p8--launchd-scheduled-scan-03).
- macOS `UserNotifications` integration — «you’ve got X GB of dev junk since last week».
- `--interactive` prompt-per-item mode.
- Optional **rich TUI** (`devsweep tui`) — full-screen navigation à la `lazygit` /
  `npkill`, keyboard bulk-select, drill-down preview. Opt-in subcommand.

Encryption at rest for Claude session archives — AES-GCM with a Keychain-stored key, opt-
in via `--encrypt-archive`.

---

## 1.0 — Feature-complete CLI

Stability milestone. CLI + TUI feature-complete, notarized release, Homebrew tap stable.

Before 1.0 ships, the pre-1.0 checklist must be satisfied:

- Trademark review for module ids that reference third-party products (Claude, Cursor,
  Ollama, HuggingFace, …).
- `SECURITY.md` final with threat model, accepted risks, PGP key, disclosure channel.
- Fuzz testing of `displaySafe` and the JSON encoder on adversarial path names.
- Upgrade-path tests between 0.x versions (JSONL format migrations if any).
- Homebrew formula tested end-to-end from a fresh macOS install.

Post-1.0 work continues in 1.x — the CLI remains the primary interface.

---

## 1.x — GUI (post-1.0)

Swift / SwiftUI application consuming the same Plugin API.

**Expected:**
- Drill-down disk visualisation (treemap / sunburst) with dev-context annotations.
- Per-category / per-project breakdown, not just per-directory.
- Preview + bulk-clean dialog with confirmation matching CLI safety contract.
- Dark / light themes, accessibility.

The CLI continues shipping and evolving. GUI is a complement, not a replacement.

---

## 2.x — Tray / menu-bar + FSEvents monitoring

Always-visible presence in the macOS menu bar. Passive monitoring via `FSEvents` on
tracked dev directories, proactive hints («`~/.claude/projects/` accumulated 12 GB in the
last 30 days — archive?»).

This tier depends on having a hardened GUI codebase and real usage data. It is
intentionally far out in the roadmap so we don’t trade product quality for surface area.

---

## Later — Repo integration

Devsweep becomes part of the rubbish workflow, not external to it.

- `.devsweep.toml` per-repo config — project-specific rules (e.g. this project’s
  `node_modules` are never cold, never prompt).
- `pre-commit` / `pre-push` hooks that suggest cleanup before a push if local tree is
  bloated.
- Git-aware clean — refuse to remove artifacts whose parent repo has uncommitted work.

---

## App Store?

Not a primary release channel.

Sandboxed apps cannot reach `~/Library/Developer/*`, `~/.gradle/*`, `~/.claude/*`, etc.
without the user selecting each folder through the security-scoped file picker — which
defeats the one-click cleanup UX. If an App Store version ever ships, it will be a separate
sandboxed GUI target with a restricted scope (`~/Downloads`, Trash management only), sold
at a nominal price for convenience.

The main product stays direct + open-source + donate-backed.
