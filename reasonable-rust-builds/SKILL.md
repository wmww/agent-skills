---
name: reasonable-rust-builds
description: Keep Rust/cargo target dirs from eating tens of GBs (per worktree, per test binary, stale artifact pileup) without slowing builds. Use when starting a Rust project, reorganizing a Rust repo's build/test/worktree workflow, or when target/ disk use is a problem.
---

# Reasonable Rust builds

Disk use is a product: **linked binaries × debuginfo × artifact "universes" × checkouts × time** (cargo never deletes anything). Attack each factor once, at setup; nothing here costs build speed.

**Everything recurring must live in the repo**, not in this skill: other machines and agents won't have it. Copy the scripts from `scripts/` next to this file into the project's `scripts/` and commit them, put config in `Cargo.toml`/`.cargo/config.toml`, and put the rules in the project's agent instructions (snippet at the end). Keep what you add short.

## Measure first

```sh
du -sh target/*/ target/*/{deps,build,incremental}      # where in target/
ls target/debug/deps | grep -v '\.' | wc -l             # linked executables (Unix)
du -ah target/debug/deps | sort -h | tail               # biggest artifacts
find ~ -name CACHEDIR.TAG -path '*target*' -exec dirname {} \; | xargs du -sh  # every target dir on the box
```

## 1. Debuginfo

```toml
[profile.dev]
debug = "line-tables-only"   # backtraces keep file:line; drops most DWARF
```

Full debuginfo is typically most of every binary's size (one real project: 630 MB → 245 MB per binary; ~30 binaries). If someone actually uses a debugger, use `debug = true` + `split-debuginfo = "unpacked"` instead (DWARF stays beside objects, not copied into each binary; smaller than full, larger than line tables). `[profile.dev.package."*"] opt-level = 2` is a good optional companion (faster tests, deps compiled once). Don't give `profile.test`/`bench` their own settings.

## 2. Fewer linked binaries

Every linked executable embeds the whole static dep tree, so count binaries, not crates:

- **Each `tests/*.rs` file is its own binary.** Use one integration-test binary per crate: `tests/suite/main.rs` declaring `mod foo; mod bar;` (files in `tests/suite/`). Shared helpers become a module. Filter with `cargo test --test suite foo::`. Watch for tests that relied on process isolation (global state, one-per-process inits); share those via a `OnceLock`.
- Prefer unit tests (`#[cfg(test)] mod tests` in `src/`): they land in the lib's single test binary.
- `test = false` on `[[bin]]`s and `[lib]`s that have no unit tests, `doctest = false` on libs with no doctests (edition 2024 merges doctests into one binary; older editions link one *per doctest*).
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
- dev-dependencies that enable extra features on a normal dependency split `cargo build` from `cargo test`. Enable those features in `[dependencies]` too.
- Toggling `CARGO_INCREMENTAL`, `RUSTFLAGS`, `RUSTC_WRAPPER`, `CARGO_TARGET_DIR`, or env that build scripts watch (`rerun-if-env-changed`). Put needed flags in `.cargo/config.toml`, never per command.
- Profile edits and toolchain bumps (pin with `rust-toolchain.toml`) — legitimate, sweep afterwards.
- rust-analyzer checking with other settings: set its `cargo.targetDir` to `true` (own dir).

Check: run the project's canonical commands twice in any order; the repeats must compile nothing. Compiling a registry crate means a second universe.

Also fix any build script whose `rerun-if-changed` names a missing path: it is always dirty, so the crate and everything above it rebuild every time (`cargo build -v` says "the file X is missing").

## 5. Worktrees: own target dir, seeded

Each checkout keeps its own `target/` inside it (deleted along with the worktree). Never point checkouts of one repo at a shared `CARGO_TARGET_DIR`/`build.target-dir`: cargo hashes path crates relative to the workspace root, so peers overwrite each other's artifacts, freshness falls back to mtimes, and a checkout can silently link another's code. (Don't hide `build-dir` under `~/.cargo` either; the bytes outlive the worktree.) sccache doesn't help: its keys include the path.

`scripts/seed-target.py` gives a new worktree a copy of the main checkout's build dir, minus the path crates: `deps/` hardlinked (rustc replaces artifacts, never edits them), `.fingerprint/`, `build/` copied with absolute paths rewritten (mtimes preserved, or every build-script crate goes stale). Measured on a 4.6 GiB V8/wgpu workspace: seed 3 s, first build 35 s vs 8m43 cold. It needs the worktree on the main checkout's filesystem. Run it from a tracked post-checkout hook (fires on `git worktree add`, run in the new worktree; the script no-ops elsewhere):

```sh
mkdir -p .githooks && printf '#!/bin/sh\npython3 scripts/seed-target.py\n' > .githooks/post-checkout
chmod +x .githooks/post-checkout && git config core.hooksPath .githooks  # per clone
```

(Or call it from whatever tooling creates worktrees.) Remove finished worktrees with `git worktree remove`; stale ones are whole target dirs.

## 6. Sweep stale artifacts

`scripts/sweep-target.py [--dry-run]` builds the canonical invocations with `--message-format=json`, collects every live unit hash, and deletes everything else under the profiles those builds touch (old universes, old dep versions, old toolchains; over-deleting only costs a rebuild). Pass the project's other build shapes with `--also "build -p app --release --no-default-features"`, or wrap it in a project script. Not `cargo-sweep`: mtime can't tell live from dead. Run after toolchain/profile/dependency changes or when `target/` grows; it frees real disk in a seeded worktree only for entries the main checkout no longer has.

## 7. CI

Build with the same invocation you test with (`cargo test --workspace --no-run`, then `cargo test --workspace`) so the cache holds one universe. Cache only deps: `Swatinem/rust-cache` already drops workspace crates and `incremental/`; otherwise delete those before saving. Key the cache on `Cargo.lock` + toolchain.

## Project instructions snippet

Adapt and add to the project's AGENTS.md/CLAUDE.md:

```markdown
- Build/test only with `cargo build --workspace` / `cargo test --workspace` (narrow with test filters or `--test`). No `-p` subsets, no per-command `RUSTFLAGS`/`CARGO_INCREMENTAL`/`CARGO_TARGET_DIR`: each new combination builds and keeps another full copy of the deps.
- New integration tests are modules of `tests/suite/`, not new `tests/*.rs` files (each file links its own binary).
- After cloning: `git config core.hooksPath .githooks` (seeds new worktrees' target dirs via scripts/seed-target.py; run it by hand in a worktree made otherwise). Remove finished worktrees.
- If target/ grows large, run `scripts/sweep-target.py`.
```
