---
name: reasonable-rust-builds
description: Keep Rust/cargo builds fast and target dirs from eating tens of GBs (per worktree, per test binary, stale artifact pileup). Use when starting a Rust project, reorganizing a Rust repo's build/test/worktree workflow, or when builds are slow or target/ disk use is a problem.
---

# Reasonable Rust builds

Disk is **linked binaries × debuginfo × artifact sets × checkouts**, and cargo never deletes anything. Time is mostly **recompiling what didn't change**: a second artifact set, incremental off, a cold target, a sweep that threw the caches away. Set each factor once, write down what a build costs, and treat a 3x deviation as a bug.

Everything recurring lives in the repo, not in this skill: copy `scripts/wt` next to this file into the project's `scripts/` or `tools/`, put config in `Cargo.toml`, and put the rules in the project's agent instructions (snippet at the end). Keep it short.

## Know your numbers

Measure on a quiet machine and record the results in the project's notes, because agents otherwise come to believe that minutes are normal. One 21-crate workspace on a 16-thread laptop: no-op build 0.4 s; edit a mid-stack crate then `cargo test --workspace` 8 s build + 31 s tests; edit the bottom crate 10-16 s; clean build 82 s, plus 53 s for test binaries, plus 17 s for a first clippy; a 30-seed simulator sweep 2.2 min. Load multiplies every step 2-5x, so compare CPU seconds, not wall: `/usr/bin/time -f "%e s %U user" cargo ...`, and count `Compiling` lines. The same project had been living with 4-7 minute builds and 40-minute test runs for a day before anyone measured; every one of them had a cause below.

```sh
cargo build --workspace 2>&1 | grep -c Compiling          # after an edit: only the crate and its dependents
du -sh target/*/ target/debug/{deps,incremental,build}    # where the GB are
ls target/debug/deps | grep -v '\.' | wc -l                # linked executables (Unix)
cargo build --timings                                      # target/cargo-timings/*.html: the critical path
cargo build -v 2>&1 | grep -i dirty                        # why a crate rebuilt
```

Signs of trouble: a build on a tree built minutes ago compiles crates you didn't touch; `target/` grows by GB per day; `--release`, `-p`, `--jobs` or `--features` creeping into commands; a timing test running in the dev profile.

## 1. Debuginfo

```toml
[profile.dev]
debug = "line-tables-only"     # backtraces keep file:line; drops most DWARF

[profile.dev.package."*"]
opt-level = 2                  # deps compiled once, tests run fast
debug = false                  # dependency frames lose file:line; nothing else
```

```toml
# .cargo/config.toml: compress what is left, at link time (Linux; lld and GNU ld both take it)
[target.x86_64-unknown-linux-gnu]
rustflags = ["-C", "link-arg=-Wl,--compress-debug-sections=zlib"]
```

Full debuginfo is most of every binary (one project: 630 MB → 245 MB per binary). Even with line tables only, debug sections were still 80% of the test binaries in a 21-crate workspace (1.7 GB of 2.1 GB; the string table and the compile-unit scaffolding around the line tables, mostly the project's own crates). Measured on that workspace, 38 test binaries:

| | binaries | backtrace frames |
|---|---|---|
| line tables, uncompressed | 1.7 GB | file and line |
| line tables, zlib-compressed sections | 0.65 GB | file and line |
| line tables, zstd-compressed sections | 0.57 GB | function names only: the symbolizer does not read zstd |
| `debug = false` + `strip = "debuginfo"` | 0.34 GB | function names only; panic messages keep their own location |

Take zlib: 40% of the size, nothing lost, compile CPU unchanged. Strip only if the last 0.3 GB matters more than locations on backtrace frames. Don't add `split-debuginfo = "unpacked"` to a long-lived target: it halves the binaries once but leaves a new `.dwo` set in `deps/` on every rebuild that nothing deletes, 3-5 GB per target after a few days of edits. No separate `profile.test`/`bench` settings.

## 2. Fewer linked binaries

Every executable embeds the whole static dependency tree, so count binaries, not crates.

- Each `tests/*.rs` file is a binary. Use one integration-test binary per crate: `tests/it/main.rs` with `mod foo; mod bar;` over `tests/it/*.rs`; filter with `cargo test --test it foo::`. Tests that relied on process isolation share state through a `OnceLock`.
- Prefer unit tests (`#[cfg(test)] mod tests` in `src/`): they land in the lib's one test binary.
- `test = false` on bins and libs with no unit tests, `doctest = false` on libs with no doctests, and a policy script that fails when such a target gains one (otherwise they are skipped silently). Edition 2024 merges doctests into one binary; older editions link one per doctest.
- `cargo test` builds every example: keep them few or make them tests.
- One workspace, one `Cargo.lock`, one `target/`, never sibling standalone crates.

Linking itself is cheap with rustc's bundled lld (≥ 1.90 on x86_64 Linux): a mid-stack edit relinking ten test binaries fits inside an 8 s build step.

Optional, for a heavy stack where many binaries each embed ≳100 MB: the bevy_dylib pattern, a `crate-type = ["dylib"]` crate re-exporting the heavy crates that dev binaries depend on (`use foo_dylib as _;`), excluded from the workspace, shipped with `--no-default-features`. Its unhashed output path means every feature set it is built with fights over one file: enable on its deps exactly what workspace resolution enables.

## 3. One artifact set

Every distinct combination of features, flags and env compiles a new copy of the affected crates and everything above them, kept forever. Causes: `-p` subsets (features unify over the selected packages only: always `--workspace`, narrow with `--test`/name filters); a different feature set per command (pick one for build, test and clippy alike; one extra set for an opt-in area such as networking is the most you want); dev-dependencies enabling features a normal dependency doesn't (enable them in `[dependencies]` too); per-command `CARGO_INCREMENTAL`, `RUSTFLAGS`, `RUSTC_WRAPPER`, `CARGO_TARGET_DIR`; profile edits and toolchain bumps (legitimate; see §6); rust-analyzer with other settings (give it `cargo.targetDir = true`).

Check: run the project's canonical commands twice in any order; the repeats compile nothing (clippy's `-- -D warnings` is part of its fingerprint, so count `Checking` too). Fix any build script whose `rerun-if-changed` names a missing path (always dirty) or that watches more than it reads (one hashed every workspace crate's `src`, so a leaf edit rebuilt a core crate and everything above it).

## 4. Speed

- **Incremental stays on** for dev (the default). Off, a one-line edit cost 57 s instead of 11 s. It is a few GB per checkout and does not grow with edits (±0.1 GB per cycle).
- **One profile for all development.** Release is a second copy of everything, and its thin-LTO relink of every test binary after a one-line edit takes minutes where dev takes seconds. If tests are compute-heavy, `[profile.dev] opt-level = 1` for workspace crates: one project's test run went 168 s → 32 s and its simulator sweep ran within 1.3x of release, for edits about 8 s slower. Release is for timing checks only, run on purpose, in the task's worktree; keep its `target/release` (a few hundred MB) so the next check is incremental. Timing-budget tests never run in dev: they take minutes there and prove nothing.
- `cargo build --timings` shows the crate everything waits on; that is the place to split.

## 5. Worktrees: a few warm ones, inside the repo

A fresh worktree's `target/` is cold (a full build, 2-4 minutes), and seeding it from another checkout is fragile. Keep a few long-lived worktrees at `.worktrees/wt<N>` inside the repo (gitignored; tools that scan sources must use explicit roots), each with its own ordinary in-tree `target/`, and move tasks through them: `scripts/wt claim <branch>` picks a free one (or makes a new one when all are busy) and checks the branch out there, `scripts/wt release` detaches HEAD when the branch is committed, `scripts/wt trim` removes free ones beyond the two that stay warm. git rewrites only the files that differ between branches and the path never changes, so a task's first build costs exactly a branch switch, incrementally. Agents work, test and gate there; the main checkout is never built: run the checks in the worktree after `git rebase main`, then `git merge --ff-only` on main. Disk is one target per warm worktree, about 6 GB each, flat.

Rejected, measured:
- **Seeding** a new worktree's target from another's (hardlinks, copied fingerprints): fast when fresh, but a changed crate compiles in full, the hook skipped silently when the source was mid-build, and a stale source rebuilt everything.
- **One build checkout behind a lock**, sources synced in by content with fresh mtimes on changed files: as fast as a warm worktree and one target on the box, but the lock must cover test runs (tests find fixtures through the compile-time manifest dir), so every job serializes. Worth it only when disk, not parallelism, is the constraint.
- **A shared `CARGO_TARGET_DIR`** across checkouts: cargo keys path crates without the path and judges freshness by mtime; a checkout whose edit is older than another's build reads as fresh with the other's code inside (verified with `nm`), and incremental is discarded on every alternation.
- **sccache**: misses path crates in another directory (0 hits).

## 6. When target/ grows

With §1 and §3 in place nothing accumulates, so there is nothing to sweep. After a toolchain, profile or dependency change the old artifacts are orphaned: delete `target/` in that worktree and take one cold build. Don't run an artifact sweeper automatically. One that ran after every merge had a counting bug (it treated a crate's build and clippy-check units as one, kept clippy's incremental dirs and deleted the build's), and every build after it compiled from scratch at 16x the CPU for a day before anyone noticed. If you must sweep, dry-run it and check that the next incremental build is still incremental.

## 7. CI

Build with the invocation you test with (`cargo test --workspace --no-run`, then `cargo test --workspace`) so the cache holds one artifact set. Cache only dependencies, keyed on `Cargo.lock` plus toolchain.

## Project instructions snippet

```markdown
- Work in a task worktree: `scripts/wt claim <branch>` prints it (`.worktrees/wtN`, warm `target/` kept between tasks); `scripts/wt release` when done. Never build in the main checkout, never share a target dir between checkouts.
- Build and test only with `cargo test --workspace` (narrow with `--test it <module>::` and name filters) and `cargo clippy --workspace --all-targets -- -D warnings`, dev profile; `--release` only for timing checks, by hand, in your worktree. No `-p`, no other `--features`, no per-command `RUSTFLAGS`/`CARGO_INCREMENTAL`/`CARGO_TARGET_DIR`: each combination builds and keeps another copy of the tree.
- New integration tests go in the crate's one test binary (`tests/it/`), not new `tests/*.rs` files.
- Expected costs are in notes/build.md; if yours are 3x those at low load, stop and find out why.
```
