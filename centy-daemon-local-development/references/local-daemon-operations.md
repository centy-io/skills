# Local daemon operations

Moved out of `SKILL.md` to keep the always-loaded file short. Read this when a task needs a destructive reset or a launchd upgrade.

## Explicit destructive reset

Run only after the user explicitly asks to destroy local changes. Intent, in order:

1. Fetch the remote main branch with pruning, so the target tip is current.
2. Hard-reset the current branch to that remote tip. Why: this discards tracked changes only.
3. Remove untracked files and directories. Why: a hard reset leaves untracked files behind, so this step is required for a literal clean tree. It is destructive.
4. Initialize submodules recursively again.

If another worktree holds `main`, reset the current branch as described above. If submodule initialization fails on a stale, nonempty, uninitialized submodule directory, inspect that directory first. Remove it only under the same explicit authorization, then retry.

Proof: the status shows a clean tree on the branch, the diff check reports no whitespace errors, and the local HEAD equals the remote main tip read live from origin.

## Upgrade the local daemon

On macOS, launchd keeps the daemon running with `KeepAlive`. Discover the job label and the installed binary path by inspection: list launchd jobs filtered for Centy, and resolve the binary on the command search path. Do not assume either value.

1. Back up the installed binary beside itself under a distinct name, before replacing anything. Why: rollback.
2. Copy the release build beside the installed binary under a temporary name, then rename it over the installed path. Why: the rename replaces the file atomically, so a running process never reads a half-written binary.
3. Terminate the old daemon process, found by its PID from the launchd job state. Do not start a new one yourself. Why: the service manager restarts the daemon from the replaced binary.
4. Prove the upgrade with a real, session-less tool call to the MCP endpoint on the port the daemon was started with (default `50052`). No session header and no initialize handshake are needed. Call the daemon-info tool, which reports the version, and expect the new version. An old version means the old process still runs or launchd did not restart the job.
5. Prove the fix by behaviour as well. Builds between releases often report the same version string, so the version alone does not show which build runs. Call the RPC that the change fixed and expect the fixed result.

Several agent sessions may share one machine and upgrade the same daemon. Before the swap, confirm the installed binary is still the one you inspected (same size or checksum). If it changed, another session installed a build: find out what it holds before you replace it. Why: an unguarded swap silently reverts another session's newer build, or theirs reverts yours. After any restart you did not cause, re-check the fixed behaviour, because the live build may not contain it.
