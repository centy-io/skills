---
name: centy-daemon-local-development
description: "Use when working on a local Centy daemon checkout."
version: 1.5.0
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

Only with explicit user consent to destroy local changes. The reference lists the steps, the proof, and the stale-submodule handling: `references/local-daemon-operations.md`. Why: the reset is destructive and rarely needed.

## Build and run

- Build the release profile, then run the release binary.
- For non-default ports, pass the gRPC and MCP addresses as flags. Keep the test gRPC port off the MCP port `50052`.
- Proof: a listener or API probe answers. A successful build does not prove the daemon runs.

## Upgrade the local daemon

On macOS, launchd keeps the daemon running. The reference gives the launchd upgrade steps, the backup and rollback rule, and the shared-machine checks: `references/local-daemon-operations.md`. Why: the same procedure serves every upgrade, including release proof.

## Ship a change

Each gate below failed a pull request at least once because it was skipped locally.

- Every new worktree needs its submodules. Initialize `proto` before the first build. Why: a worktree starts with empty submodule directories, and the build script fails on the missing proto files.
- Run clippy exactly as CI does: all targets, warnings denied. Read the flags from the build workflow, not from memory. Why: a plain clippy run skips test targets and passes code that CI rejects.
- Add a changeset file to every pull request. A change with no release note, such as tests or CI, gets an empty changeset. Copy the shape of an existing empty one in the changeset folder. Why: the changeset check fails any pull request without one.
- Prove a bug-fix test: run it against the base code with only the source change reverted, see it fail, then restore the fix. Why: a test that passes on both sides guards nothing.
- For a nondeterministic bug, such as an unordered map or a race, run that check many times on both sides. Why: one passing run on the base code can be luck, not proof that the test guards nothing.

Proof: the local clippy run with the CI flags, the full test suite and the changeset check all pass before you push.


## Cut a release

- **Prepare.** A release bumps the daemon crate version and its lock entry, and adds one hand-written changeset. The changeset lists every change since the last tag. Lead it with any behaviour change, such as an input that was accepted before and is now rejected. Why: callers break silently otherwise.
- **Choose the version on purpose.** The project is pre-1.0. A new RPC, a new built-in type or a behaviour change makes a minor bump. Fixes alone make a patch bump. Why: a minor bump warns callers to check their use.
- **Release only on green.** Main CI must be green on the exact commit, including the MCP smoke step, and the release pull request's own checks must be green. Freeze main against other sessions that merge there. Why: code that lands between the green check and the tag ships untested. If it lands anyway, it ships: add it to the notes. Never claim a release excludes code already on main.
- **Tag the squash merge commit.** Confirm that commit is on main before tagging. Why: the release workflow builds every platform package and the checksums from that tag.
- **Prove the published artefact,** not the pull request build. Check the checksum. Make a session-less tool call and expect the new version. Hold a port and expect a clear error. Send SIGTERM and expect a clean exit with status 0. Then upgrade the local launchd daemon the same way, and delete the backup only after that upgrade is verified. Why: the pull request build can differ from the published one.

## Daemon-written files and project config

- The managed `.centy/.gitignore` ignores only files with a `.local.` infix. Users commit `.centy`. Why: any other daemon-written runtime file, such as a log or history file, leaks into their git status. Give every such file the `.local.` infix, and check a fresh project's git status after the daemon runs.
- Built-in link types must exist inside the daemon. Why: a client that silently writes project config to declare them pollutes committed files and drifts per client. Fix the missing type in the daemon, not in the client.

## Pitfalls

- A process search can show no running daemon while a source checkout exists. Separate source discovery from runtime state.
- A hard reset does not remove untracked files. The clean step is needed for a literal clean reset, and it is destructive.
- Do not claim the daemon runs until a listener or API call confirms it.
- A passing tool-list check does not prove calls work. Always smoke a real tool call. A missing tonic `ready()` call once made every call panic with `buffer full; poll_ready must be called first` while the list call still worked.
- CI uses floating stable clippy. A red build in an untouched area is likely toolchain drift. Check the last run on `main` before blaming the PR.
