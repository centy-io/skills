---
name: centy-daemon-local-development
description: "Use when working on a local Centy daemon checkout."
version: 1.0.0
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

Use `./target/release/centy-daemon --addr 127.0.0.1:50052` for a non-default port. Verify a launched daemon with an actual listener/API probe, rather than inferring it from a successful build.

## Pitfalls

- A `ps` search can show no running daemon while a source checkout already exists; distinguish source discovery from runtime state.
- `git reset --hard` does not remove untracked files; `git clean -ffd` is necessary for a literal clean reset and is destructive.
- Do not claim that the daemon is running until a listener or API call confirms it.
