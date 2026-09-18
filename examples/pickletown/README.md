# Pickletown: A Town Charter Companion

> Last updated: 2026-09-18. Pickletown is a living workspace; details here reflect its state at the time of writing. The mechanism-level reference lives in Pickletown's own `docs/ARCHITECTURE.md` (private repo); this companion is the spec-to-implementation view.

Pickletown is a real town. One developer has used it daily since January 2026 to manage work across ~120 tracked repositories and ~70 project folders, including a large Rails monolith, several open-source tools, and the Town Charter repo itself. It is the workspace Town Charter was extracted from.

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
- [Routing & Delegation](#routing--delegation)

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
    cq/
      checkout/                  # plain clone (the newer layout)
        .claude/worktrees/
          fix-trace-open/
    ... (~120 tracked repos total)
  beans/
  docs/
  projects/
  workflows/
  town/
  .claude/
```

Tracked repos come in two layouts. The original: `repos/<name>/bare.git/` holds a bare clone and `repos/<name>/worktrees/<branch>/` holds each working area. The newer one, adopted in September 2026 to work with the `wt` ([worktrunk][worktrunk]) tool: `repos/<name>/checkout/` is a plain clone and extra branches live at `checkout/.claude/worktrees/<branch>/`. `pt` detects which layout a repo uses and every command dispatches on it. Branch directories for work-unit branches carry a bean ID (like `gt-pw2k+add-oauth`) so you can trace a worktree back to its tracking item at a glance.

### Tool Choices

The CLI is `pt` (short for `pickletown`). Two commands handle the workspace lifecycle:

- `pt track <url> --name <name>` adds a repository. It creates the clone and an initial default-branch working area.
- `pt checkout <repo> <branch>` creates a new working area in the right place for the repo's layout, then runs pt's `post-checkout` hook (which runs `mise trust`).

`pt` exists because raw `git worktree add` does not know about the workspace's directory conventions. Without the wrapper, every new worktree would require remembering the path pattern and typing it out. The CLI makes the convention the default.

From inside a Claude Code session the same two operations go through an MCP server instead of the shell: `pt serve` exposes `track_repo` and `create_worktree` at `localhost:9876/mcp`. The reason is mechanical. Claude Code's Bash sandbox denies writes to `.git/config`, `.git/hooks/*`, `.claude/agents/*`, and `.mcp.json` at any depth, which is exactly what a clone or worktree creation writes. The MCP server runs outside that sandbox, so the checkout succeeds and pt's hooks still run. A `PreToolUse` hook blocks the shell form so sessions don't rediscover the failure.

### What Works Well

**Multiple branches active simultaneously.** The main benefit of the bare-clone-plus-worktree approach. You can have a PR under review in one worktree, a hotfix in another, and a spike in a third, all for the same repo, all checked out and ready. No stashing, no branch switching, no reconstructing state.

**Convention-based navigation.** Because every repo and every worktree follows the same directory pattern, both humans and AI assistants can find things by convention. `repos/zenpayroll/worktrees/main/` is predictable. No configuration lookup required. That said, typing out these paths by hand is tedious. AI assistants navigate by convention just fine, but for humans, a `cd` helper (likely a shell alias or function, since a subprocess cannot change the parent shell's directory) would make this practical. Pickletown does not have one yet.

### What is Rough

**Disk usage from many active worktrees.** Each worktree is a full checkout. A large monolith with 15 active worktrees uses real disk space. Cleanup discipline matters, and it is the kind of thing that slips when you are busy. Pickletown's sanitation system (covered in [Beyond the Spec](beyond-the-spec.md)) helps with this, but the underlying cost is still there.

**Worktrees created outside `pt checkout` need manual setup.** `pt checkout` runs a post-checkout hook that handles `mise trust` automatically, so worktrees created through `pt` just work. Worktrees created with raw `git worktree add` skip that hook, and tools (ruby, node, etc.) and pre-commit hooks will not work until you run `mise trust` by hand.

**Two layouts coexist.** Every code path that enumerates worktrees has to ask which layout a repo uses. That is one helper (`RepoLayout`) and it works, but it is a permanent branch in the code that a single-layout town would not carry. The same split exists for the workspace repo's own branches, which live at `.claude/worktrees/` today and at an older `.worktrees/` for a handful of stale entries that still need cleaning up.

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

**Bean bodies rot in a specific way.** Bodies are freeform, and anything written as a relative claim ("healthy for 3 days", "just merged") reads wrong a month later with no signal that it has. The convention that emerged is a `Last verified: <date>` line on any claim about live state, and absolute identifiers (SHA, PR number) over computed distances. `beans update --body-append` exists now, so adding a status line no longer risks clobbering the body, but the rot problem is about what gets written, not how.

**Discovered associations require manual linking.** When you realize two beans are related, or that a bean should be tagged with a project, you update each one by hand. There is no automatic linking from branch names, PR references, or commit messages.

**Keeping beans committed is a chore.** Because beans are files in the workspace repo, every status change, every new bean, every body update is an uncommitted change. Pickletown handles this with a commit-safety classifier (`PathTier`: beans, docs, and projects are `content` and safe to auto-commit; pt's own source is `code` and waits for a human; tracked repos are `never`). Bean writes auto-commit on the spot, and the sanitation sweep commits whatever `content` is left over. The cost moved from "remember to commit" to "keep the root on `main`": one working tree shared by every concurrent session means a branch switch in the root silently captures everyone's commits, so branch work has to happen in a worktree. See [Beyond the Spec](beyond-the-spec.md#state-that-churns).

---

## Projects

> The spec's [Projects](../../concepts/projects.md) concept: a home for work that spans multiple repositories or sessions, with structure that supports planning, design, and handoffs.

Pickletown has about 70 project folders under `projects/`. Each one groups the plans, design docs, brainstorming artifacts, and handoffs for a body of work that does not belong to any single repo.

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

`pt` still has no `pt project create`. Project creation is manual: `mkdir -p projects/<name>/{plans,design,brainstorming,handoffs}`, then create the README manifest and the epic bean. What did land is `pt handoffs list|where|new`, which knows the project layout well enough to put a dated handoff in the right `handoffs/` directory, and `pt workspaces`, which treats each project as a workspace you can jump to (as a [tmux][tmux] session) alongside tracked repos. Navigation is otherwise by convention.

### What Works Well

**Cross-repo work has a home.** When a body of work touches three repos and needs a design doc, an implementation plan, and session handoffs, the project folder holds all of it. The alternative is scattering these artifacts across repo-specific docs directories, where they lose their relationship to each other.

**Handoffs accumulate.** Each session that parks work writes a handoff to the project's `handoffs/` directory. Over time, this builds a history of what was tried, what worked, and what was abandoned. Useful context for anyone (human or AI) picking the work back up.

**Design docs are versioned.** Because projects live in the workspace repo, design documents get the same git history as everything else. You can see how a plan evolved, who changed it, and when.

### What is Rough

**No automated project scaffolding.** Creating a new project means remembering the directory structure, creating the epic bean, and writing the README manifest by hand. A `pt project create` command would help, but it does not exist yet.

**README manifests drift.** The project README is only accurate if someone actively maintains it. Beans get completed, new ones get created, and the manifest falls behind. Pickletown's answer is a family of *reality-check* skills (`project-reality-check`, `bean-reality-check`, `worktree-reality-check`): the sanitation sweep flags a project whose manifest looks stale, and the skill is the judgment layer that reconciles the manifest against the epic bean, the handoffs directory, and git activity. It is triggered, not continuous.

---

## AI Conventions

> The spec's [AI Conventions](../../concepts/ai-conventions.md) concept: persistent instructions that shape how AI assistants behave in your workspace, accumulated over time rather than designed upfront.

Pickletown uses Claude Code's `.claude/rules/` directory for conventions and a Claude Code *plugin* for procedures. There are twelve rule files and twenty-three plugin skills. Each rule is a response to friction that got expensive enough to write down.

### The Rule Files

Rules are loaded at every session start, so they stay short and say *what*, pointing at a skill for *how*:

- **`beans.md`** ensures `pt beans` is used within Pickletown rather than bare `beans`, and keeps bean IDs out of external-facing repos.
- **`pt-root-stays-on-main.md`** is the shared-tree rule: never switch the workspace root off `main`, always scope commits to paths, re-check the branch right before acting. It exists because concurrent sessions kept getting their commits captured by a branch another session had left checked out.
- **`pt-source-protection.md`** prevents modifying pt's own source code during unrelated work. If you are fixing a bug in zenpayroll and notice something wrong with pt, the rule says to create a bean, not edit the CLI mid-task.
- **`sandbox-git-writes.md`** covers the Bash sandbox: which git writes it denies, that repo writes go through the MCP server, and that everything else is a per-command unsandboxed retry.
- **`working-with-mise.md`** covers [mise][mise] trust requirements and what to do when trust errors appear. The rule exists because those errors are easy to dismiss as noise when they actually block real work.
- **`projects.md`**, **`code-reviews.md`**, **`jira.md`**, **`workflows.md`**, **`watch-register.md`**, **`working-with-pitchfork.md`**, **`working-with-qmd.md`** cover the remaining conventions: project layout, review handling, the Jira conventions per project, the workflow runtime, the PR watch register, and the two local services sessions lean on.

### Rules Versus Skills

Two of the rules that used to be here (`pt-workflow.md`, `working-with-external-repos.md`) are gone. Their content moved into skills (`pt-work-units`, `pt-repos`) in a September 2026 reorg, once it was clear that a rule loaded into every session is the wrong place for a multi-page procedure. The rule stays as the trigger ("when a repo name comes up, use the `pt-repos` skill"); the skill carries the steps and is loaded only when needed.

The skills ship as a Claude Code plugin (`pickletown`, from a local marketplace in the workspace repo) along with the hooks that do ambient work: recording session starts, tracking which skills a session used, blocking the shell form of repo writes, nudging for devlog entries. The plugin is what makes the conventions portable to a *field crew* session that runs with a lean config (see Routing & Delegation below).

### How They Accumulate

These rules were not designed as a system. Each one started the same way: an AI session did something wrong, it cost time to fix, and the correction got written down so it would not happen again. `pt-source-protection.md` exists because a session once edited CLI source code while working on an unrelated repo. `working-with-mise.md` exists because sessions kept ignoring trust errors. `pt-root-stays-on-main.md` exists because a branch switch in the shared root captured three other sessions' commits.

The accumulation pattern works well. Rules compound: a new session that reads them starts with orientation that took months to develop. Problems that used to recur across sessions get fixed once and stay fixed.

### What is Rough

**Rules can go stale.** As workflows evolve, the rules that describe them may not keep up. A command gets renamed, a convention shifts, and the rule file still describes the old way. There is no validation that rules are still accurate. (This document rotted the same way: the April 2026 version of this section listed seven rules, two of which no longer existed by September.)

**Plugin skills are served from a cache.** Claude Code copies a plugin into a cache at install time and only refreshes it on a version bump. An edit to a skill in the workspace repo is a silent no-op until a `pt plugin sync` mirrors it into the cache or the version is bumped. Repo-local skills under `.claude/skills/` don't have this problem, and Pickletown keeps a handful of town-root procedures there for that reason.

**No feedback loop for effectiveness.** It is hard to tell which rules are actually preventing mistakes versus which ones are just taking up context window space. Some rules might be unnecessary because the underlying issue was fixed elsewhere, but there is no mechanism to detect that.

---

## Session Tracking

> The spec's [Session Tracking](../../concepts/session-tracking.md) concept: recording when AI sessions happen and what they touch, so you can reconstruct a timeline of work later.

Pickletown records sessions through a `SessionStart` hook shipped in its plugin. The hook reads Claude Code's hook payload and calls `pt sessions start <id> --source <s> --cwd <dir>`, which records the session ID, timestamp, working directory, branch, and source. It also exports `CLAUDE_SESSION_ID` into the session's environment so later hooks and commands can associate work with the session.

### The Store

The store started as an append-only JSONL file (`.sessions/session-index.jsonl`). In September 2026 it moved to a [Dolt](https://www.dolthub.com/) database under `~/.local/state/pickletown/dolt/`, served by a `dolt sql-server` that [launchd][launchd] keeps alive. Two reasons: the JSONL file was a perpetually-dirty tracked file in a working tree shared by every session, and concurrent hook invocations writing to a Dolt data-dir without a server raced on its manifest (5 of 8 parallel writers failed in one measurement; through the server, none). The CLI is unchanged: `pt sessions list|show|track|resume` read and write the same rows, and `pt sessions track --bean <id>` records an explicit association mid-session.

Because each row includes the working directory, you can still answer "when did this branch last get attention" or "how many sessions touched zenpayroll this week", now with SQL instead of grep.

### Associations

The working directory in each log entry resolves to a repo and branch through the workspace's directory conventions. A branch often maps to a bean (via the `gt-xxxx` prefix in worktree names). So a session entry implicitly associates with a bean, a repo, and sometimes a PR.

This association is partially automated: the directory-to-repo-to-branch chain is structural. The rest is partially manual. References discovered in transcripts after the fact (a session that discussed bean `gt-pw2k` without being in its worktree) require someone to notice and record the link.

### What Works Well

**Queryable session history.** Finding all sessions for a branch, a repo, or a time range is one query. Dolt gives the store its own commit/diff/log history, so the data keeps versioning without contributing to `git status` in the workspace repo.

**Ambient capture.** Because the hook fires automatically, session tracking requires zero discipline. You do not have to remember to log your work. Every session that starts in the workspace gets recorded.

### What is Rough

**Association discovery is basic.** The working directory gives you repo and branch, but that is the extent of the automation. If a session discusses work across multiple repos, only the starting directory gets recorded.

**Transcript scanning is a separate tool.** A session transcript contains rich information about what was discussed, which beans were referenced, and what decisions were made. Pickletown mines that with `cq` (SQL over session transcripts via [DuckDB][duckdb]), a tracked repo of its own, and a `scripts/active-threads` report that feeds the morning gazette. Neither writes associations back into the session store; the discovered-association pipeline the spec describes still does not exist.

---

## Session Continuity

> The spec's [Session Continuity](../../concepts/session-continuity.md) concept: preserving context across session boundaries so new sessions can pick up where previous ones left off.

Pickletown uses the [`agent-meta`](https://github.com/technicalpickles/pickled-claude-plugins) skill (with `park` and `unpark` commands) to capture and restore session context. When you park a session, the skill writes a handoff document. When you start a new session and unpark, it reads that document and orients the session before any work begins.

### Handoffs

A handoff is a markdown file that captures the state of work at the moment you stop. Handoffs go to `projects/<name>/handoffs/` when a project exists, or `docs/handoffs/` as a fallback. Filenames are date-prefixed: `2026-04-03-gdev-wish-ci-consolidation.md`. Two kinds have emerged: a *park* (continuation: what to do next) and a *wrap* (closeout, `-wrapped` suffix: what happened and what threads are left open). There are about 450 of them in `docs/handoffs/` alone as of September 2026. A `PostToolUse` hook commits a handoff the moment it is written, so a session that dies right after parking still leaves its handoff on `main`.

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

**Handoffs degrade over time.** A handoff written yesterday is useful. A handoff written two weeks ago may describe code that has since changed, decisions that were revisited, or next steps that are no longer relevant. There is no automated staleness detection for old handoffs, so you have to judge freshness yourself. The `unpark` path does one thing about this: a hook injects the current state of any PR the handoff mentions from the watch register (below, under CLI Patterns), so at least the "is this merged yet" question is answered with live data rather than the handoff's claim.

---

## CLI Patterns

> The spec's [CLI Patterns](../../concepts/cli.md) concept: a local command-line tool that encodes workspace conventions into deterministic operations.

Pickletown's CLI is `pt` (alias for `pickletown`), written in Ruby with no gem to install. It wraps git, GitHub, and beans operations into commands that enforce the workspace's directory conventions and work unit model, and it has grown to about fifteen command families.

### Core Commands

The spec-level commands map to four areas:

**Repo management.** `pt track` adds a repository to the workspace. `pt list` shows what is tracked. `pt new-repo` creates a brand-new local repo in the layout.

**Working area management.** `pt checkout` creates a worktree in the right place with the right naming convention. `pt worktrees` lists active worktrees across one or all repos.

**Work unit lifecycle.** `pt status` shows the combined state of a bean, branch, worktree, and PR. `pt resume` prints the worktree path and loads context for picking work back up. `pt close` verifies the PR merged, updates the bean, and offers to clean up the worktree. `pt sup` is the morning dashboard.

**Overview.** `pt worktrees` gives a cross-repo view of all active working areas. `pt beans list` shows tracking items, filterable by tag, status, or repo. `pt workspaces` lists every repo and project as a workspace and connects to it as a [tmux][tmux] session.

Beyond those, `pt` fronts the town's automation: `pt sanitation` (the maintenance sweep), `pt crew` (dispatch a field crew), `pt watch` (the PR watch register: your own open PRs, one recommended action each, refreshed every fifteen minutes by a [launchd][launchd] poller), `pt sessions` and `pt handoffs`, `pt search` (semantic search over the town via qmd), `pt serve` (the web UI and MCP server), `pt sync` (commit the workspace repo's own changes, with Claude deciding what to commit, skip, or gitignore), `pt characters`, `pt plugin sync`, `pt hooks`, and `pt init` for standing up a new town.

### Ref Resolution

All workflow commands accept flexible refs. A bean ID (`gt-pw2k`), a short ID (`pw2k`), a PR number (`#123`), or a full GitHub URL all resolve to the same work unit. The CLI figures out which repo, branch, worktree, and PR you mean. You use whatever identifier you have at hand.

### What Works Well

**Deterministic operations.** `pt checkout zenpayroll my-branch` always creates the worktree at `repos/zenpayroll/worktrees/my-branch/`. There is no ambiguity about where things go. One command replaces what would otherwise be four manual steps (fetch, create worktree, set path, trust [mise][mise]).

**Convention enforcement.** The CLI encodes the workspace's directory layout and naming patterns. You cannot accidentally create a worktree in the wrong place or track a repo with a conflicting name, because the tool prevents it.

### What is Rough

**Coverage gaps.** Some workflows still require raw git commands. There is no `pt project create` for scaffolding projects, and some git operations (interactive rebase, cherry-pick) have no pt wrapper and probably should not.

**Name collision.** `pt` conflicts with a different `pt` available through Homebrew. If you have the Homebrew version installed, you need to adjust your PATH to prefer Pickletown's `pt` or create a shell alias. This was an early surprise that required manual intervention.

**Discovery.** `pt --help` is now a full command map grouped by family, which fixes the "what exists" problem. It still does not explain the flow (track, then checkout, then status, then close); that lives in the `pt-repos` and `pt-work-units` skills and in `docs/onboarding-martin.md`, a walkthrough written for the first person other than the author to stand up a town with `pt init`.

**The CLI is a shared surface for humans and hooks.** `pt sessions start` is called by a Claude Code hook on every session start; `pt claude-hook` runs the crew-orientation and dispatcher-nudge hooks; the MCP server invokes the same command classes the CLI does. A slow or failing `pt` command is therefore a slow or failing session start. The workspace resolver in particular is written to never raise and to treat any unreadable signal (no [tmux][tmux] server, cwd outside the tree) as "unknown" rather than an error.

---

## Routing & Delegation

> The spec's [Routing & Delegation](../../concepts/routing-and-delegation.md) concept: a coordinating session with the wide view routes bounded work to isolated execution contexts instead of doing it in place, across an in-process / local / remote spectrum.

Pickletown's coordinating session runs as **Dispatch** (the persona is Patch Callahan, who works the downtown desk with the wide view). A `dispatcher` skill is the routing rule: when a request arrives, Dispatch decides whether to handle it inline (it needs the cross-project view), survey it first (the shape is unclear), or hand it to a crew (it is bounded fieldwork in one repo). The point is to keep the downtown session from silting up with the detail of one repo's work.

The three execution modes map straight onto the spec's spectrum.

### In-process: survey crews

For research and recon, Dispatch fans out **survey crews**: subagents inside the same session, spawned through Claude Code's [Agent and Explore tools](https://code.claude.com/docs/en/sub-agents). They sweep the codebase or the workspace, read what they need, and report findings back into Dispatch's context. They are read-only and short-lived. This is how an unclear request gets its shape before anything is delegated for real.

### Local isolated: field crews

For bounded fieldwork, Dispatch spawns a **field crew** with `pt crew`: a separate Claude session, scoped to one bean and one job site (worktree), running under a lean [`cenv`](https://github.com/technicalpickles/cenv) environment so its plugin and config surface is minimal. Because it is a separate session in its own working area, it can edit, build, and iterate without touching Dispatch's view. `pt crew` spawns, watches, attaches to, and tears down these crews. Two flavors exist: the interactive TUI in a [tmux][tmux] window (attachable, watched by polling the pane), and `--headless`, which runs `claude -p` with streaming JSON output rendered live in the pane and writes a compact `result.json` the watcher polls for the exact completion signal.

The job site is created by Dispatch, not the crew, through the MCP `create_worktree` tool with a `trust_env` parameter that pre-registers Claude Code's folder trust for the crew's environment, so the crew is not stopped by a "do you trust this folder?" prompt it cannot answer. A `dispatch-prep` subagent does the noisy prep (checkout, trust, toolchain warmup) outside Dispatch's context and reports a ready-or-blocked verdict. A `SessionStart` hook in the crew's environment delivers the minimal job-site safety rules (use `pt beans`, retry sandbox-denied git writes unsandboxed, `mise trust`, stay inside the job site) so the crew does not depend on the workspace's full rule set leaking in. As of late August 2026, about a hundred work orders had been dispatched this way.

### Remote: cloud crews

For work that should proceed unwatched, Dispatch dispatches a **cloud crew** with `/wish` (the Wishing Well cloud agent). It is fire-and-forget: the work runs elsewhere and Dispatch reviews the result later rather than supervising it. This is the most isolated, least supervised mode.

### The work order

The handoff artifact is a **work order**, a durable markdown file under `projects/dispatcher/work-orders/`. It carries the absolute path to the job site and a verification command, alongside the scope and intent. That path-plus-verification contract is what lets a crew land ready and lets Dispatch confirm the result instead of taking the crew's word for it.

### What Works Well

**The downtown session stays clean.** Routing bounded work to crews is what keeps the coordinating session able to hold the wide view across ~19 repos. The win is context economy, exactly as the concept frames it; the parallelism is a bonus.

**Survey-first picks the right mode.** Cheap in-process recon before delegating means Dispatch rarely hands a job to the wrong kind of crew. The shape of the work is known before a heavier local or remote crew spins up.

### What is Rough

**The routing discipline is convention, not tooling.** Dispatch follows the `dispatcher` skill, but nothing enforces "don't grab a shovel." A session can still dive into fieldwork it should have delegated. The guard is the skill and the habit, not a hard rule.

**Work-order quality is manual.** A handoff is only as good as what Dispatch writes into it. A work order missing a real job-site path or a verification command produces a crew that lands lost or work nobody can confirm, which is the concept's named failure mode in practice.

**Remote crews have thin visibility.** Fire-and-forget cloud crews proceed without supervision, which is the point, but it also means a stuck or wrong remote crew is not noticed until you go looking. Local crews you can attach to; remote crews you mostly wait on.

---

## Workflow Narrative

The concepts above are building blocks. This section shows how they compose over the course of a real day.

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

[worktrunk]: https://worktrunk.dev
[mise]: https://mise.jdx.dev/
[duckdb]: https://duckdb.org/
[tmux]: https://github.com/tmux/tmux
[launchd]: https://www.launchd.info/
