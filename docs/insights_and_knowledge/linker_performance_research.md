# Research: Faster Linking in Buck2 (Remote Execution + Incremental + Toolchain)

This note focuses on practical, *deployable* options for reducing link latency in Buck2 builds.

---

## Executive summary (highest ROI)

If you want improvements quickly, do these first:

1. **Run heavy links on remote workers** with `--prefer-remote` (or `--remote-only` for benchmarking).
2. **Switch linker implementation** to `lld` first, then evaluate `mold` for Linux/ELF workloads.
3. **Use ThinLTO cache** for optimized iterative builds.
4. **Use Buck2 incremental actions** only for workflows where output/state can be safely updated in place (for example, MSVC incremental-link style flows).

---

## 1) Execute linker actions on remote executors

### 1.1 What Buck2 supports

Buck2 supports RE backends via the Bazel Remote Execution API, configured in `.buckconfig` under `[buck2_re_client]`, plus an execution platform (`CommandExecutorConfig`) that enables remote execution behavior.

For command-line execution strategy, Buck2 build flags include:

- `--local-only`
- `--remote-only`
- `--prefer-local`
- `--prefer-remote`

`--prefer-remote` specifically avoids local-vs-remote racing and prefers RE-capable actions on remote workers.

### 1.2 Recommended rollout pattern for linking

**Phase 0 (measurement):**
- Compare the same target set using `--local-only`, `--remote-only`, and `--prefer-remote`.
- Track p50/p95 link-action duration, RE queue wait, and end-to-end wall time.

**Phase 1 (production default):**
- Use `--prefer-remote` for day-to-day builds so large links run where cores/memory are better.
- Keep local fallback available while hardening toolchain parity.

**Phase 2 (stability hardening):**
- Keep linker toolchains pinned in the remote image.
- Ensure link actions are hermetic (no undeclared host inputs).

### 1.3 Benchmarking caveat

When benchmarking remote link performance, avoid false “speedups” from remote cache reads/writes unless that is part of your intended workflow. Buck2 also documents that for strict benchmarking, a random isolation dir is often preferred even when using `--no-remote-cache`.

---

## 2) Incremental linking in Buck2

### 2.1 Buck2 mechanism

Buck2 supports incremental user-defined actions via `ctx.actions.run` when you use:

- `no_outputs_cleanup = True`
- `metadata_env_var`
- `metadata_path`

This allows the action to:

1. read prior outputs/state,
2. compare prior-vs-current input digests,
3. update only changed pieces.

### 2.2 Safe pattern for incremental action state

A robust pattern is:

1. read previous incremental state;
2. remove stale state before mutating outputs;
3. update outputs;
4. write fresh state only after successful update.

This avoids reusing corrupted incremental metadata after a failed run.

### 2.3 Where incremental linking is practical

- **Most practical today on Windows/MSVC** (`/INCREMENTAL` + `.ilk` reuse).
- Microsoft documents cases that force a full link (for example `/OPT:REF`, `/OPT:ICF`, `/ORDER`, or missing/unwritable `.ilk`).

**Guideline:** use incremental link profiles for developer loops; keep full non-incremental links for release-quality outputs.

---

## 3) Linker choice: `lld` vs `mold`

### 3.1 `lld`

- Mature, cross-format linker (ELF, PE/COFF, Mach-O, Wasm).
- Usually lowest-risk first migration from GNU ld/gold in mixed environments.

### 3.2 `mold`

- Designed specifically for high-performance ELF linking.
- Strong option for Linux-heavy C/C++ monorepos once compatibility checks pass.

### 3.3 Migration plan

1. A/B test baseline linker vs `lld`.
2. A/B test `lld` vs `mold` on Linux targets.
3. Compare link wall time, peak RSS, and binary compatibility constraints (debug/sanitizer/symbol rules).
4. Pin linker version in remote container/toolchain definition.

---

## 4) ThinLTO cache for optimized iterative builds

ThinLTO supports incremental rebuild acceleration via cache directories configured through linker flags:

- `ld.lld`: `--thinlto-cache-dir=<dir>`
- `lld-link`: `/lldltocache:<dir>`

Use cache pruning policy suitable for CI/RE worker disk constraints.

---

## 5) Buck2 knobs that reduce link *end-to-end* latency

Even when link execution stays remote, these can reduce total build time:

1. **Deferred materialization**: avoid downloading intermediates unless local actions actually need them.
2. **`sqlite_materializer_state = true`**: preserve materializer state across restarts and avoid unnecessary re-downloads.
3. **`defer_write_actions = true`**: keep non-critical writes off the critical path in suitable setups.

---

## 6) Concrete playbook (30-day)

### Week 1: Baseline and visibility

- Select top 10 slowest link-heavy targets.
- Run each target 3x under:
  - `--local-only`
  - `--remote-only`
  - `--prefer-remote`
- Capture p50/p95 action time and total wall clock.

### Week 2: Linker swap

- Move those targets to `lld` in toolchain config.
- Re-run same matrix and verify artifact compatibility.

### Week 3: Incremental dev profile

- Windows targets: enable `/INCREMENTAL` for dev profile only.
- Custom Buck2 link-like actions: implement metadata/state incremental pattern.

### Week 4: Optimized loop

- Enable ThinLTO cache for optimized dev builds.
- Add cache size/prune policy.

Success criterion: >20% p95 reduction on selected large-link targets without release artifact regressions.

---

## 7) Sources

- Buck2 Incremental Actions: https://buck2.build/docs/rule_authors/incremental_actions/
- Buck2 Remote Execution: https://buck2.build/docs/users/remote_execution/
- Buck2 build command docs: https://buck2.build/docs/users/commands/build/
- Buck2 deferred materialization: https://buck2.build/docs/users/advanced/deferred_materialization/
- Buck2 `CommandExecutorConfig` API: https://buck2.build/docs/api/build/CommandExecutorConfig/
- Buck2 build option implementation (`--remote-only`, `--prefer-remote`, cache notes):
  - `app/buck2_client_ctx/src/common/build.rs`
- LLVM lld docs: https://lld.llvm.org/
- Clang ThinLTO docs: https://clang.llvm.org/docs/ThinLTO.html
- mold project: https://github.com/rui314/mold
- MSVC `/INCREMENTAL` docs: https://learn.microsoft.com/en-us/cpp/build/reference/incremental-link-incrementally?view=msvc-170
