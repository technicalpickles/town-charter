# CLI Patterns

## Motivation

The gap between "I want to resume working on X" and actually doing it is several commands: find the working area, check the tracking item, look at the PR, piece together context. Each command is an opportunity for things to go sideways. An AI assistant doing all of this can take different paths on different days, losing track of intermediate state or making unnecessary tool calls along the way.

A CLI collapses multi-step operations into single deterministic calls. One command that does one thing reliably beats an agent making four tool calls that might take different paths each time. For mechanical operations, predictability is more valuable than flexibility.

## Mental Model

The CLI is a deterministic interface to town operations. One command, one outcome, every time.

It complements the AI rather than replacing it. The CLI handles mechanical operations: status checks, context loading, working area lifecycle, branch checkout. The AI handles judgment: what to work on, how to approach a problem, what the status means and what to do about it. That division keeps each doing what it does best.

The CLI's primary abstraction is the work unit. Commands create, inspect, resume, and close work units. The CLI knows how to find the right working area from a tracking ID, a branch name, or a PR number. You should not need to know which identifier format to use.

## Capabilities

A CLI implementation for a town covers four areas:

- **Repo management**: Add, list, and remove tracked repositories. After adding a repo, the workspace knows it exists and where working areas for it live.

- **Working area management**: Create an isolated working area for any repo and branch, list all active working areas across the workspace, and remove working areas that are no longer needed.

- **Work unit lifecycle**: The four operations: create a work unit, check combined status, resume to get into context, close after the work is done.

- **Overview**: A dashboard across all active work. What is in flight, what needs attention, what is waiting on CI or review. One command, all repos.

## In Practice

The workflows section shows what these operations look like in practice. The pattern throughout is the same: a single CLI call replaces a sequence of manual steps.

For example, resuming work manually means locating the working area, opening the PR, reading the tracking item, and orienting to where you left off. That is a context reconstruction problem repeated every time you pick something up. The CLI reduces it to one call that prints the working area path, the tracking item state, and the relevant PR context together.

The manual approach is not just slow; it is fragile. The CLI is neither.

For a concrete implementation of this concept, see the [Pickletown example](../examples/pickletown/README.md#cli-patterns).

## Design Considerations

**The spec does not prescribe command names, flag syntax, or output format.** What matters is that each operation is a single entrypoint, produces the same result every time, and is composable with other tools. The CLI should feel like a thin layer over the workspace and work tracking concepts, not a separate system.

**Flexible reference resolution is required.** A work unit should be addressable by tracking ID, branch name, or PR number. Any of these should resolve to the full work unit without knowing which format you started with. You encounter work from different directions: a notification mentions a PR number, a colleague pastes a branch name, you remember a tracking ID. The CLI should handle all of them.

**Determinism is the point.** The CLI is not smarter than the AI, but it is consistent. That consistency is what makes it safe to call from within an AI session without worrying about what path it will take. An AI invoking a CLI command gets the same result it would have gotten yesterday or will get tomorrow.
