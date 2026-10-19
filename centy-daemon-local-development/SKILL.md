---
name: centy-daemon-local-development
description: "Use when working on a local Centy daemon checkout."
version: 1.4.0
author: Hermes Agent
license: MIT
metadata:
  hermes:
    tags: [centy, daemon, rust, grpc, git, submodules]
    related_skills: [github-repo-management, safe-branch-synchronization]
---

# Centy Daemon Local Development

Use for a local checkout of the Centy daemon, its Rust workspace, and its submodules.

## When to Use

- Cloning, repairing, resetting, building, or launching the Centy daemon locally.
- Preparing a daemon change for a pull request.
- Diagnosing a mismatch between an existing daemon checkout and its runtime state.

## Repository model

- Canonical repository: `centy-io/centy-daemon`. Why: other checkouts and forks are not the source of truth.
- The root workspace includes `daemon/` and `xtask/`. The executable crate is `daemon`, which produces the `centy-daemon` binary.
- The checkout needs two required submodules, `installer` and `proto`. The optional `web` submodule may own its own `proto`. Initialize submodules recursively after every clone, rebase, or switch of the pinned web revision.
- Project data lives in a `.centy/` directory as Markdown records with YAML frontmatter. The daemon serves native gRPC and gRPC-Web.
- Default gRPC listener: `127.0.0.1:50051`.
- The daemon also serves a built-in, loopback-only, stateless MCP Streamable HTTP endpoint at `http://127.0.0.1:50052/mcp` (since v0.15.x). Change its bind address with the `--mcp-addr` flag or the `CENTY_MCP_ADDR` variable.
- Clients configure only that URL as an HTTP MCP server. Never start a bridge process: no stdio binary and no npm bridge package, which is obsolete. Why: the daemon already serves MCP, so a bridge adds a second process that can fail silently.
- To serve the bundled web app during local integration, set `CENTY_SERVE_WEB` to 1 when starting the daemon. The daemon then starts the `web` submodule's Next dev server on `127.0.0.1:5180`. Prove both parts: the gRPC listener answers, and the web root answers over HTTP. On shutdown, confirm the child web process also exits.

## Safe checkout discovery

1. Before cloning, inspect the destination and its direct children. A parent directory can hold several independent Centy repositories and is not itself a checkout.
2. For an existing checkout, read its branch and working-tree state, remotes, worktree list, and submodule status before changing anything. A source tree can exist while no daemon runs, so check processes and listeners separately.
3. If another worktree holds `main`, do not force-remove that worktree to switch branches in the primary checkout. On an explicit reset request, reset the current branch to the remote main tip instead. Why: the branch label may differ while the tree is exact.
4. Do not mistake a repository parent for a checkout. Inspect direct children.

## Explicit destructive reset

Run only after the user explicitly asks to destroy local changes. Intent, in order:

1. Fetch the remote main branch with pruning, so the target tip is current.
2. Hard-reset the current branch to that remote tip. Why: this discards tracked changes only.
3. Remove untracked files and directories. Why: a hard reset leaves untracked files behind, so this step is required for a literal clean tree. It is destructive.
4. Initialize submodules recursively again.

If another worktree holds `main`, reset the current branch as described above. If submodule initialization fails on a stale, nonempty, uninitialized submodule directory, inspect that directory first. Remove it only under the same explicit authorization, then retry.

Proof: the status shows a clean tree on the branch, the diff check reports no whitespace errors, and the local HEAD equals the remote main tip read live from origin.

## Build and run

- Build the release profile, then run the release binary.
- For non-default ports, pass the gRPC and MCP addresses as flags. Keep the test gRPC port off the MCP port `50052`.
- Proof: a listener or API probe answers. A successful build does not prove the daemon runs.

## Upgrade the local daemon

On macOS, launchd keeps the daemon running with `KeepAlive`. Discover the job label and the installed binary path by inspection: list launchd jobs filtered for Centy, and resolve the binary on the command search path. Do not assume either value.

1. Back up the installed binary beside itself under a distinct name, before replacing anything. Why: rollback.
2. Copy the release build beside the installed binary under a temporary name, then rename it over the installed path. Why: the rename replaces the file atomically, so a running process never reads a half-written binary.
3. Terminate the old daemon process, found by its PID from the launchd job state. Do not start a new one yourself. Why: the service manager restarts the daemon from the replaced binary.
4. Prove the upgrade with a real, session-less tool call to the MCP endpoint on the port the daemon was started with (default `50052`). No session header and no initialize handshake are needed. Call the daemon-info tool, which reports the version, and expect the new version. An old version means the old process still runs or launchd did not restart the job.
5. Prove the fix by behaviour as well. Builds between releases often report the same version string, so the version alone does not show which build runs. Call the RPC that the change fixed and expect the fixed result.

Several agent sessions may share one machine and upgrade the same daemon. Before the swap, confirm the installed binary is still the one you inspected (same size or checksum). If it changed, another session installed a build: find out what it holds before you replace it. Why: an unguarded swap silently reverts another session's newer build, or theirs reverts yours. After any restart you did not cause, re-check the fixed behaviour, because the live build may not contain it.

## Ship a change

Each gate below failed a pull request at least once because it was skipped locally.

- Every new worktree needs its submodules. Initialize `proto` before the first build. Why: a worktree starts with empty submodule directories, and the build script fails on the missing proto files.
- Run clippy exactly as CI does: all targets, warnings denied. Read the flags from the build workflow, not from memory. Why: a plain clippy run skips test targets and passes code that CI rejects.
- Add a changeset file to every pull request. A change with no release note, such as tests or CI, gets an empty changeset. Copy the shape of an existing empty one in the changeset folder. Why: the changeset check fails any pull request without one.
- Prove a bug-fix test: run it against the base code with only the source change reverted, see it fail, then restore the fix. Why: a test that passes on both sides guards nothing.
- For a nondeterministic bug, such as an unordered map or a race, run that check many times on both sides. Why: one passing run on the base code can be luck, not proof that the test guards nothing.

Proof: the local clippy run with the CI flags, the full test suite and the changeset check all pass before you push.


## Daemon-written files and project config

- The managed `.centy/.gitignore` ignores only files with a `.local.` infix. Users commit `.centy`. Why: any other daemon-written runtime file, such as a log or history file, leaks into their git status. Give every such file the `.local.` infix, and check a fresh project's git status after the daemon runs.
- Built-in link types must exist inside the daemon. Why: a client that silently writes project config to declare them pollutes committed files and drifts per client. Fix the missing type in the daemon, not in the client.

## Pitfalls

- A process search can show no running daemon while a source checkout exists. Separate source discovery from runtime state.
- A hard reset does not remove untracked files. The clean step is needed for a literal clean reset, and it is destructive.
- Do not claim the daemon runs until a listener or API call confirms it.
- A passing tool-list check does not prove calls work. Always smoke a real tool call. A missing tonic `ready()` call once made every call panic with `buffer full; poll_ready must be called first` while the list call still worked.
- CI uses floating stable clippy. A red build in an untouched area is likely toolchain drift. Check the last run on `main` before blaming the PR.
