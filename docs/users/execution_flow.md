---
id: execution_flow
title: Execution Flow for Complex Buck2 Projects
---

This guide explains how Buck2 evaluates large graphs with multiple toolchains,
target platforms, and execution platforms, including remote execution and
multiple executors. It focuses on the order of decisions and where you define
each piece, so you can predict and debug how actions run.

## 1) Define targets and dependencies (unconfigured graph)

Start by declaring targets in `BUCK` files. Build files hold target
declarations, while shared logic (macros, helpers) belongs in `.bzl` files so
targets stay declarative and predictable.【F:docs/concepts/build_file.md†L6-L53】
Targets reference other targets via `deps`, which forms the unconfigured target
graph Buck2 will configure later.【F:docs/concepts/build_file.md†L31-L110】

## 2) Declare toolchains once in the toolchains cell

Toolchains are regular rules that set `is_toolchain_rule = True` and return the
toolchain provider(s) expected by build rules (typically `*ToolchainInfo`).【F:docs/rule_authors/writing_toolchains.md†L6-L13】
Toolchain targets live in the `toolchains//` cell; the cell’s location is set by
`cells.toolchains` in `.buckconfig`.【F:docs/rule_authors/writing_toolchains.md†L12-L15】

Rules then depend on toolchains using `toolchain_dep` attributes, which lets
rule implementations consume the right compiler/linker without hardcoding
paths.【F:docs/rule_authors/writing_toolchains.md†L16-L19】 When you need multiple
variants (debug vs release, different compilers, CPU families), toolchain rules
typically use `select` to swap settings based on the active configuration.【F:docs/rule_authors/writing_toolchains.md†L71-L72】

**Why this matters:** in large repos, you declare each toolchain once and then
pick among variants using configuration. That keeps target definitions stable
while the toolchain selection remains programmable.

## 3) Pick target platforms and resolve configuration

Buck2 applies configurations (collections of constraint values) to targets so
`select` branches can be resolved.【F:docs/concepts/glossary.md†L77-L92】 Target
platform selection is driven by:

- `default_target_platform` attributes on rules.
- `--target-platforms` on the command line when you want to build multiple
  variants (e.g., Linux + macOS).【F:docs/concepts/glossary.md†L87-L92】

You can inspect how configuration affects a target with `uquery` vs `cquery`.
`cquery` shows the configured node and includes the `buck.target_configuration`
and `buck.execution_platform` fields so you can see what each target is built
for and on.【F:docs/concepts/configurations.md†L65-L82】

## 4) Associate each configured target with an execution platform

An execution platform is a rule that describes how actions should run (local,
remote, or hybrid) and whether cache uploads are supported.【F:docs/concepts/glossary.md†L126-L133】
Each configured target is paired with an execution platform, and that
association is visible in `cquery` via the `buck.execution_platform`
attribute.【F:docs/concepts/configurations.md†L65-L82】

**Ordering note:** target configuration happens before execution platform
selection, which ensures that execution platform selection can take the target’s
constraints into account (for example, matching an OS/CPU pair).

## 5) Configure remote execution and multiple executors

Remote execution endpoints are configured in `.buckconfig` under
`[buck2_re_client]`, including the engine, action cache, and CAS endpoints for
your provider(s).【F:docs/users/remote_execution.md†L19-L46】

You then create one or more execution platform rules that enable remote
execution by setting `remote_enabled = True` in `CommandExecutorConfig`, and
optionally keep local execution enabled to support hybrid execution.【F:docs/users/remote_execution.md†L52-L75】

**Multiple executors:** in practice, this is how you represent multiple remote
executors (different execution platforms). Each execution platform can carry
distinct remote properties, such as container images or instance names, and
Buck2 will select the appropriate platform when configuring targets.【F:docs/users/remote_execution.md†L52-L75】

## 6) Putting it together (order of decisions)

In order, Buck2 evaluates:

1. **Target graph definition** from `BUCK` files and `deps`.【F:docs/concepts/build_file.md†L31-L110】
2. **Toolchain availability** from `toolchains//` via `toolchain_dep`.【F:docs/rule_authors/writing_toolchains.md†L16-L19】
3. **Target platform/configuration** to resolve `select` branches and
   produce configured targets.【F:docs/concepts/glossary.md†L77-L92】
4. **Execution platform selection** for each configured target; visible as
   `buck.execution_platform` in `cquery`.【F:docs/concepts/configurations.md†L65-L82】
5. **Executor choice (local/remote/hybrid)** based on execution platform
   configuration and `[buck2_re_client]` settings.【F:docs/users/remote_execution.md†L19-L75】

When debugging, use `cquery` to confirm the target configuration and execution
platform, and confirm that your execution platform configuration points at the
right remote executor(s).【F:docs/concepts/configurations.md†L65-L82】【F:docs/users/remote_execution.md†L19-L75】
