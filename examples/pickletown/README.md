# Pickletown: A Town Charter Companion

> Last updated: 2026-04-06. Pickletown is a living workspace; details here reflect its state at the time of writing.

Pickletown is a real town. One developer uses it daily to manage work across ~19 tracked repositories, including a large Rails monolith, several open-source tools, and the Town Charter repo itself. It is the workspace Town Charter was extracted from.

This document maps each spec concept to its Pickletown implementation: the directory layout, the CLI commands, the conventions that stuck, and the rough edges that remain. It is honest about what works and what does not.

<!-- markdownlint-disable MD036 MD051 -->

## Table of Contents

**Concepts**

- [Workspace](#workspace)
- [Work Tracking](#work-tracking)
- [Projects](#projects)
- [AI Conventions](#ai-conventions)
- [Session Tracking](#session-tracking)
- [Session Continuity](#session-continuity)
- [CLI Patterns](#cli-patterns)

**In Action**

- [Workflow Narrative](#workflow-narrative)

**See also:** [Beyond the Spec](beyond-the-spec.md) covers Pickletown patterns that go further than Town Charter defines.

<!-- markdownlint-enable MD036 MD051 -->

---

## Workspace

> The spec's [Workspace](../../concepts/workspace.md) concept: a single directory that contains everything, with tracked repositories stored as bare clones and working areas created per branch.

### The Layout

Pickletown lives at `~/pickleton/`. The top-level structure:

```text
~/pickleton/
  repos/
    zenpayroll/
      bare.git/
      worktrees/
        main/
        gt-pw2k+add-oauth/
    town-charter/
      bare.git/
      worktrees/
        main/
    gusto-karafka/
      bare.git/
      worktrees/
        main/
        gt-ly8i+use-routing-draw-cop/
    ... (~19 tracked repos total)
  docs/
  projects/
  citizens/
  .claude/
  .sessions/
```

Each tracked repo follows the same shape: `repos/<name>/bare.git/` holds the bare clone, and `repos/<name>/worktrees/<branch>/` holds each active working area. Branch directories often include a bean ID prefix (like `gt-pw2k+add-oauth`) so you can trace a worktree back to its tracking item at a glance.

### Tool Choices

The CLI is `pt` (short for `pickletown`). Two commands handle the workspace lifecycle:

- `pt track <url> --name <name>` adds a repository. It creates the bare clone at `repos/<name>/bare.git/` and an initial `main` worktree.
- `pt checkout <repo> <branch>` creates a new working area. It wraps `git worktree add` and enforces the `repos/<name>/worktrees/<branch>/` layout.

`pt` exists because raw `git worktree add` does not know about the workspace's directory conventions. Without the wrapper, every new worktree would require remembering the path pattern and typing it out. The CLI makes the convention the default.

### What Works Well

**Multiple branches active simultaneously.** The main benefit of the bare-clone-plus-worktree approach. You can have a PR under review in one worktree, a hotfix in another, and a spike in a third, all for the same repo, all checked out and ready. No stashing, no branch switching, no reconstructing state.

**Convention-based navigation.** Because every repo and every worktree follows the same directory pattern, both humans and AI assistants can find things by convention. `repos/zenpayroll/worktrees/main/` is predictable. No configuration lookup required.

### What is Rough

**Worktrees need `mise trust` after creation.** Pickletown uses mise for tool version management. Every new worktree is an untrusted directory until you run `mise trust`, which means tools (ruby, node, etc.) and pre-commit hooks will not work until you do. It is easy to forget and confusing when it fails silently.

**Disk usage from many active worktrees.** Each worktree is a full checkout. A large monolith with 15 active worktrees uses real disk space. Cleanup discipline matters, and it is the kind of thing that slips when you are busy. Pickletown's sanitation system (covered in [Beyond the Spec](beyond-the-spec.md)) helps with this, but the underlying cost is still there.
