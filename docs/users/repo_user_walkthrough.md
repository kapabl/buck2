---
id: repo_user_walkthrough
title: Buck2 User Walkthrough From the Repository
---

This page summarizes what you need to know as a Buck2 user by pointing at the
key materials already shipped in this repository. It highlights the essentials
for getting Buck2 installed, shaping your workspace, and running everyday
commands.

## What Buck2 offers

Buck2 is designed to be fast, hermetic, and multi-language, letting you model
large dependency graphs while keeping iteration tight.【F:README.md†L19-L50】 The
project ships the same prelude Meta uses internally so you can build on top of a
battle-tested rule set even before adding your own extensions.【F:README.md†L73-L80】

## Install Buck2

You can download a fresh binary from the [`latest` release
page](https://github.com/facebook/buck2/releases/tag/latest) or build from source
using the repository itself.【F:docs/getting_started/install.md†L13-L38】 To build
from source, install the matching nightly toolchain and install the crate:

```bash
rustup install nightly-2025-08-01
cargo +nightly-2025-08-01 install --git https://github.com/facebook/buck2.git buck2
```

Add Cargo's bin directory to your `PATH` and run `buck2 --help` to confirm the
CLI is available.【F:docs/getting_started/install.md†L39-L59】 Using
[dotslash](https://dotslash-cli.com) alongside the bi-monthly releases is also a
repository-friendly way to keep the right binary per commit.【F:docs/getting_started/install.md†L17-L22】

## Lay out a workspace

A Buck2 workspace is organized around build files, typically named `BUCK`, that
declare your targets in Starlark.【F:docs/concepts/build_file.md†L6-L29】 Build
files can host multiple targets, and you define cross-package dependencies by
adding targets to `deps` instead of referencing sibling source files directly in
other packages.【F:docs/concepts/build_file.md†L31-L110】 Buck2 keeps builds
predictable by restricting build files to target declarations—control flow and
arbitrary functions belong in `.bzl` files—and by enforcing deterministic target
naming rules.【F:docs/concepts/build_file.md†L31-L53】 This minimal example shows a
pair of C/C++ targets sharing sources inside a package:

```python
cxx_binary(
    name = 'hello',
    srcs = [
        'main.c',
    ],
    deps = [
        ':greeting',
    ],
)

cxx_library(
    name = 'greeting',
    srcs = [
        'greeting.c',
    ],
    exported_headers = [
        'greeting.h',
    ],
)
```
【F:docs/concepts/build_file.md†L55-L95】

## Run the core commands

A Buck2 command combines an action (such as `build`, `run`, or `test`), one or
more target patterns, and optional flags.【F:docs/getting_started/core_concepts.md†L426-L444】
You can pass multiple targets or target patterns in a single invocation, and
Buck2 will evaluate them in parallel when possible.【F:docs/getting_started/core_concepts.md†L446-L455】
After building, `buck2 run`, `buck2 test`, and `buck2 install` let you execute or
package the results, while `buck2 query` and `buck2 cquery` help explore the
configured graph.【F:docs/getting_started/core_concepts.md†L457-L468】 For quick
inspections, the cheat sheet in `docs/users/cheatsheet.md` provides ready-made
`cquery` snippets for listing package targets, printing attributes, and
reviewing dependencies.【F:docs/users/cheatsheet.md†L7-L62】

## Where to go next

- The [Getting Started tutorials](../getting_started/index.md) walk through your
  first build and add dependencies incrementally.
- The [Concepts](../concepts/key_concepts.md) section explains terminology like
  targets, packages, and the Buck daemon.
- The [Cheat Sheet](./cheatsheet.md) and the [Build Observability
  guides](./build_observability) are practical references when diagnosing build
  issues.
