# Performance model

Devsweep runs on the user’s own machine in interactive and scheduled contexts. The rules
below are not suggestions — they are invariants. Slow or unthrottled scans compete with
Xcode builds, Docker pulls, and everything else the user actually wants to be doing.

The invariants are numbered **P1–P10** and referenced across the codebase.

---

## P1 — FastWalker

- Back the walker with `open(O_DIRECTORY)` + `getattrlistbulk(2)`, not
  `FileManager.enumerator`. `getattrlistbulk` returns a batch of attributes per syscall
  (name, inode, type, size, modification time); `FileManager.enumerator` is `readdir` +
  per-file `stat`. Measured difference is 3–7× on APFS SSD.
- Parallelise across `projectRoots` using `withTaskGroup` with bounded concurrency **4–8**.
  Unbounded parallelism hits APFS copy-on-write lock contention and regresses throughput.
- On every descend: canonicalise the child URL (`realpath`), then check
  `excludedBasenames` (exact match), then `excludedPathPrefixes` (startsWith), then
  `GlobalDenyList`.

**Target: 100 000 files in under 5 seconds on Apple Silicon SSD.** Verified by
`AC-30` benchmark (see [`../tests`](../Tests/) once the suite lands).

Profile with `fs_usage -w -f filesys devsweep` during walk — expected syscall counts
should be one `getattrlistbulk` per batch of ~128 children, not one `getattr` per file.

## P2 — Batched streaming scan

`CleanupModule.scan()` returns `AsyncThrowingStream<[ScanItem], Error>`. Emit chunks of
**100–500 items or roughly 256 KB**, whichever comes first.

Per-item `yield` adds 1–5 µs of `await` suspend/resume overhead; at 50 000 items that’s
50–250 ms of pure scheduling burnt. Batched streaming keeps the cost negligible and lets
consumers (CLI progress, history writer) process a chunk atomically under back-pressure.

Per-item APIs are available as helper iterators built on top
(`stream.flatMap { $0 }`).

## P3 — Batched fsync

`RemovalLogger` (an `actor`) buffers JSONL entries in memory and flushes with one `fsync`
per **10–50 items or 100 ms timer** — whichever comes first.

Per-item `fsync` is 1–10 ms on SSDs. For `js.node_modules` with ~500 items that would be
0.5–5 seconds of pure IO wait purely for logging. Batching brings it to <100 ms.

Crash safety is preserved by the append-only format: a truncated trailing line at crash
time is detected by parsing and discarded. The next session picks up with a fresh entry.

Target: **`AC-31` — fsync syscall count ≤ `ceil(n / 50) + 2` for n item actions in a
session.** Measurable with `fs_usage -w -f filesys devsweep | grep fsync`.

## P4 — Archive streaming hash

`ArchiveService` computes sha256 of the source **during** compression, not via a second
pass after writing the archive:

1. Open source.
2. Read chunk → feed to `CryptoKit.SHA256.Hasher` _and_ to the zstd encoder.
3. Encoder writes zstd bytes to destination.
4. On EOF: finalise hash, write index entry, fsync.

Verify = `fstat(dest).size > 0` + hash comparison. No re-decompress. For a 200 MB Claude
JSONL archive this saves +400 MB of IO and 2–5 s of wall-clock.

A full re-decompress would be «theatre of safety», not safety: the only failure modes it
catches that streaming-hash-at-write does not are bugs inside the zstd library itself,
which is not our threat model.

Target: **`AC-32` — source read exactly once per archive operation.**

## P5 — Resource-lock checks

Modules declare `requiredQuiescence: [ProcessPredicate]`. Implementation uses macOS
native APIs, not `lsof` sub-processes:

- `noProcessWithBundleID(_)` / `noProcessWithBundleIDPrefix(_)` →
  `NSRunningApplication.runningApplications(withBundleIdentifier:)`. Near-instant, pure API.
- `noOpenHandleToPath(URL)` → **batch** `proc_listpids(PROC_ALL_PIDS)` +
  `proc_pidinfo(_, PROC_PIDLISTFDS, ...)`. One syscall pair per process, scanned once per
  session, not per path.
- `noLaunchdService(_)` → `launchctl list` parsed once and cached for the session.

Per-item `lsof`-fork is forbidden. Cold `lsof` process start-up is ~30–100 ms — on a 30-item
xcode.deriveddata session that alone would be up to 3 seconds of pure fork overhead.

For `.irreversible` modules (Docker, Gradle daemon), supplement the best-effort check with
`flock()` advisory locks on marker files to minimise the TOCTOU window.

## P6 — Size computation

`ScanItem.directory(totalSize:)` is computed via `getattrlistbulk` recursive walk
accumulating `ATTR_FILE_ALLOCSIZE` (physical block allocation — the space actually freed),
parallelised over sub-directories with a bounded task group.

For very large hot trees (>10 GB, >100 000 files) a lazy strategy is acceptable: emit `-1`
as sentinel, let a background task compute actual size, and re-emit through the stream
with updated value. The CLI renders «computing size…» in progress.

Target: DerivedData at 50 GB / 50 000 files completes in under 3 s on Apple Silicon SSD.

## P7 — SQLite history index (0.2+)

When `history-index.sqlite` lands in 0.2, it is **incremental** — not rebuild-from-scratch.

The SQLite table `ingested` stores `last_processed_offset` per JSONL file. On start-up the
indexer reads only the new bytes since the last offset. For a 100 MB archived month plus
200 KB of fresh data that’s <50 ms, not the 1–3 s a full re-ingest would take.

Full rebuild (`devsweep history rebuild-index --force`) runs in a background Task with
`.background` priority. The cache is non-authoritative — `restore` and safety logic read
primary JSON/JSONL directly, never the SQLite.

## P8 — launchd scheduled scan (0.3+)

For scheduled scans via `launchd`, the process must not starve the user’s foreground work.

`launchd` plist:

```xml
<key>LowPriorityIO</key><true/>
<key>ProcessType</key><string>Background</string>
<key>Nice</key><integer>10</integer>
```

In Swift entry, when launched in background mode (detected via env var / `-background`
flag):

```swift
setiopolicy_np(IOPOL_TYPE_DISK, IOPOL_SCOPE_PROCESS, IOPOL_THROTTLE)
Task(priority: .background) { ... }
```

Accepted trade-off: background-mode scan takes 2–3× wall-clock vs interactive mode. That
cost buys non-interference with Xcode builds, Docker pulls, and screen-recording.

## P9 — ProgressReporter rate limit

Modules call `progress.report(item:)` freely — the _implementation_ throttles. The
reporter flushes at most **30 times per second** (≥34 ms between flushes) measured via the
injected `Clock`. Between flushes it accumulates counts, current path, bytes-processed.

Without this throttle, a fast walker at 20 000 items / second turns the terminal redraw
itself into the bottleneck.

## P10 — Hardlink dedup (for 0.2+ Ollama / HuggingFace)

Ollama stores model blobs as hardlinks across manifests. HuggingFace uses symlinks into a
shared `blobs/` store. Naive recursive size computation double-counts.

Modules scanning such trees use a `Set<FileIdentifier>` where
`FileIdentifier = (dev_t, ino_t)` for inline dedup: the first encounter of an inode counts
size; subsequent encounters are skipped from the size total. Same behaviour as
`du --count-links`.

Without dedup, reported «size that will free» can be 3–5× actual.

---

## Profiling commands

Recommended when working on perf-sensitive code:

- `fs_usage -w -f filesys devsweep` — syscall profile during a walk. Count
  `getattrlistbulk` vs `getattr` vs `stat` to validate P1.
- `xctrace record --template "Time Profiler" --launch devsweep ...` — Instruments Time
  Profiler for CPU/suspend costs (P2, P9).
- `iotop` / `sudo fs_usage` during a launchd-triggered scan — verify IO throttling is
  effective (P8).
- `dtrace -n 'syscall::fsync:entry /execname=="devsweep"/ { @ = count(); }'` — fsync
  counting (P3).

---

## Benchmarks in CI

Performance invariants are validated by the benchmark suite (target: `DevsweepPerfTests`).
CI runs benchmarks on pull-requests in advisory mode for 0.1 — regressions are reported
but do not block merge. From 0.2 onwards, critical benchmarks (P1, P3, P4) become blocking.

Results are captured as JSON artifacts attached to each CI run; trend visualisation is
post-1.0 work.
