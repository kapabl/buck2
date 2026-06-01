---
id: accelerating_linking
title: Accelerating Linking (including Remote Execution)
---

For very large C/C++ or mixed-language builds, link steps are often the primary
critical-path bottleneck. Buck2 has a few concrete levers that can help,
including support for incremental remote outputs in `ctx.actions.run` and
prelude-level controls for where and how links execute.

## 1) Run links where they are fastest

Buck2 prelude exposes `link_execution_preference` for C/C++ rules. You can set
it to:

- `any`
- `full_hybrid`
- `local`
- `local_only`
- `remote`

Use this to bias large link actions to remote executors, or to full-hybrid mode
if your executor supports racing local + remote and taking the winner.

See `link_execution_preference_attr()` and its mapping into action execution
attributes (`prefer_remote`, `prefer_local`, `full_hybrid`, etc.).

## 2) Enable distributed ThinLTO when possible

If your toolchain supports it, prelude C++ link flow can dispatch distributed
ThinLTO instead of a monolithic local link.

Relevant knobs:

- Target attr: `enable_distributed_thinlto`
- Toolchain capability: `supports_distributed_thinlto`
- Toolchain mode requirement: `lto_mode == thin`

When enabled and compatible, the linker flow routes through specialized
distributed link paths (GNU or Darwin implementations).

## 3) Use incremental remote outputs for custom linker-like actions

Buck2 `ctx.actions.run` supports `incremental_remote_outputs` for actions that
can reuse prior outputs remotely.

Important constraints enforced by Buck2:

- `incremental_remote_outputs = True` requires `no_outputs_cleanup = True`.
- Actions using incremental remote outputs cannot use content-based output
  paths.
- At execution time, incremental behavior only proceeds if *all* prior outputs
  are present/materialized; otherwise Buck2 falls back to normal execution.

This is appropriate for linker wrappers (or post-link packagers) that can update
an existing output graph instead of rebuilding from scratch.

## 4) Toolchain-level behavior that impacts link latency

In `cxx_toolchain`, `cache_links` influences whether links are forced local via
`link_binaries_locally = not value_or(ctx.attrs.cache_links, True)`.

Practical takeaway:

- If link outputs are cacheable/reusable in your environment, keep links
  cache-friendly and remote-friendly.
- If links are intentionally non-cacheable (e.g., heavy stamping), local linking
  may still be selected by toolchain policy.

## 5) Quick investigation workflow for a specific huge target

1. Inspect link args and artifacts via generated subtargets like
   `linker.argsfile` (and `index.argsfile` when distributed ThinLTO is active).
2. Check whether link actions run local vs remote by adjusting
   `link_execution_preference`.
3. Evaluate `enable_distributed_thinlto` on the heaviest binaries first.
4. For custom/starlark link-like steps, prototype `incremental_remote_outputs`
   with robust metadata/state handling.

## Recommended order of operations

For most very large projects:

1. First: move heavyweight links to `remote` or `full_hybrid` where possible.
2. Second: turn on distributed ThinLTO for eligible binaries/toolchains.
3. Third: add custom incremental remote-output workflows only where links are
   still dominant and naturally incremental.

