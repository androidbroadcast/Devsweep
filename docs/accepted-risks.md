# Accepted risks

Devsweep makes trade-offs. Some are defensible, some are honest acknowledgement that a
class of attack is out of scope. Both kinds belong here so neither users nor contributors
are surprised.

If any of these accepted risks changes status (moves into scope, or is mitigated) — update
this document in the same PR that changes the behaviour.

---

## Malware running under the user’s UID

Any process already running as the user shares the user’s filesystem permissions. `0600`
log files and `0700` archive directories protect against _other_ local users, not against
same-UID malicious code.

Complete protection would require a hardened-runtime sandbox with a curated entitlements
list; that is out of scope for 0.1. Mitigations in effect:

- Keychain-based encryption key for Claude archive (0.3+) raises the bar.
- `hardened-runtime` is enabled for notarized builds.
- `SECURITY.md` specifies what to look for if a report arrives.

## Third-party backup and sync tools

Time Machine is excluded from devsweep state/cache/archive directories via
`CSBackupSetItemExcluded`. iCloud Drive, Dropbox, Google Drive, Arq, Backblaze and similar
third-party sync or backup tools do **not** honour this flag — they replicate whatever
lives inside the paths they are told to watch.

If you have synced `$HOME` (or a subtree containing `~/.local/state/`, `~/.cache/`,
`~/.local/share/`) to such a service, your removal log / archived Claude sessions /
history manifests may be replicated off-device. README states this; users with unusual
setups accept the exposure.

Full-disk encryption (FileVault) masks this at rest but not against the sync vendor.

## `lsof` / resource-lock race

The resource-lock service checks whether a process holds a file handle at the moment of
the scan. A process may open the file immediately after the check and before the unlink.

We minimise the window through native `proc_listpids` + `proc_pidinfo` (no fork overhead),
batched per session. For `.irreversible` modules (Docker, Gradle daemon), `flock()`
advisory locks on marker files add a serialisation point. Beyond that, TOCTOU at the
kernel level requires advisory locking the kernel does not expose to userspace on paths.

For `.safe` and `.reversible` modules (Xcode DerivedData, Claude sessions, etc.) the cost
of a race is negligible — files recreate, Trash restores.

## Removal log integrity

`~/.local/state/devsweep/removed.jsonl` is `0600` — protected against other users. Against
same-UID malware (see above), a crafted line could mislead `restore`.

HMAC-signed log entries are planned for 0.3+. Until then: `restore` re-validates each
action against on-disk reality before performing it; if a manifest references a Trash item
that does not exist, the restore skips that item and reports it in output.

Accepted for 0.1.

## Plugin trust model (v0.1: bundled-only)

Third-party plugins in 0.1 require recompiling devsweep. That is the trust boundary. If
your binary links a third-party module, you trust that module as much as the bundled ones.

A future runtime-plugin mechanism will use external executables with `sandbox-exec`
seatbelt profiles, not `dyld`. No ETA — it will ship when there’s a clear, maintainable
design. Until then, the answer to «can I load plugins at runtime?» is no.

## SQLite history index is non-authoritative (0.2+)

When `history-index.sqlite` arrives in 0.2, it is a **cache projected** from the JSONL
and session manifests. All safety-relevant logic (`restore`, `history show`) reads the
primary files directly. If the SQLite index is corrupted or wiped, nothing is lost; it
rebuilds incrementally from the source of truth.

Corollary: users who want fully clean state can `rm ~/.cache/devsweep/history-index.sqlite`
without consequence.

## `--allow-danger` accepts user liability

Overriding `GlobalDenyList` via `--allow-danger=<policy-id>` is a deliberate, audited
choice. The flag:

- is argv-only (not read from config/env/aliases),
- requires an interactive typed confirmation,
- aborts unconditionally in non-TTY contexts unless an additional
  `--i-understand-this-is-ci-and-accept-data-loss` flag is passed (also argv-only).

The design assumes automation wrappers will accumulate flags; the confirmation must stay
in the loop. If you override the deny-list and destroy something important, that is an
expected failure mode of the override, not a bug.

## Surface that depends on undocumented paths

Claude Code JSONL format, Cursor `workspaceStorage` layout, Ollama blob structure,
HuggingFace hub layout — these are not public contracts. Vendors can and have reshuffled
them. Devsweep validates the structure it expects before acting; when a mismatch is
detected, the module aborts soft with a diagnostic rather than guessing.

Accepted: some modules may stop detecting targets on the day a vendor releases a
breaking change. Mitigation: CI matrix with current versions of the tracked tools
(coming in 0.2+ as those modules land).

---

## Reporting

If you find a safety or security issue that is **not** listed above, or you believe
something listed above should be mitigated sooner than planned: open a private GitHub
Security Advisory on this repository.
