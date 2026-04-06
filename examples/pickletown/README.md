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

---

## Work Tracking

> The spec's [Work Tracking](../../concepts/work-tracking.md) concept: a lightweight system for tracking what you are working on, tied to branches and code rather than living in a separate tool.

Pickletown uses **beans** (from the [`beans` CLI](https://github.com/hmans/beans)) as its tracking item. A work unit in Pickletown is four things tied together: a bean (tracking), a branch (code), a worktree (location), and a PR (review).

### A Bean

A bean is a markdown file stored in a configured directory inside the workspace repo. Here is a real one:

```text
$ pt beans show gt-6os6

gt-6os6  in-progress   epic  [normal]  project-pt-tooling
PT Tooling
──────────────────────────────────────────────────

  Pickle Town CLI tooling: pt commands, workflow management, workspace
  migration.

  ## Status (2026-03-31)

  Active. Core pt commands working. Recent focus on citizen system, sanitation
  automation, and beans workflow.
```

Beans have an ID (`gt-6os6`), a status, a type, a priority, and tags. The body is freeform markdown. Because they are files, they are versioned alongside everything else in the workspace.

### The Work Unit

`pt status` pulls all four components together. Given a bean ID, a PR number, or a GitHub URL, it resolves the full picture:

```text
$ pt status gt-pw2k

Bean:      gt-pw2k  in-progress  task  [normal]
Title:     Add OAuth token refresh
Branch:    gt-pw2k+add-oauth
Worktree:  ~/pickleton/repos/zenpayroll/worktrees/gt-pw2k+add-oauth/
PR:        #4521  open  2 approvals  checks passing
```

This is the core payoff. Instead of running `beans show` + `gh pr view` + `git worktree list` and mentally stitching the results together, one command gives you the full state.

### Tool Choices

Beans stores tracking items as markdown files in a configured directory, versioned in the workspace repo. `pt` commands operate on the work unit as a whole:

- `pt status <ref>` shows the combined state of bean, branch, worktree, and PR.
- `pt resume <ref>` prints the worktree path and shows context for picking work back up.
- `pt close <ref>` verifies the PR merged, updates the bean status, and offers to clean up the worktree.

Ref resolution is flexible. Bean ID (`gt-pw2k`), short ID (`pw2k`), PR number (`#4521`), and GitHub URL all resolve to the same work unit.

### What Works Well

**Beans travel with the workspace.** They are files in the repo, so they are always available, always versioned, and never locked behind a SaaS tool's API. Grepping across beans works. Bulk updates work. Offline access works.

**`pt` collapses multi-step operations.** Closing a work unit used to mean: check if the PR merged, update the bean, delete the worktree, prune the branch. Now it is one command that walks through each step.

### What is Rough

**Bean body updates replace the entire body.** There is no append or comment command. If you want to add a status update to a bean, you have to read the current content first and include it in the update. Easy to lose content if you are not careful.

**Discovered associations require manual linking.** When you realize two beans are related, or that a bean should be tagged with a project, you update each one by hand. There is no automatic linking from branch names, PR references, or commit messages.

---

## Projects

> The spec's [Projects](../../concepts/projects.md) concept: a home for work that spans multiple repositories or sessions, with structure that supports planning, design, and handoffs.

Pickletown has roughly 20 project folders under `projects/`. Each one groups the plans, design docs, brainstorming artifacts, and handoffs for a body of work that does not belong to any single repo.

### The Layout

A real example, the project tracking this very document:

```text
projects/town-charter/
  README.md
  plans/
  design/
  brainstorming/
  handoffs/
```

`README.md` is the manifest: status, goals, related beans, key artifacts. The subdirectories are conventional. `plans/` holds implementation plans. `design/` holds architecture and specs. `brainstorming/` holds explorations and spikes. `handoffs/` holds session context for resumption (date-prefixed, like `2026-04-03-gdev-wish-ci-consolidation.md`).

Each project has an epic bean. The town-charter project's epic is `gt-rp12`. Related beans are tagged `project-town-charter`, so `pt beans list --tag project-town-charter` shows everything associated with the project.

### Tool Choices

`pt` does not have project-specific commands yet. Project creation is manual: `mkdir -p projects/<name>/{plans,design,brainstorming,handoffs}`, then create the README manifest and the epic bean. Navigation is by convention, the same way workspace navigation is.

### What Works Well

**Cross-repo work has a home.** When a body of work touches three repos and needs a design doc, an implementation plan, and session handoffs, the project folder holds all of it. The alternative is scattering these artifacts across repo-specific docs directories, where they lose their relationship to each other.

**Handoffs accumulate.** Each session that parks work writes a handoff to the project's `handoffs/` directory. Over time, this builds a history of what was tried, what worked, and what was abandoned. Useful context for anyone (human or AI) picking the work back up.

**Design docs are versioned.** Because projects live in the workspace repo, design documents get the same git history as everything else. You can see how a plan evolved, who changed it, and when.

### What is Rough

**No automated project scaffolding.** Creating a new project means remembering the directory structure, creating the epic bean, and writing the README manifest by hand. A `pt project create` command would help, but it does not exist yet.

**README manifests drift.** The project README is only accurate if someone actively maintains it. Beans get completed, new ones get created, and the manifest falls behind. There is no automation to keep it in sync with the actual bean state.
