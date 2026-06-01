---
id: transitions_one_pager
title: Buck2 Transitions (One-Pager)
---

A **transition** in Buck2 is a function that rewrites the configuration used on
part of the dependency graph.

In plain terms: when target `A` depends on `B`, a transition lets Buck2 build
`B` in a *different* configuration than `A`.

## Why transitions exist

Without transitions, configuration mostly propagates unchanged from a target to
its deps. Transitions are the controlled escape hatch when a dependency must be
built differently (for example, for a different OS/CPU constraint set).

## Mental model

- A configuration includes:
  - constraint values (platform-like constraints), and
  - config values/settings.
- A transition takes an input configuration and returns a new one.
- Buck2 then configures downstream nodes with that returned configuration.

Because of this, the **same label can appear multiple times** in the configured
graph (same target, different configurations).

## Transition types

Buck2 exposes two primary styles:

1. **Incoming transition**
   - Declared on a rule instance via `incoming_transition`.
   - Applied when *someone depends on that target*.
   - Rule definitions must opt in via `supports_incoming_transition = True`.

2. **Outgoing transition**
   - Declared on a dependency attribute (for example
     `attrs.transition_dep(cfg = ":my_transition")`).
   - Applied to dependencies reached through that attribute.

## How a transition is defined

A transition is provided by a **configuration rule** that returns
`TransitionInfo(impl = ...)`.

- The implementation receives the pre-transition `PlatformInfo`.
- It returns a new `PlatformInfo` (or, in legacy split mode, multiple outputs).
- Transition-defining rules are marked `is_configuration_rule = True`.

## Important rule: idempotence

Buck2 requires transitions to be idempotent:

- applying it once and
- applying it twice

must produce the same final configuration.

If this is violated, Buck2 errors.

## Practical guidance

- Keep transitions small and explicit; only change what you must.
- Prefer modeling via normal attributes + toolchains/platforms first; add
  transitions only where propagation alone cannot express intent.
- Treat split transitions and old `transition(...)` object style as legacy; the
  `TransitionInfo` target-based API is the preferred path for new code.

## How to inspect transitions

Use configured graph tooling:

- `buck2 cquery ...` to inspect configured nodes (including per-target
  configurations).
- When a label appears multiple times in output, it often indicates distinct
  configured instances caused by transitions.

## Quick example use-case

You have a mostly Linux target graph, but one dependency must be resolved as
watchOS-specific. An outgoing transition on that dependency edge can swap the OS
constraint only for that edge, instead of changing the whole parent target's
configuration.
