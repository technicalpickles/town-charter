# Pickletown: A Town Charter Companion

> Last updated: 2026-04-06. Pickletown is a living workspace; details here reflect its state at the time of writing.

Pickletown is a real town. One developer uses it daily to manage work across ~19 tracked repositories, including a large Rails monolith, several open-source tools, and the Town Charter repo itself. It is the workspace Town Charter was extracted from.

This document maps each spec concept to its Pickletown implementation: the directory layout, the CLI commands, the conventions that stuck, and the rough edges that remain. It is honest about what works and what does not.

<!-- markdownlint-disable MD036 -->

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

<!-- markdownlint-enable MD036 -->

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

**Disk usage from many active worktrees.** Each worktree is a full checkout. A large monolith with 15 active worktrees uses real disk space. Cleanup discipline matters, and it is the kind of thing that slips when you are busy. Pickletown's sanitation system (covered in [Beyond the Spec](beyond-the-spec.md)) helps with this, but the underlying cost is still there.

**Worktrees created outside `pt checkout` need manual setup.** `pt checkout` runs a post-checkout hook that handles `mise trust` automatically, so worktrees created through `pt` just work. Worktrees created with raw `git worktree add` skip that hook, and tools (ruby, node, etc.) and pre-commit hooks will not work until you run `mise trust` by hand.

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

---

## AI Conventions

> The spec's [AI Conventions](../../concepts/ai-conventions.md) concept: persistent instructions that shape how AI assistants behave in your workspace, accumulated over time rather than designed upfront.

Pickletown uses Claude Code's `.claude/rules/` directory to store conventions. There are currently seven rule files, each one a response to friction that got expensive enough to write down.

### The Rule Files

- **`beans.md`** ensures `pt beans` is used within Pickletown rather than bare `beans`. Without this, beans can end up in a `.beans.yml` found in a tracked repo instead of the workspace's configured directory.
- **`code-reviews.md`** defines the PR review workflow: when to suggest reviews, how to save artifacts, how to handle findings.
- **`projects.md`** documents the project folder structure, the relationship between projects and epic beans, and handoff conventions.
- **`pt-source-protection.md`** prevents modifying pt's own source code during unrelated work. If you are fixing a bug in zenpayroll and notice something wrong with pt, the rule says to create a bean, not edit the CLI mid-task.
- **`pt-workflow.md`** documents pt's workflow commands and ref resolution so sessions can use `pt status`, `pt resume`, and `pt close` correctly.
- **`working-with-external-repos.md`** is a checklist for tracking repos before touching code. Check if it is tracked, check if the URL is tracked under a different name, create a worktree through `pt checkout`.
- **`working-with-mise.md`** covers mise trust requirements and what to do when trust errors appear. The rule exists because those errors are easy to dismiss as noise when they actually block real work.

### How They Accumulate

These rules were not designed as a system. Each one started the same way: an AI session did something wrong, it cost time to fix, and the correction got written down so it would not happen again. `pt-source-protection.md` exists because a session once edited CLI source code while working on an unrelated repo. `working-with-mise.md` exists because sessions kept ignoring trust errors.

The accumulation pattern works well. Rules compound: a new session that reads all seven files starts with orientation that took weeks to develop. Problems that used to recur across sessions get fixed once and stay fixed.

### What is Rough

**Rules can go stale.** As workflows evolve, the rules that describe them may not keep up. A command gets renamed, a convention shifts, and the rule file still describes the old way. There is no validation that rules are still accurate.

**No feedback loop for effectiveness.** It is hard to tell which rules are actually preventing mistakes versus which ones are just taking up context window space. Some rules might be unnecessary because the underlying issue was fixed elsewhere, but there is no mechanism to detect that.

---

## Session Tracking

> The spec's [Session Tracking](../../concepts/session-tracking.md) concept: recording when AI sessions happen and what they touch, so you can reconstruct a timeline of work later.

Pickletown records sessions in `.sessions/session-index.jsonl`. A hook fires at session start and captures the session ID, timestamp, working directory, branch, and source.

### The Log

Each session start produces one line:

```json
{"event":"start","session_id":"abc123","ts":"2026-04-06T17:59:51Z","cwd":"/Users/josh/pickleton/repos/zenpayroll/worktrees/main","branch":"main","source":"hook"}
```

The log is append-only JSONL. Because it includes the working directory, you can answer questions like "when did this branch last get attention" or "how many sessions touched zenpayroll this week" with a grep.

### Associations

The working directory in each log entry resolves to a repo and branch through the workspace's directory conventions. A branch often maps to a bean (via the `gt-xxxx` prefix in worktree names). So a session entry implicitly associates with a bean, a repo, and sometimes a PR.

This association is partially automated: the directory-to-repo-to-branch chain is structural. The rest is partially manual. References discovered in transcripts after the fact (a session that discussed bean `gt-pw2k` without being in its worktree) require someone to notice and record the link.

### What Works Well

**Queryable session history.** The JSONL format is simple and greppable. Finding all sessions for a branch, a repo, or a time range is a one-liner. The log grows slowly (one line per session start) so it stays manageable.

**Ambient capture.** Because the hook fires automatically, session tracking requires zero discipline. You do not have to remember to log your work. Every session that starts in the workspace gets recorded.

### What is Rough

**Association discovery is basic.** The working directory gives you repo and branch, but that is the extent of the automation. If a session discusses work across multiple repos, only the starting directory gets recorded.

**Transcript scanning is not automated yet.** A session transcript contains rich information about what was discussed, which beans were referenced, and what decisions were made. Extracting that information is manual today. An automated pipeline from transcript to associations would close the gap, but it does not exist yet.

---

## Session Continuity

> The spec's [Session Continuity](../../concepts/session-continuity.md) concept: preserving context across session boundaries so new sessions can pick up where previous ones left off.

Pickletown uses the `agent-meta` skill (with `park` and `unpark` commands) to capture and restore session context. When you park a session, the skill writes a handoff document. When you start a new session and unpark, it reads that document and orients the session before any work begins.

### Handoffs

A handoff is a markdown file that captures the state of work at the moment you stop. Handoffs go to `projects/<name>/handoffs/` when a project exists, or `docs/handoffs/` as a fallback. Filenames are date-prefixed: `2026-04-03-gdev-wish-ci-consolidation.md`.

Each handoff captures:

- **Work unit reference.** The bean ID, branch, and PR so the new session can find the code.
- **What was being done.** The goal of the session, not just the last command.
- **Current state.** What is working, what is broken, what is half-finished.
- **Next steps.** Concrete actions, not vague directions.
- **Decisions made.** Why the current approach was chosen, what alternatives were rejected.

The `agent-meta:park` skill automates the capture. It prompts for the key fields and writes the handoff file. `agent-meta:unpark` reads a handoff and loads the context into a fresh session.

### What Works Well

**Fresh sessions with clean context.** Each new session starts from the handoff rather than inheriting stale state from a long-running conversation. There is no accumulated confusion from abandoned approaches or forgotten dead ends. The new session gets exactly the context it needs and nothing else.

### What is Rough

**Handoffs degrade over time.** A handoff written yesterday is useful. A handoff written two weeks ago may describe code that has since changed, decisions that were revisited, or next steps that are no longer relevant. There is no automated staleness detection for old handoffs, so you have to judge freshness yourself.

---

## CLI Patterns

> The spec's [CLI Patterns](../../concepts/cli.md) concept: a local command-line tool that encodes workspace conventions into deterministic operations.

Pickletown's CLI is `pt` (alias for `pickletown`), built in Node.js. It wraps git, GitHub, and beans operations into commands that enforce the workspace's directory conventions and work unit model.

### Core Commands

The commands map to four areas:

**Repo management.** `pt track` adds a repository to the workspace (bare clone plus initial worktree). `pt list` shows what is tracked.

**Working area management.** `pt checkout` creates a worktree in the right place with the right naming convention. `pt worktrees` lists active worktrees across one or all repos.

**Work unit lifecycle.** `pt status` shows the combined state of a bean, branch, worktree, and PR. `pt resume` prints the worktree path and loads context for picking work back up. `pt close` verifies the PR merged, updates the bean, and offers to clean up the worktree.

**Overview.** `pt worktrees` gives a cross-repo view of all active working areas. `pt beans list` shows tracking items, filterable by tag, status, or repo.

### Ref Resolution

All workflow commands accept flexible refs. A bean ID (`gt-pw2k`), a short ID (`pw2k`), a PR number (`#123`), or a full GitHub URL all resolve to the same work unit. The CLI figures out which repo, branch, worktree, and PR you mean. You use whatever identifier you have at hand.

### What Works Well

**Deterministic operations.** `pt checkout zenpayroll my-branch` always creates the worktree at `repos/zenpayroll/worktrees/my-branch/`. There is no ambiguity about where things go. One command replaces what would otherwise be four manual steps (fetch, create worktree, set path, trust mise).

**Convention enforcement.** The CLI encodes the workspace's directory layout and naming patterns. You cannot accidentally create a worktree in the wrong place or track a repo with a conflicting name, because the tool prevents it.

### What is Rough

**Coverage gaps.** Some workflows still require raw git commands. There is no `pt project create` for scaffolding projects, no `pt beans comment` for appending to a bean, and some git operations (interactive rebase, cherry-pick) have no pt wrapper and probably should not.

**Discovery.** There is no `pt help` that explains the overall workflow model. Each command has its own help text, but the relationship between commands (track, then checkout, then status, then close) is only documented in `.claude/rules/` files. A new user would not know the intended flow without reading those rules.

---

## Workflow Narrative

The seven concepts above are building blocks. This section shows how they compose over the course of a real day.

### Starting new work

> See also: [Starting New Work](../../workflows/starting-new-work.md)

A Slack thread surfaces a bug in the payroll calculation service. Time to fix it.

```text
$ pt beans create "Fix overtime calc rounding error" -t bug -s in-progress --tag repo-zenpayroll
Created gt-mw7f  Fix overtime calc rounding error

$ pt checkout zenpayroll gt-mw7f+fix-overtime-rounding
Fetching origin...
Creating worktree at repos/zenpayroll/worktrees/gt-mw7f+fix-overtime-rounding/
Done.

$ cd ~/pickleton/repos/zenpayroll/worktrees/gt-mw7f+fix-overtime-rounding/
```

Three things now exist as a single unit: a bean for tracking, a branch for code, and a worktree for an isolated working area. The branch name carries the bean ID, so the connection is structural. You are in the directory, looking at code.

### Switching context

> See also: [Switching Context](../../workflows/switching-context.md)

Mid-fix, a teammate asks for a review on a switchboard PR. You need to look at their code, but you have uncommitted changes in the overtime fix.

```text
$ pt worktrees switchboard
switchboard
  main              clean
  gt-k4r1+add-dlq   3 uncommitted

$ cd ~/pickleton/repos/switchboard/worktrees/main/
```

Your overtime worktree stays exactly as you left it: uncommitted files, half-written test, editor state. You did not stash anything. You did not commit a placeholder. You walked to a different directory.

You review the PR, leave comments, and walk back.

### Resuming work

> See also: [Resuming Work](../../workflows/resuming-work.md)

Back to the overtime fix. You remember the bean ID from earlier (or check `pt beans list -s in-progress`).

```text
$ pt resume gt-mw7f

Bean:      gt-mw7f  in-progress  bug  [normal]
Title:     Fix overtime calc rounding error
Branch:    gt-mw7f+fix-overtime-rounding
Worktree:  ~/pickleton/repos/zenpayroll/worktrees/gt-mw7f+fix-overtime-rounding/
PR:        (none)

$ cd ~/pickleton/repos/zenpayroll/worktrees/gt-mw7f+fix-overtime-rounding/
```

One command gives you the full picture: where the code is, what state the work is in, whether a PR exists yet. Your uncommitted changes are still there. You pick up exactly where you stopped.

### Tracking a repo

> See also: [Tracking a Repo](../../workflows/tracking-a-repo.md)

Later that afternoon, a thread in Slack mentions a flaky test in a repo you have not worked with before: `gusto-karafka`.

```text
$ pt list | grep karafka
(no output)

$ pt track git@github.com:Gusto/gusto-karafka.git --name gusto-karafka
Cloning into bare repo at repos/gusto-karafka/bare.git/...
Creating initial worktree at repos/gusto-karafka/worktrees/main/
Done. Run: mise trust ~/pickleton/repos/gusto-karafka/worktrees/main

$ pt checkout gusto-karafka gt-t92v+fix-flaky-consumer-test
Fetching origin...
Creating worktree at repos/gusto-karafka/worktrees/gt-t92v+fix-flaky-consumer-test/
Done.
```

The new repo is tracked in the workspace, following the same conventions as every other repo. Your AI assistant can discover it. Navigation works the same way. Six months from now, when someone asks "where is the gusto-karafka repo," the answer is the same as it is for every other repo: `repos/gusto-karafka/worktrees/`.

### Closing out work

> See also: [Closing Out Work](../../workflows/closing-out-work.md)

The overtime fix PR got approved and merged. Time to clean up.

```text
$ pt close gt-mw7f

PR #4892 is merged. Proceeding.
Updated gt-mw7f status: in-progress → completed
Removing worktree: repos/zenpayroll/worktrees/gt-mw7f+fix-overtime-rounding/
Pruning branch: gt-mw7f+fix-overtime-rounding
Done.
```

One command verifies the merge, updates the bean, removes the worktree, and prunes the branch. Everything that was created together gets cleaned up together.

### Reviewing activity

> See also: [Reviewing Activity](../../workflows/reviewing-activity.md)

End of day. You want to know what you touched.

The session log in `.sessions/session-index.jsonl` has a line for every session that started today: which directory, which branch, what time. Grepping it by date gives you a timeline.

```text
$ grep "2026-04-06" .sessions/session-index.jsonl | jq -r '[.ts, .cwd] | @tsv'
2026-04-06T09:14:22Z    /Users/josh/pickleton/repos/zenpayroll/worktrees/gt-mw7f+fix-overtime-rounding
2026-04-06T11:03:44Z    /Users/josh/pickleton/repos/switchboard/worktrees/main
2026-04-06T11:41:09Z    /Users/josh/pickleton/repos/zenpayroll/worktrees/gt-mw7f+fix-overtime-rounding
2026-04-06T14:15:33Z    /Users/josh/pickleton/repos/gusto-karafka/worktrees/gt-t92v+fix-flaky-consumer-test
2026-04-06T16:50:01Z    /Users/josh/pickleton/repos/town-charter/worktrees/main
```

Five sessions, four repos, the full shape of the day. Sessions that produced commits sit alongside sessions that were pure investigation. The gazette (covered in [Beyond the Spec](beyond-the-spec.md)) turns this raw data into a daily summary, but the session log is the foundation.

---

That is one day. The pattern repeats: create work, switch freely, resume by reference, track new repos as they appear, close what is done, review what happened. Each workflow is a single operation because the workspace structure and the work unit model make it possible. The commands are simple. The simplicity comes from the conventions underneath.
