---
name: reasonable-rust-builds
description: Keep Rust/cargo builds fast and target dirs bounded (per worktree, per test binary, stale artifact pileup), including for agents working in parallel worktrees. Use when starting a Rust project, reorganizing a Rust repo's build/test/worktree workflow, or when builds are slow or target/ disk use is a problem.
---

# Reasonable Rust builds

Disk is **linked binaries × debuginfo × artifact sets × checkouts**, and cargo never deletes anything from `target/`. Time is mostly **recompiling what didn't change**: a second artifact set, incremental off, a cold target. Set each factor once, write down what a build costs, and treat a 3x deviation as a bug.

This plugin ships `wt` (on PATH as `bin/wt` next to this file), the worktree tool below. Setting a project up: enable the plugin in the project's `.claude/settings.json` (`extraKnownMarketplaces` → `wmww-agent-skills`: github `wmww/agent-skills`; `enabledPlugins` → `"reasonable-rust-builds@wmww-agent-skills": true`), put config in `Cargo.toml` / `.cargo/config.toml`, and put the rules in the project's agent instructions (snippet at the end).

## Know your numbers

Measure once on a quiet machine and record in the project's notes: no-op build, one-line edit + `cargo test --workspace`, clean build, `wt claim` time, settled `target/` size. Without written numbers agents come to believe minutes are normal. Under load compare CPU seconds, not wall: `/usr/bin/time -f "%e s %U user" cargo ...`.

```sh
cargo build --workspace 2>&1 | grep -c Compiling          # after an edit: only the crate and its dependents
du -sh target/*/ target/debug/{deps,incremental,build}    # where the GB are
cargo build --timings                                      # target/cargo-timings/*.html: the critical path
cargo build -v 2>&1 | grep -i dirty                        # why a crate rebuilt
```

Signs of trouble: a build on a tree built minutes ago compiles crates you didn't touch; `target/` grows by GB per day; `--release`, `-p`, `--features` or env vars creeping into commands.

## 1. Debuginfo

```toml
[profile.dev]
debug = "line-tables-only"     # backtraces keep file:line; drops most DWARF

[profile.dev.package."*"]
opt-level = 2                  # deps compiled once, tests run fast
debug = false                  # dependency frames lose file:line; nothing else
```

```toml
# .cargo/config.toml: compress what is left, at link time (Linux)
[target.x86_64-unknown-linux-gnu]
rustflags = ["-C", "link-arg=-Wl,--compress-debug-sections=zlib"]
```

Full debuginfo is most of every binary (630 MB → 245 MB with line tables). zlib compression takes the rest to about 40%, keeps file:line, costs no measurable CPU. Not zstd (the backtrace symbolizer can't read it, frames lose locations). Not `split-debuginfo = "unpacked"`: it leaves a new `.dwo` set per rebuild that nothing deletes, GBs per target within days.

## 2. Fewer linked binaries

Every executable embeds the whole static dependency tree, so count binaries, not crates.

- Each `tests/*.rs` file is a binary. One integration-test binary per crate: `tests/it/main.rs` with `mod foo;` over `tests/it/*.rs`; filter with `cargo test --test it foo::`.
- Prefer unit tests (`#[cfg(test)] mod tests` in `src/`): they share the lib's one test binary.
- `test = false` / `doctest = false` on targets with none. Keep examples few: `cargo test` builds them all.
- One workspace, one `Cargo.lock`, one `target/` per checkout.

## 3. One artifact set

Every distinct combination of features, flags and env compiles a new copy of the affected crates and everything above them, kept forever. Causes:

- `-p` subsets: features unify over the selected packages only. Always `--workspace`; narrow with `--test`/name filters.
- A different `--features` per command. Pick one set for build, test and clippy alike.
- Dev-dependencies enabling features a normal dependency doesn't: enable them in `[dependencies]` too.
- Per-command `CARGO_INCREMENTAL`, `RUSTFLAGS`, `RUSTC_WRAPPER`, `CARGO_TARGET_DIR`. Settings go in config files, once.
- rust-analyzer with other settings: give it `cargo.targetDir = true` (its own subdir).

Check: run the canonical commands twice in any order; the repeats compile nothing. Fix any build script whose `rerun-if-changed` names a missing path (always dirty) or watches more than it reads. Don't stamp the git commit into a crate low in the stack: rerunning on `.git/HEAD` or `refs/heads` (shared by all worktrees) rebuilds everything above it on every checkout, and on every commit in any worktree. Stamp in the leaf binary.

## 4. Speed

- **Incremental stays on** (the dev default). Off, a one-line edit cost 57 s instead of 11 s.
- **One profile for all development**: dev. Release is a second copy of everything and its LTO relink takes minutes per edit. If tests are compute-heavy, `[profile.dev] opt-level = 1` for workspace crates (one project: tests 168 s → 32 s, edits ~8 s slower). Release only for timing checks, by hand.
- rustc's bundled lld (x86_64 Linux, rustc ≥ 1.90) makes linking cheap; mold buys nothing more. sccache misses path crates in another directory, so it doesn't help worktrees.

## 5. Worktrees: a few warm ones, reused

A fresh worktree's `target/` is a cold build (minutes). So keep a few long-lived worktrees at `.worktrees/wt<N>` inside the repo (gitignored), each with its own ordinary `target/`, and move tasks through them with `wt`:

- `wt claim <branch>` takes the lowest free one (a new one if all are busy), checks the branch out there and prints the path: under a second, and the first build recompiles only what differs from that worktree's last build, since git rewrites only changed files and the path never changes. It then runs the project's `scripts/wt-setup` if there is one (gitignored generated files a build needs: patched vendor crates, codegen).
- `wt release` prunes the target (below) and frees the worktree. It's the agent's very last step, after its last commit: released early, the next claim takes it while the first agent is still writing.
- `wt trim` removes free worktrees beyond two.
- Agents build, test and run checks in their worktree. The **main checkout is never built**; it only fast-forwards: `git rebase main` in the worktree, run the checks there, then `git merge --ff-only <branch>` in main.
- A coordinator claims a worktree per sub-agent and puts the path in its brief. Don't use the agent harness's own worktree isolation (Claude Code's `isolation: "worktree"`, `EnterWorktree`): each makes a fresh checkout with a cold target.

**Bounded disk: keep what's in use.** Cargo never deletes a unit (a crate built with particular settings; its files all carry the unit's hash), so every dependency, toolchain or profile change leaves the old units behind. But on every build, even a no-op, cargo reads the fingerprint of each unit in the build graph, which updates its access time. `wt release` uses that: it runs `cargo build --workspace --all-targets` (a no-op on a just-tested tree, which marks the standard set), deletes every unit whose fingerprint wasn't read or written since the previous release (`.fingerprint/<name>-<hash>`, `deps/*-<hash>*`, `build/<name>-<hash>`), deletes every `incremental/` dir not written since then, and resets access times so the next read registers under `relatime`. So a target holds the standard set plus whatever the last task built. It's driven by builds, not the clock, so an idle worktree loses nothing. Mistakes only cost rebuilds: a pruned live unit compiles again; a pruned live incremental dir costs one non-incremental compile of that crate (seconds). Something used only occasionally (a release timing check) is rebuilt when next needed. On a `noatime` filesystem it skips pruning and says so.

Don't prune by age (`cargo-sweep --time`: it can't tell live from dead in a young target) or per file (on a no-op build cargo reads fingerprints, not rlibs, so file atimes delete live rlibs), and don't define live as the outputs of a fixed list of commands (the marking build only adds to what the task used): one sweeper that did kept clippy's incremental dirs and deleted the build's, and every build compiled from scratch at 16x the CPU for a day. Deleting a whole target is always safe.

Rejected, measured:
- **One `CARGO_TARGET_DIR` shared by checkouts**: cargo keys path crates without the path and judges freshness by mtime, so a checkout can link another checkout's code and pass as fresh.
- **Seeding** a new worktree's target from another (hardlinks, copied fingerprints, rewritten build-script paths): fragile, and silently cold when the source was stale or mid-build.
- **One build checkout behind a lock**: correct, but the lock must cover test runs, so all agents serialize.

## 6. CI

Build with the invocation you test with (`cargo test --workspace --no-run`, then `cargo test --workspace`). Cache dependencies only, keyed on everything that changes their artifacts: `Cargo.lock`, `Cargo.toml` (profiles), `.cargo/config.toml`, toolchain.

## Project instructions snippet

```markdown
- `wt` comes from the reasonable-rust-builds plugin (github wmww/agent-skills, enabled in `.claude/settings.json`). If it's not on PATH, stop and ask the user to install it; don't improvise worktrees.
- Work in a task worktree: `wt claim <branch>` prints it (`.worktrees/wtN`, warm `target/` kept between tasks); `wt release` only as your very last step, after your last commit. Never build in the main checkout; main only fast-forwards (`git rebase main` and the checks in the worktree, then `git merge --ff-only <branch>` in main).
- Coordinators: claim a worktree per sub-agent and give it the path; never use agent-tool worktree isolation (cold targets). `wt trim` when the parallel work winds down.
- Build and test only with `cargo test --workspace` (narrow with `--test it <module>::` and name filters) and `cargo clippy --workspace --all-targets`. No `-p`, no other `--features`, no per-command `RUSTFLAGS`/`CARGO_INCREMENTAL`/`CARGO_TARGET_DIR`: each combination builds and keeps another copy of the tree. `--release` only for timing checks.
- Expected costs are in notes/<build note>; if yours are 3x those at low load, stop and find out why.
```
