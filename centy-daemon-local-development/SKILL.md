---
name: centy-daemon-local-development
description: "Use when working on a local Centy daemon checkout."
version: 1.1.0
author: Hermes Agent
license: MIT
metadata:
  hermes:
    tags: [centy, daemon, rust, grpc, git, submodules]
    related_skills: [github-repo-management, safe-branch-synchronization]
---

# Centy Daemon Local Development

Use for a local checkout of `centy-io/centy-daemon`, its Rust daemon, and its submodules.

## When to Use

- Cloning, repairing, resetting, building, or launching Centy daemon locally.
- Diagnosing a mismatch between an existing daemon checkout and its runtime state.

## Repository model

- Canonical repository: `https://github.com/centy-io/centy-daemon.git`.
- The root workspace includes `daemon/` and `xtask/`; the executable crate is `daemon`, producing `centy-daemon`.
- The checkout has required `installer` and `proto` Git submodules. The optional/consumer-facing `web` submodule may itself own `proto`; always initialize with `git submodule update --init --recursive` after cloning, rebasing, or switching the pinned web revision.
- Centy stores project data in a `.centy/` directory as Markdown records with YAML frontmatter. The daemon serves native gRPC and gRPC-Web; its default listener is `127.0.0.1:50051`.
- The daemon also serves a built-in, loopback-only, stateless MCP Streamable HTTP endpoint at `http://127.0.0.1:50052/mcp` (since v0.15.x). Select another loopback address with `--mcp-addr` or `CENTY_MCP_ADDR`. Clients configure only that URL, for example `{"type":"http","url":"http://127.0.0.1:50052/mcp"}` or `claude mcp add --transport http centy http://127.0.0.1:50052/mcp`. Never use `npx centy-mcp` or a stdio binary; the npm bridge is obsolete.
- To launch the bundled web app during local integration, use `CENTY_SERVE_WEB=1 ./target/release/centy-daemon`. It starts the `web` submodule's Next dev server on `127.0.0.1:5180`; verify both the gRPC listener and `curl --fail http://127.0.0.1:5180/`, then terminate the daemon and confirm its child web process exits.

## Safe checkout discovery

1. Before cloning, inspect the intended destination and its direct children. A parent directory can contain several independent Centy repositories rather than itself being a repository.
2. For an existing daemon checkout, inspect `git status --short --branch`, remotes, `git worktree list --porcelain`, and `git submodule status` before mutating it. A source tree can be present but not running; check processes/listeners separately.
3. If another worktree has `main` checked out, do not force-remove that worktree just to switch branches in the primary checkout. On an explicit reset request, reset the current branch directly to `origin/main`; report that the branch name may remain different even though its tree is exact.
4. Do not mistake a repository parent for a checkout: inspect direct children, since a product directory may hold several sibling repositories.

## Explicit destructive reset

Only after the user explicitly requests destruction of local changes:

```bash
git fetch --prune origin main
git reset --hard origin/main
git clean -ffd
git submodule update --init --recursive
```

If `main` is held by another worktree, run the reset on the current branch instead of `git checkout main`; the checkout's tree and commit will match `origin/main` even if its local branch label differs. If submodule initialization is blocked by a stale, nonempty uninitialized submodule directory, inspect it first. Remove that directory only under the same explicit reset authorization, then retry recursive submodule initialization.

Verify with `git status --short --branch`, `git diff --check`, and compare `git rev-parse HEAD` against `git ls-remote origin refs/heads/main`.

## Build and run

```bash
cargo build --release
./target/release/centy-daemon
```

Use `./target/release/centy-daemon --addr 127.0.0.1:50061 --mcp-addr 127.0.0.1:50062` for non-default ports. Keep `--addr` off the MCP port (`50052`). Verify a launched daemon with an actual listener/API probe, rather than inferring it from a successful build.

## Upgrade the local daemon

On macOS, a launchd job keeps the daemon running (`KeepAlive`). Find its label and binary by inspection, then set `LABEL` and `BIN` from the output; do not assume them:

```bash
launchctl list | grep -i centy        # job label
command -v centy-daemon               # installed binary
```

1. Back up the installed binary: `cp "$BIN" "$BIN.bak"`.
2. Install the release build without overwriting a running file in place: `cp target/release/centy-daemon "$BIN.new" && mv "$BIN.new" "$BIN"`.
3. Find the old PID with `launchctl print "gui/$(id -u)/$LABEL" | grep pid`, then `kill <pid>`. launchd starts the new binary.
4. Verify with a session-less call. The endpoint needs no `Mcp-Session-Id` or initialize handshake. Use the MCP port the daemon was started with (default `50052`):

```bash
curl -sS http://127.0.0.1:50052/mcp \
  -H 'content-type: application/json' \
  -H 'accept: application/json, text/event-stream' \
  -d '{"jsonrpc":"2.0","id":1,"method":"tools/call","params":{"name":"centy_v1_CentyDaemon_GetDaemonInfo","arguments":{}}}'
```

Expect the new version in the result. If it shows the old version, the old PID still runs or launchd did not restart the job.

## Pitfalls

- A `ps` search can show no running daemon while a source checkout already exists; distinguish source discovery from runtime state.
- `git reset --hard` does not remove untracked files; `git clean -ffd` is necessary for a literal clean reset and is destructive.
- Do not claim that the daemon is running until a listener or API call confirms it.
- `tools/list` working does not prove `tools/call` works. Always smoke a real `tools/call`. A missing tonic `ready()` once made every call panic with `buffer full; poll_ready must be called first` while `tools/list` still worked.
- CI uses floating stable clippy. A red build in an untouched area is likely toolchain drift. Check the last run on `main` before blaming the PR.
