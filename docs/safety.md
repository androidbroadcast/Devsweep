# Safety model

Devsweep deletes files. The whole product category has a long history of inflicting damage
on users; we assume every piece of code will eventually be wrong, and design so that
mistakes are caught by the next layer. This document lists the invariants every contributor
and reviewer must keep.

If you are asking «should I skip one of these?» — the answer is no.

The invariants are numbered **S1–S14** and referenced across the codebase and in every issue
that touches a destructive path.

---

## S1 — Single destructive path

All destructive operations route through one service: `Remover` in `DevsweepCorePrivate`.
Modules **never** call `FileManager.removeItem`, `FileManager.trashItem`, `unlink`, nor spawn
processes that do.

A module expresses intent through `plan()` returning `[CleanAction]`. `Remover` executes.

Enforcement: CI analyses the SwiftPM import graph; any `DevsweepModules` file referencing
`FileManager.removeItem`, `trashItem`, `Foundation.Process`, or `Darwin.posix_spawn` fails
the build. See [`architecture.md` §Package layout](architecture.md#package-layout).

## S2 — Global deny-list

Before any action runs, `Remover` consults `GlobalDenyList` as the last line of defence.
The list is hardcoded — not read from config — and overriding it requires
`--allow-danger=<policy-id>` _and_ an interactive typed confirmation (`DELETE N items`).

Categories always denied:

- **SSH / GPG** — `~/.ssh/**`, `~/.gnupg/**`
- **Cloud credentials** — `~/.aws/credentials`, `~/.aws/config`, `~/.config/gcloud/**`, `~/.azure/**`
- **Package publish tokens** — `~/.npmrc`, `~/.yarnrc`, `~/.yarnrc.yml`, `~/.pypirc`,
  `~/.cargo/credentials.toml`, `~/.gem/credentials`, `~/.m2/settings.xml`,
  `~/.m2/settings-security.xml`
- **Git / VCS credentials** — `~/.netrc`, `~/.authinfo`, `~/.authinfo.gpg`,
  `~/.git-credentials`, `~/.config/gh/hosts.yml`, `~/Library/Application Support/GitHub CLI/**`
- **K8s / Docker / Vault / Terraform** — `~/.kube/config`, `~/.docker/config.json`,
  `~/.docker/contexts/**`, `~/.vault-token`, `~/.terraform.d/credentials.tfrc.json`,
  `~/.terraformrc`
- **Keychain** — `~/Library/Keychains/**`
- **Dotenv / secrets files** — `**/.env`, `**/.env.local`, `**/.env.*.local`,
  `**/.env.production`, `**/secrets.yml`, `**/secrets.yaml`
- **Build-tool credential propagation** — `~/.gradle/gradle.properties`,
  `~/.sbt/.credentials`
- **Lockfiles (exact names)** — `Cargo.lock`, `package-lock.json`, `yarn.lock`,
  `pnpm-lock.yaml`, `bun.lockb`, `Gemfile.lock`, `Podfile.lock`, `composer.lock`,
  `poetry.lock`, `uv.lock`, `go.sum`, `pdm.lock`, `mix.lock`
- **Local databases (Homebrew)** — `/opt/homebrew/var/postgres*/**`,
  `/opt/homebrew/var/mysql/**`, `/opt/homebrew/var/mongodb/**`,
  `/opt/homebrew/var/redis/**`, equivalents under `/usr/local/var/`
- **System paths** — `/System/**`, `/Library/Preferences/**`,
  `~/Library/Preferences/**`

Secret-like paths are also **hidden from scan output entirely** — not merely skipped during
clean. Reporting «we would have deleted `~/.ssh/id_rsa`» is itself a leak.

## S3 — Symlink policy

`CleanAction.trash(_, symlinkPolicy:)` takes an explicit policy, not a boolean:

```swift
public enum SymlinkPolicy: Sendable {
    case rejectIfSymlink
    case trashSymlinkOnly
    case trashTargetRequireSafe(maxDepth: Int)
}
```

In managed trees that require external CLI-only deletion — `~/.ollama/`,
`~/.cache/huggingface/`, `~/Library/Caches/CocoaPods/` — `Remover` rejects `.trash` actions
regardless of policy and requires `.externalCommand("hf", ...)` / `.externalCommand("ollama",
...)` / `.externalCommand("pod", ...)`. Recursive fallback is forbidden.

## S4 — TOCTOU atomicity

Between `scan` and `clean --apply` minutes or hours may pass. Between `stat` and `unlink` in
our own code, microseconds may pass. Both windows are exploitable via symlink swap
([CWE-367], [CWE-59]).

Therefore:

- **File actions** — `open(O_NOFOLLOW | O_CLOEXEC)` → `fstat` confirming inode, size, mtime
  match the scan snapshot → `unlinkat(AT_SYMLINK_NOFOLLOW)` or `renamex_np` into the Trash
  staging directory, all using the held directory descriptor.
- **Directory actions** — resolve through `realpath` at scan time; on each descend during
  `clean`, re-open with `O_NOFOLLOW` and verify the resolved path did not acquire a symlink
  component.

If any check fails on an item, that action is aborted with
`reason=toctou_mismatch | resource_busy | deny_list_matched`; remaining actions proceed.

[CWE-367]: https://cwe.mitre.org/data/definitions/367.html
[CWE-59]: https://cwe.mitre.org/data/definitions/59.html

## S5 — Risk levels

Every module declares a `RiskLevel`:

- `.safe` — rebuild-on-demand (e.g. Xcode DerivedData).
- `.reversible` — goes to Trash; recoverable via `devsweep restore`.
- `.irreversible` — Docker prune, HuggingFace cache purge, Ollama model delete. Trash is
  not involved; bytes are actually released.

For `.irreversible`, the CLI requires typed confirmation: the user types
`DELETE N items` verbatim. `--allow-danger` is never sufficient on its own.

## S6 — File permissions

- Removal log (`~/.local/state/devsweep/removed.jsonl`): `0600`
- Session manifests (`~/.local/state/devsweep/sessions/*.json`): `0600`
- Archives (`~/.local/share/devsweep/archive/`): directories `0700`, files `0600`
- Config (`~/.config/devsweep/config.toml`): `0600`
- Scan cache (`~/.cache/devsweep/`): `0700` / `0600`

Permissions are set explicitly after creation via POSIX attributes and re-asserted at start-
up.

## S6a — Backup exclusion

`CSBackupSetItemExcluded` is called on `~/.local/state/devsweep/`, `~/.cache/devsweep/`,
`~/.local/share/devsweep/archive/` on first run, so Time Machine doesn’t silently copy
removal logs or archived sessions to external media.

Third-party sync tools are outside this control — users who sync `$HOME` to iCloud / Dropbox
must accept the exposure. See [`accepted-risks.md`](accepted-risks.md).

## S7 — Claude sessions: trash by default

Claude Code stores conversation transcripts in `~/.claude/projects/**/*.jsonl`. Transcripts
can contain API keys pasted by the user, proprietary code, PII. The default plan for
`ai.claude.sessions` is **trash + warning**, not archive:

- `clean --module ai.claude.sessions --apply` prints a warning about plaintext content and
  moves files to the Trash.
- Archive is opt-in via `--archive`. The archive is plaintext zstd with `0600` permissions
  in 0.1; encryption-at-rest via AES-GCM with a Keychain-stored key ships in 0.3+.
- Users who archive should know: that directory must not be included in unencrypted
  backups.

### S7a — archive-verify-fsync-then-trash

Archive flow is strictly ordered so no step can leave a half-written archive paired with a
deleted original:

1. Write zstd file.
2. Update JSON index through a temp file and atomic rename.
3. `fsync` the file + the parent directory.
4. Verify: `fstat` reports size > 0, sha256 (computed streaming during compress; see
   [`performance.md` §P4](performance.md#p4--archive-streaming-hash)) matches the input
   hash.
5. **Only after** all four steps — move the original to Trash.

Any failure in steps 1–4 leaves the original intact and removes the partial archive.

## S8 — Module-level isolation

Third-party plugins inherit the trust level of devsweep itself, so defence-in-depth is done
at the _build_ level rather than runtime sandboxing:

- `Foundation.Process`, `Darwin.posix_spawn`, `FileManager.removeItem`,
  `FileManager.trashItem` are **only available in `DevsweepCorePrivate`**.
- `DevsweepModules` imports `DevsweepCore` only. `DevsweepCorePrivate` is `internal
  import`-ed only in the composition root (the `devsweep` target).
- Modules call external processes exclusively through `CommandRunner` injected via context.
  `CommandRunner` enforces an allowlist on `tool` names (`git`, `hf`, `ollama`, `pod`,
  `brew`, `docker`, `colima`, `simctl`, `xcrun`) and rejects binaries resolved from
  `/tmp`, `/var/tmp`, `~/Downloads`, `~/Desktop`.
- CI runs import-graph analysis (not text grep) to enforce all of the above. See
  [`architecture.md` §Package layout](architecture.md#package-layout).

## S9 — Output sanitisation

Every path-shaped string rendered to the user or written to a log passes through
`displaySafe(path:)`:

- Strip ANSI (`\x1b[...`, `\x9b`), control chars (U+0000–U+001F, U+007F).
- Strip BiDi override codepoints (U+202A–U+202E, U+2066–U+2069).
- Non-printable bytes rendered as `\xNN`.

This defeats the «filename pretending to be a prompt» category of attack ([CWE-116],
[CWE-150]).

JSON output of paths carries two fields:

- `"path"` — raw-escaped (safe to parse).
- `"path_display"` — `displaySafe`-ified (safe to print).

Consumers choose.

[CWE-116]: https://cwe.mitre.org/data/definitions/116.html
[CWE-150]: https://cwe.mitre.org/data/definitions/150.html

## S10 — Permission-denied paths are reported explicitly

When the CLI lacks privileges to access a path (for example `/Library/Logs/DiagnosticReports/`
without Full Disk Access), the scanner **does not skip silently**. The path appears in
`--json` output with
`{"skipped": true, "reason": "permission_denied_fda_required"}` and in human-readable output
as a diagnostic line.

Silent skips hide the fact that part of the user’s scope was not considered.

## S11 — Resource-lock quiescence

Cleanup can corrupt state if the owning process is running. Modules declare:

```swift
var requiredQuiescence: [ProcessPredicate] { get }
```

Checks use native APIs, not `lsof` sub-shells (see
[`performance.md` §P5](performance.md#p5--resource-lock-checks)):

- `noProcessWithBundleID(String)` and `noProcessWithBundleIDPrefix(String)` via
  `NSRunningApplication.runningApplications(withBundleIdentifier:)`.
- `noOpenHandleToPath(URL)` via `proc_listpids` + `proc_pidinfo(PROC_PIDLISTFDS)` batched
  for the whole session.
- `noLaunchdService(String)` via `launchctl list`.
- `noGitOperationInWorktree(URL)` via inspection of `.git/index.lock`,
  `.git/HEAD.lock`, etc.

For `.irreversible` modules (Docker, Gradle daemon), `flock()` advisory locks on marker
files supplement the best-effort checks.

## S12 — `--allow-danger` is argv-only and TTY-only

The `--allow-danger=<policy-id>` flag:

- Is read **only** from `CommandLine.arguments` — never from config, aliases, environment.
- Does not suppress the typed confirmation prompt; it only unlocks the possibility of
  confirming.
- In non-TTY contexts (CI, cron, `nohup`), `.irreversible` actions abort unconditionally
  unless a separate explicit `--i-understand-this-is-ci-and-accept-data-loss` flag is also
  passed. This flag is likewise argv-only.

The design assumes that automation wrappers can and do pile additional flags on commands —
the confirmation must stay in the loop.

## S13 — Plugin trust model

For 0.1, all plugins are **bundled** and compiled into the binary. A third party produces a
new devsweep binary themselves. Runtime plugin loading is explicitly not supported.

Future: if runtime plugins ever land, they will do so through an external-executable
protocol (stdin/stdout JSON), not via `dyld`. See [`architecture.md`](architecture.md#the-plugin-api).

## S14 — Input validation grammars

Every argument that becomes a path component, a module id, a date, or a config lookup is
matched against a strict grammar before it reaches any effectful code. Invalid input → exit
code 2 + short error.

| Argument | Grammar |
|---|---|
| `--session <id>` | `^\d{4}-\d{2}-\d{2}-\d{6}-[0-9a-f]{8}-[0-9a-f]{4}-[0-9a-f]{4}-[0-9a-f]{4}-[0-9a-f]{12}$` |
| `--since <value>` | `^\d{1,4}(d\|w\|m\|y)$\|^\d{4}-\d{2}-\d{2}$` — capped at 10 years |
| `--module <id>` | `^[a-z0-9]+(\.[a-z0-9-]+)*$` — must also exist in `ModuleRegistry` |
| `--allow-danger=<id>` | fixed whitelist of known policy ids |

`--session` ids additionally pass a `realpath` boundary check: after constructing the file
path, the resolved canonical path must start with
`~/.local/state/devsweep/sessions/`. This catches `../`-style traversal even if the regex
were later relaxed.

---

## Cross-references

Every task issue that touches a destructive code path carries a safety checklist in its
body. The checklist maps directly to the invariants above; see
`.github/ISSUE_TEMPLATE/task.yml`.

Security vulnerability reports: please use GitHub Security Advisories. Do not open public
issues.
