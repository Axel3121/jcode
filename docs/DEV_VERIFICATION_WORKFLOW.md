# Local Development Verification Workflow

This is the one obvious path for verifying a JCode change locally before
committing or pushing. It exists because raw `cargo build` / `cargo test`
silently uses the conservative `jobs = 4` fallback in `.cargo/config.toml`,
and running broad or overlapping checks in multiple uncoordinated rounds
wastes real time without adding evidence.

## Prefer the adaptive Cargo wrapper over raw Cargo

- `scripts/dev_cargo.sh` is the canonical entry point. It sizes
  `CARGO_BUILD_JOBS` from currently-available memory (clamped to CPU count),
  selects an available fast linker (mold, then lld, then system), enables
  sccache only when the build is non-incremental (sccache cannot cache
  incremental units), supports feature profiles via
  `JCODE_DEV_FEATURE_PROFILE`, and serializes compile-capable actions across
  concurrent jcode worktrees on the same machine so several agents don't all
  assume the full core count at once.
- `scripts/cargo_exec.sh` is a thin pass-through to `dev_cargo.sh` and is
  already used by the test/refactor helper scripts (`test_fast.sh`,
  `test_e2e.sh`, `refactor_phase1_verify.sh`, `agent_trace.sh`,
  `real_provider_smoke.sh`).
- In self-dev sessions, prefer `selfdev build` / `selfdev build-reload` /
  `selfdev test`, which route through this same wrapper.
- Use raw `cargo` directly only when the wrapper genuinely does not apply
  (for example a one-off `cargo doc` or `cargo tree` that isn't a
  build/test/check action the wrapper's job-sizing and gating matter for).

The `.cargo/config.toml` `jobs = 4` setting is an intentional memory-safe
fallback for *direct* Cargo invocations on a shared machine. It is not the
recommended high-throughput path, and this workflow doc does not change it.

## Verify in stages

Run the cheapest stage that still proves the change, then escalate only as
needed. Do not run every stage unconditionally.

1. **Focused** — tests for the changed behavior only.
   ```bash
   scripts/cargo_exec.sh test -p <crate> <test_filter>
   ```
   For fast library/binary iteration, `scripts/test_fast.sh` builds with a
   minimal feature profile and runs `--lib --bin jcode`.

2. **Affected** — the crate(s)/package(s)/cohort a change actually touches.
   ```bash
   scripts/cargo_exec.sh test -p <crate>
   scripts/cargo_exec.sh check -p <crate>
   ```
   Use the workspace's crate DAG (`Cargo.toml` `[workspace] members`) to scope
   this to what actually depends on the changed code, not the whole tree.

3. **Broad guardrail** — once the focused/affected evidence is stable, run
   most of what CI enforces, locally:
   ```bash
   scripts/check_guardrails.sh              # full guardrail set
   scripts/check_guardrails.sh --skip-slow   # skip check/clippy/machete
   ```
   This covers most of CI's "Quality Guardrails" and "Format" jobs
   (formatting, `cargo check`/`clippy --all-targets --all-features`,
   lockfile freshness, warning/size/panic/swallowed-error ratchets,
   dependency boundaries, wildcard re-export ratchet, desktop2 frame budget,
   onboarding invariants) so most failures are caught before pushing instead
   of on a shared CI run. It is not a byte-for-byte mirror: as of this
   writing it does not run CI's "Enforce Rust and TypeScript SDK surface
   parity" step (`cargo test -p jcode-sdk parity`). If a change touches the
   Rust or TypeScript SDK surface, also run that check directly:
   ```bash
   scripts/cargo_exec.sh test -p jcode-sdk parity -- --nocapture
   ```

4. **Full / exceptional** — a full workspace build, `--all-features`, a
   clean rebuild, or a vendored-dependency build. Only run these when the
   issue contract requires it or a concrete discovered risk justifies it
   (for example, a change to a feature-gated module that focused/affected
   testing cannot exercise). State the reason when you run one. CI's
   guardrail job already runs `--all-features`/`--all-targets` with the
   system dependencies it needs (e.g. `libfontconfig1-dev` for
   `jcode-desktop2`); don't speculatively reproduce that locally unless the
   change plausibly interacts with those feature-gated paths.

## Batch, don't repeat

Prefer one invocation that yields equivalent evidence over several
overlapping ones. For example, a single `cargo check -p <crate> --tests` can
stand in for separately re-running `build` and `check` on the same target
when nothing changed in between. Re-run a stage only when the code under
test changed since the last run of that stage.

## Timing evidence

`scripts/dev_cargo.sh` already records one JSON line per invocation
(timestamp, duration, action, profile, exit code) to
`~/.jcode/logs/rust-actions.jsonl` (override with `JCODE_RUST_ACTION_LOG_PATH`,
disable with `JCODE_RUST_ACTION_LOG=0`). For a materially build-heavy task,
report coarse timing for the major stages from this log rather than adding a
separate timing mechanism, e.g.:

```bash
tail -n 20 ~/.jcode/logs/rust-actions.jsonl
```

## Correctness always wins

Required correctness or security evidence overrides speed optimization.
Staging and batching are about avoiding *redundant* expensive work, not
about skipping mutation tests, guardrails, integration-path tests, or any
other evidence an issue's acceptance criteria actually require.
