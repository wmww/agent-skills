---
name: reasonable-rust-builds
description: Keep Rust/cargo builds fast and target dirs from eating tens of GBs (per worktree, per test binary, stale artifact pileup). Use when starting a Rust project, reorganizing a Rust repo's build/test/worktree workflow, or when builds are slow or target/ disk use is a problem.
---

# Reasonable Rust builds

Disk use is a product: **linked binaries × debuginfo × artifact "universes" × checkouts × time** (cargo never deletes anything). Build time is mostly **recompiling what didn't change** (incremental off, a second profile, a cold target dir per new worktree). Attack each factor once, at setup.

**Everything recurring must live in the repo**, not in this skill: other machines and agents won't have it. Copy the scripts from `scripts/` next to this file into the project's `scripts/` and commit them, put config in `Cargo.toml`/`.cargo/config.toml`, and put the rules in the project's agent instructions (snippet at the end). Keep what you add short.

## Measure first

```sh
du -sh target/*/ target/*/{deps,build,incremental}      # where in target/
du -sh main/target wt/target                            # one call counts hardlinks once: wt's unique bytes
ls target/debug/deps | grep -v '\.' | wc -l             # linked executables (Unix)
du -ah target/debug/deps | sort -h | tail               # biggest artifacts
readelf -SW BIN | sort -k6                              # .debug_* share of a binary (hex sizes)
cargo build --timings                                   # target/cargo-timings/*.html: per-unit times
find ~ -name CACHEDIR.TAG -path '*target*' -exec dirname {} \; | xargs du -sh  # every target dir on the box
```

## 1. Debuginfo

```toml
[profile.dev]
debug = "line-tables-only"     # backtraces keep file:line; drops most DWARF
split-debuginfo = "unpacked"   # Linux/macOS: the rest stays beside the objects, not in every binary

[profile.dev.package."*"]
opt-level = 2                  # optional: faster tests, deps compiled once
debug = false                  # dependency frames lose file:line; nothing else
```

Full debuginfo is typically most of every binary's size (one real project: 630 MB → 245 MB per binary; ~30 binaries). Even line tables were half of each test binary in another (21 crates, 38 executables): deps' `debug = false` then `unpacked` took them from 1.8 GB to 0.9 GB in all, and one-line-edit relinks got faster; backtraces still show file:line (the line tables stay in the binary). If someone actually uses a debugger, use `debug = true` with `unpacked`. Never `strip` test binaries: that drops the line tables. Don't give `profile.test`/`bench` their own settings.

## 2. Fewer linked binaries

Every linked executable embeds the whole static dep tree, so count binaries, not crates:

- **Each `tests/*.rs` file is its own binary.** Use one integration-test binary per crate: `tests/suite/main.rs` declaring `mod foo; mod bar;` (files in `tests/suite/`). Shared helpers become a module. Filter with `cargo test --test suite foo::`. Watch for tests that relied on process isolation (global state, one-per-process inits); share those via a `OnceLock`.
- Prefer unit tests (`#[cfg(test)] mod tests` in `src/`): they land in the lib's single test binary.
- `test = false` on `[[bin]]`s and `[lib]`s that have no unit tests (then guard it: a check in CI or a policy script that fails when such a target gains a `#[test]`, or they are skipped silently), `doctest = false` on libs with no doctests (edition 2024 merges doctests into one binary; older editions link one *per doctest*: slow, though in a temp dir, not `target/`).
- `cargo test` also builds every example. Keep examples few or make them tests.
- One workspace with one `Cargo.lock` and one `target/`, never sibling standalone crates each with their own.

## 3. One shared library for a heavy stack (optional)

Only when many binaries (tests, tools, the app) each embed a large stack (≳100 MB each after step 1). The bevy_dylib pattern turns them into ~MB stubs over one `.so`:

- A crate `foo-dylib` with `crate-type = ["dylib"]` whose `lib.rs` is `pub use` of the heavy workspace crates. Dev binaries depend on it and write `use foo_dylib as _;`; the app bin via a default `dynamic = ["dep:foo-dylib"]` feature. **Ship with `--no-default-features`** (static).
- **Exclude it from the workspace** (`exclude = [...]`, no `.workspace = true` inheritance in its manifest). Built as a requested root it gets static libstd and consumers fail with "cannot satisfy dependencies so `std` only shows up once"; as a dependency cargo passes `-C prefer-dynamic`.
- Its output path is unhashed, so every feature set it's built with fights over one file. Enable on its deps exactly the features workspace-wide resolution enables (including ones dev-deps add). Symptom of a mismatch: the `.so` relinks when alternating commands.
- `cargo run`/`cargo test` find the `.so` themselves; to run `target/debug/app` directly emit rpaths from `build.rs`: `$ORIGIN`, `$ORIGIN/deps` and `rustc --print target-libdir` (libstd.so) via `cargo::rustc-link-arg-bins=-Wl,-rpath,...` (`@loader_path` on macOS, nothing on Windows).
- Lib *unit*-test binaries stay static (they compile the crate itself under `cfg(test)`).

## 4. One artifact universe

Every distinct combination of features/flags/env compiles a *new copy* of the affected crates and everything above them, kept forever (one agent session minted ~7 copies, 10 GB, in an afternoon). Causes:

- `-p` subsets: features unify over the selected packages only. Stick to `--workspace` and narrow with test filters / `--test <name>`. (`resolver.feature-unification = "workspace"` fixes this but is nightly-only as of cargo 1.98; use it once stable.)
- Different features per command (`cargo clippy --all-features` beside plain `cargo test`, `--features a` then `--features a,b`). Pick one dev feature set and pass it to build, test and clippy alike.
- dev-dependencies that enable extra features on a normal dependency split `cargo build` from `cargo test`. Enable those features in `[dependencies]` too. A crate's dev-dependency on *itself* with a test-only feature is the same split: make the feature always on, or gate it at runtime.
- Toggling `CARGO_INCREMENTAL`, `RUSTFLAGS`, `RUSTC_WRAPPER`, `CARGO_TARGET_DIR`, or env that build scripts watch (`rerun-if-env-changed`). Put needed flags in `.cargo/config.toml`, never per command.
- Profile edits and toolchain bumps (pin with `rust-toolchain.toml`) — legitimate, sweep afterwards.
- rust-analyzer checking with other settings: set its `cargo.targetDir` to `true` (own dir).

Check: run the project's canonical commands twice in any order; the repeats must compile nothing (count `Checking` lines too: clippy's `-- -D warnings` is part of its fingerprint). Compiling a registry crate means a second universe. In a virtual workspace, `cargo run --bin X` without `-p` resolves features over the whole workspace and reuses the test build's artifacts; `cargo run -p X` starts a new universe.

Also fix any build script whose `rerun-if-changed` names a missing path: it is always dirty, so the crate and everything above it rebuild every time (`cargo build -v` says "the file X is missing"). Likewise one that watches more than it reads (one hashed every workspace crate's `src`, so an edit to a leaf binary rebuilt a core crate and everything above it: 2 s became minutes). `cargo build -v` prints why each crate is dirty.

## 5. Speed: incremental, one profile, linker

- **Keep `incremental` on for dev** (the default). Turning it off for disk (one project did) made a one-line edit + test build 57 s instead of 11-18 s. rustc keeps one session per unit and drops the old one, so it doesn't grow with edits (measured: ±60 MB per cycle); what piles up is dirs of units that no longer exist, which `sweep-target.py` bounds (§7). It is a few GB per checkout: budget for it.
- **One profile for all development work.** A release set is a second copy of everything, and its non-incremental thin-LTO rebuild after a one-line edit took 10 min where dev took 15 s. If tests are compute-heavy, put workspace crates at `[profile.dev] opt-level = 1` instead: one project's tier 1 went 168 s → 32 s, its simulator sweep ran within 1.3× of release, its tier-2 tests within ~1.6×, for edits ~8 s slower. Keep release for timing benches, run on purpose. Costs: worse stepping in a debugger, timing assertions calibrated in release fail.
- **Linker**: rustc ≥ 1.90 on x86_64 Linux already links with its bundled lld. mold halved link CPU (12.7 → 7.3 s user per edit) but not wall time on a loaded box; not worth a required system package. On older toolchains or other targets, lld/mold are a big win.
- `cargo build --timings` shows the critical path; a crate everything waits on is the place to split.

## 6. Worktrees: reusable slots, each with its own warm target dir

Don't give each task a fresh worktree: its `target/` starts cold (a full build), and making it warm by copying another checkout's is fragile. Keep a few long-lived worktrees ("slots", `../<repo>-wt/slot1..N`), each with its own ordinary in-tree `target/`, and move tasks through them: claim a free slot, check the task's branch out there, release it when done (HEAD detached so the branch can be merged or deleted elsewhere; files and `target/` stay). git rewrites only files that differ between branches (they get a new mtime) and the slot's path never changes (rustc keeps incremental state per path), so a task's first build costs exactly a branch switch: the crates whose sources differ, incrementally. `scripts/slot claim <branch> [<base>] | release [<slot>] | list` does this (claim = atomic create of a marker in the common git dir; a new slot is made when all are busy; release refuses a dirty slot). Measured (21-crate workspace, loaded 4-core box, one-line edit in a core crate with 4 crates above it): first build of a task 16-27 s; a fresh seeded worktree 197-226 s for the same edit (core crate in full) and 840 s when the seed source was stale; a cold slot 17 min, once. Disk: one target dir per slot (3-4.5 GB each here, after sweeps). Hazards: a claim left by a dead task stays until released by hand (`list` shows branch and time); ignored scratch files in a slot carry over to the next task; a slot's first-ever build is cold.

Rejected, measured:
- **Seeding** a new worktree's `target/` from the main checkout's (hardlinked `deps/`, copied fingerprints and build-script outputs with absolute paths rewritten, mtimes of identical sources copied): new worktree 3 s and 125 MB when it works, but a changed crate compiles in full (no incremental across paths), a post-checkout hook running it skipped silently when the source checkout was mid-build (a file vanished under the hardlink), and a seed from a stale source target rebuilt everything. In git history of this skill (`scripts/seed-target.py`).
- **One shared build checkout behind a queue**: a wrapper takes a flock on a plain build directory, syncs the calling worktree's tracked and untracked-non-ignored files into it by content (changed files get the current mtime, identical ones keep theirs, so freshness stays correct across trees), runs the command there and exits with its status; pass the lock fd to the command so the lock lives until every child exits, even if the wrapper is killed, and a killed sync is repaired by the next. Builds cost the same as in a slot (same mechanism) and there is one target dir on the box, but the lock must cover test runs too (tests find fixtures through the compile-time manifest dir, inside the build checkout), so every job serializes: measured holds 79 s for a tier-1 run, 128 s for a gate, 573 s for a simulator sweep, each blocking every other agent; flock isn't FIFO (starvation under contention); outputs written into the tree and ignored scratch files stay on the other side. Worth it only when disk, not parallelism, is the constraint.
- **A shared `CARGO_TARGET_DIR`/`build.target-dir`/`build.build-dir`** across checkouts: cargo hashes path crates relative to the workspace root, so peers overwrite each other's artifacts and freshness falls back to mtimes: a checkout whose edit is older than another's build silently links the other's code (`-Zchecksum-freshness` would fix that, nightly only as of cargo 1.98); and incremental is discarded on every alternation (path differs).
- **sccache**: misses path crates in another directory (absolute paths are in its hash: 0 hits).

## 7. Sweep stale artifacts

`scripts/sweep-target.py [--dry-run]` builds the canonical invocations (`build --workspace --all-targets`, plus the same with `--release` when `target/release` exists: this covers `cargo test --release` binaries) with `--message-format=json`, collects every live unit hash, and deletes everything else under the profiles those builds touch (old universes, old dep versions, old toolchains; over-deleting only costs a rebuild). Pass the project's other build shapes with `--also "build -p app --release --no-default-features"` or `--also "clippy --workspace --all-targets -- -D warnings"` (args after `--` are kept after `--`), `--no-defaults` to drop the built-in shapes, or wrap it in a project script. `incremental/` dirs are named by crate and a hash of their own, so it keeps, per crate name, as many of the newest as there are live units of that name. Not `cargo-sweep`: mtime can't tell live from dead. Run after toolchain/profile/dependency changes or when `target/` grows, in each slot.

## 8. CI

Build with the same invocation you test with (`cargo test --workspace --no-run`, then `cargo test --workspace`) so the cache holds one universe. Cache only deps: `Swatinem/rust-cache` already drops workspace crates and `incremental/`; otherwise delete those before saving. Key the cache on `Cargo.lock` + toolchain.

## Project instructions snippet

Adapt and add to the project's AGENTS.md/CLAUDE.md:

```markdown
- Build/test only with `cargo build --workspace` / `cargo test --workspace` (narrow with test filters or `--test`), dev profile; `--release` only for timing benches. No `-p` subsets, no per-command `RUSTFLAGS`/`CARGO_INCREMENTAL`/`CARGO_TARGET_DIR`: each new combination builds and keeps another full copy of the deps.
- New integration tests are modules of `tests/suite/`, not new `tests/*.rs` files (each file links its own binary).
- Work in a worktree slot: `scripts/slot claim <branch>` prints where; `scripts/slot release` when done (commit first). Don't create other worktrees or share target dirs between checkouts.
- If target/ grows large, run `scripts/sweep-target.py`.
```
