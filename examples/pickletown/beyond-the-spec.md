# Beyond the Spec

> Last updated: 2026-06-17. These are experiments built on the spec's foundation. Some have stuck, some are still evolving.

**[Back to the Pickletown companion](README.md)** if you arrived here directly.

<!-- markdownlint-disable MD036 -->

The [Town Charter spec](../../spec.md) defines seven concepts. Pickletown implements all of them, covered in the [main companion](README.md). This document covers what Pickletown has built on top of that foundation: two kinds of automation (workflows and skills) and the pattern that composes them, a morning newspaper with its own podcast, a cast of named characters, a web app and a daemon layer that keep the lights on, extended project patterns, and an expanded bean ecosystem.

These are not prescriptions. They are experiments that emerged from daily use. Some are genuinely useful. Some are playful. A few are both.

---

## Automation: One Procedure, Two Runtimes

Pickletown reaches for two kinds of automation, and they divide the work cleanly. A **workflow** is a deterministic graph of stages (shell commands, one-shot LLM calls, human choices) that runs as a Python subprocess. It has no persona. It takes inputs, runs its stages, and exits. A **skill** is a procedure loaded into the current Claude Code session: markdown with frontmatter that shapes how the assistant behaves right now. Workflows run *next to* you as subprocesses. Skills run *inside* the session you are already in.

There used to be a third kind. A *citizen* was an automated agent with a name, a persona, a scope, and a toolset: you defined a role, then let it operate. The Sanitation Worker was the first one built and the only one that did load-bearing work. ADR 0004 retired the citizens runtime in May 2026. The reason was plain: every citizen that actually ran did so by running a workflow as its first step and reading the output. The persona-and-toolset shell never used the agency it was supposed to provide. So the runtime went away, the `pt citizen` command was deleted, and the surviving work folded into workflows. Persona and identity come back as a templating concern where they earn their keep (see the Characters section below), not as a runtime.

### Single Procedure, Two Runtimes

Retiring citizens left a real question. Some work needs to run both ways: autonomously on a schedule (gather the facts, emit structured findings) and interactively with a human (judgment calls, redirecting mid-investigation, batch decisions). The answer is the pattern that replaced citizens. One skill defines the procedure once, and both runtimes load it.

The skill is the canonical playbook. Autonomous Claude loads it from a `claude -p` stage inside a workflow. Interactive Claude loads it in a resumed human session. Same Skill tool, same steps. The only adaptation lives inside the skill: each step is labeled **autonomous-safe** (deterministic gather work, structured output) or **interactive-only** (anything that needs a human prompt or an ownership conversation). The autonomous stage does the safe steps and hands off; the human picks up where it left off.

`workflows/pr-review/` proves the shape. Its `ReviewNode` runs the `josh-pr-review` skill autonomously and drafts a review; its `HandoffNode` prints `claude --resume <session-id>` so a human can continue with the same skill already loaded. Sanitation generalizes the idea to a daily driver.

### Sanitation Worker

The Sanitation Worker is the worked example, and the case that drove the whole retirement. It is now a PocketFlow workflow plus a skill, not a citizen. It runs maintenance sweeps across the town: cleaning up merged worktrees, triaging beans that have gone quiet, keeping tool repos current with upstream, and flagging workspace health issues. The role is deliberately narrow. It does not write code, answer questions, or weigh in on design. It sweeps, and that is all.

#### Why It Exists

A working town accumulates cruft. Branches merge but their worktrees hang around. Beans get marked `in-progress`, worked on for an afternoon, and then forgotten when something more interesting shows up. Tool repos drift out of sync with upstream. `main` falls behind. State files pile up in the workspace repo itself. None of this is a crisis on its own, but it compounds. After months of daily use across ~90 repos, you have a dozen stale worktrees, twenty questionable beans, and a few repos where `main` is two weeks behind.

Any single item takes thirty seconds to handle. The aggregate is the chore nobody gets around to, so the town gets progressively worse to live in. Sanitation exists to make that chore tractable.

#### What It Checks

The checks cover the kinds of cruft that accumulate in a real town:

- **Worktree Health**: worktrees whose branches have merged, worktrees with no commits, and worktrees with uncommitted changes older than a week.
- **Repo Freshness**: tracked repos where `main` is behind upstream, or where `git gc` has not run in a while.
- **Bean Staleness**: `in-progress` beans with no commits, no PR, and no recent activity.
- **Town Hygiene**: uncommitted state files in the pickletown repo itself, or untracked files that should be categorized or gitignored.
- **Claude Hook Health**: broken or missing hooks in the workspace's `.claude/` configuration.

Each finding gets a severity tier (`error`, `warning`, `notice`, `ok`) so a scanning human can ignore anything below `warning` on a quick pass.

#### The Pipeline

The workflow is a seven-stage graph: `plan → classify → judge → execute → digest → context_map → handoff`. The split is the whole point. Mechanical work runs in deterministic code; judgment runs where an LLM or a human adds value.

1. **Plan** runs the checks and enriches each finding with the context a human would otherwise have to gather (PR status, git log, bean history), writing a JSON action plan. Each action carries an ID, a proposed command, and a finding block with enough context to decide without further investigation.
2. **Classify** is pure Python, no LLM. Rule-based, it auto-approves the high-confidence, low-risk actions and queues the rest.
3. **Judge** is an LLM stage that re-decides only the queued actions Classify did not auto-approve.
4. **Execute** runs the approved commands and reports back.
5. **Digest** and **Context Map** summarize the run. The digest also feeds the gazette on days when Sal Vage is the writer.
6. **Handoff** prints `claude --resume <session-id>` so a human can pick up the open triage decisions interactively, with the full context already loaded.

That last stage is the single-procedure-two-runtimes pattern in action: the same `/sanitation` skill drives both the autonomous stages and the resumed session. The decisions also serve as a lightweight audit trail, since every run leaves a record of what was approved and what was passed on.

#### The Reality

Sanitation is no longer manually invoked. It runs as `workflows/sanitation/bin/sanitation`, and a pitchfork daemon fires it on a six-hour cron (see The Daemon Layer below). Unattended, it drains the deterministic, auto-approved actions and skips the interactive triage. Run with a TTY, it stops at the handoff so a human can decide the rest.

The old aspiration (a worker that runs on a schedule and cleans up what it can without asking) is now just how it works. Getting there did not need a citizen runtime. It needed the workflow to be boring, predictable, and reversible, which is what makes unattended acting safe.

---

## Workflows

A workflow is a deterministic graph of stages (shell commands, one-shot LLM calls, human choices) that runs as a Python subprocess. Workflows have no persona. They take inputs, run their stages, and exit. Humans invoke them. Cron, CI, or other workflows invoke them. The workflow does not care who called.

### Why It Is a Separate Concept

The original Sanitation Worker was a single agent that did everything: gathered context, ran checks, made decisions, executed actions. That worked, but it wasted the agent's turns on deterministic work. Splitting those into typed stages, where pure-Python rules handle the obvious cases and an LLM stage only weighs in on the genuinely ambiguous ones, is the seam where workflows became their own concept.

LLM stages are good at judgment, terrible at running thirty `git` commands in a row to gather context. Shell and Python stages are good at running thirty `git` commands in a row, terrible at deciding what to do with the output. A workflow graph lets each kind of stage do what it does best, in one pipeline.

### The Runtime

Pickletown's workflows run on [PocketFlow](https://github.com/the-pocket/PocketFlow). The runtime was settled after an evaluation period under `projects/pocketflow-eval/`, which now survives as a project log: ADRs explaining the adoption decision, a CONTEXT.md that nails down vocabulary, and a learnings file capturing what was hard. Forward-looking framework code lives at `workflows/lib/`; new workflows live at `workflows/<name>/`.

Fabro was the prior runtime. It is deprecated and the migration was uneventful enough that the most interesting artifact from it is an ADR explaining why PocketFlow won.

### Anatomy

A workflow lives at `workflows/<name>/`:

```text
workflows/<name>/
  bin/<name>            # bash wrapper that boots uv + main.py
  flow.py               # the graph (nodes and edges)
  main.py               # entry point: arg parsing, banner, flow.run
  nodes.py              # Node subclasses for this workflow
  prompts/              # prompt files for any LLM stages
  README.md             # what this workflow does and how to run it
  requirements.txt      # pinned framework dependency
```

The framework primitives in `workflows/lib/` cover the common stage shapes: `ShellNode` for shell commands, `ClaudeCodeNode` for one-shot `claude -p` calls (an agentic Claude Code session), `LLMNode` for a direct single-model completion, `ChoiceNode` for human gates, and `EndNode` for terminal stages. A workflow author subclasses these and declares class attributes. Anything rule-based gets a plain Python `Node`: sanitation's classifier, for instance, is pure Python with no model call at all. When a stage needs data from an external service, `workflows/lib/pt_workflows/mcp_client.py` calls any MCP tool already configured for Claude Code without spinning up a session. The graph in `flow.py` wires nodes together with edge conditions like `succeeded`, `failed`, `partially_succeeded`, and `skipped`.

The bin scripts compute their own pt-root by walking up to find `.beans.yml`, so workflows are location-independent: a workflow runs the same wherever its directory sits. That made retiring the old `projects/pocketflow-eval/poc/` staging area a matter of moving directories, not rewiring how anything ran.

### What Has Shipped

The staging area is drained; these all live under `workflows/<name>/` now:

- **`sanitation/`** runs the maintenance sweep described above (plan → classify → judge → execute → digest → context_map → handoff).
- **`morning-gazette/`** generates the daily report, the voiced gazette, an annotation pass, and an optional podcast episode, then commits and pushes the edition.
- **`pr-review/`** drafts a Josh-style review with `claude -p`, gates on an approve/handoff/skip choice, then either posts the review or hands off a resumable session. It is the reference implementation of single-procedure-two-runtimes.
- **`watch-pr/`** and **`watch-merge/`** are long-lived blocking processes: one follows a PR through CI, reviews, and merge (auto-chaining to the other), the other follows the merged commit through CI-on-main and deploy. The wait lives in a subprocess instead of a held-open session.
- **`clerk-survey/`** prints a town-wide orientation (active epics, top tags, tracked repos, the latest beans), the surviving piece of the old Clerk citizen.

Each is a graph of the same primitives, varying only in which stages it wires together.

### A Note on Vocabulary

The Town Charter spec uses *workflow* in a different sense: user-flow patterns like [Starting New Work](../../workflows/starting-new-work.md) that show concepts in action. Those are scenarios. Pickletown's workflows are the runtime concept above. Same word, different scope. The spec scenarios describe **what** a person does in the town. Pickletown's workflows describe **how** specific automated work runs.

---

## The Daily Gazette

The Pickle Town Gazette is a daily summary of workspace state, formatted as a small-town newspaper. Active beans, recent commits, PR status, worktree health, all presented with headlines, bylines, and the occasional editorial flourish. What began as a hand-run skill is now the `morning-gazette` workflow: a multi-stage PocketFlow graph that gathers the data, drafts the edition in a character's voice, optionally cuts a podcast, and commits the result.

### The Pipeline

The stages chain like this:

`collect → pick_writer → report → editorialize → pick_annotator → annotate → gate → (write_script → tts → publish) → commit`

- **Collect** runs a shell script that gathers the window's session log, bean status, and git activity across repos.
- **Pick Writer** rotates the day's byline against the previous edition, drawing from the characters tagged `gazette-writer`.
- **Report** formats the raw data; **Editorialize** rewrites it in the chosen writer's voice.
- **Pick Annotator** and **Annotate** add a second pass: a different character (tagged `gazette-annotator`) writes a companion document and drops footnotes into the edition. When Margaret Pennywhistle draws the assignment, that companion is the community-standards review (`docs/daily-gazette/YYYY-MM-DD-pickle-town-gazette-community-standards-review.md`), a recurring critique of workspace health and convention adherence.
- **Gate** is the one human choice: cut a podcast or not (default yes; an unattended run passes `--podcast` or `--no-podcast` to skip the prompt).
- **Write Script → TTS → Publish** generate a dialogue script, render it to audio with `podcast-tts.mjs`, and write show notes. Dill McBroadcast hosts; the day's annotator is the guest.
- **Commit** is terminal on both branches. It stages today's artifacts, commits, and pushes, so the web app's gazette and podcast pages pick up the new edition without anyone running git by hand. The push is best-effort: a git hiccup is surfaced loudly but never loses a gazette already written to disk.

### Writers and Voice

The bylines are characters, not code. Writers and annotators live as persona files under `town/characters/`, discovered by tag, so adding a voice to the rotation needs no code change. The format stays playful (headlines like "OVERTIME FIX LANDS IN PRODUCTION") while the content stays genuinely useful for daily orientation. On days Sal Vage writes, the edition themes itself as a salvage log and pulls from the sanitation digest.

### History and Procedure

The gazette has been running since February 4, 2026. Editions land in `docs/daily-gazette/` with date-prefixed filenames; podcast scripts, audio, and show notes sit under `docs/daily-gazette/podcast/`. The archive is versioned in the workspace repo, so you can look back at any day's edition.

The `/daily-gazette` skill is still the canonical playbook for what an edition should contain and how it should read. The workflow runs that procedure autonomously; a human can load the same skill in a session for a one-off. Single procedure, two runtimes again.

---

## Skills

Skills are reusable workflows packaged as invocable commands. In Claude Code, they appear as slash commands: `/sanitation`, `/daily-gazette`, `/commit`, `/park`.

### The Pattern

A skill is a markdown file with frontmatter that describes when to trigger and how to behave. The frontmatter specifies the trigger conditions (slash command name, contextual triggers), and the body contains the instructions the AI assistant follows when the skill is invoked.

The pattern is: notice a workflow you repeat, capture it as a skill, invoke it deterministically. Instead of explaining the commit conventions every session, the `/commit` skill encodes them. Instead of manually remembering the code review steps, the `/code-review` skill walks through them.

### Pickletown's Skill Library

Pickletown has skills in its `.claude/skills/` directory covering:

- **Workflow automation.** Commits, code review, PR creation, CI debugging.
- **Town operations.** Sanitation, gazette generation, podcast production.
- **Context management.** Parking and unparking sessions, snapshots, handoffs.
- **Development patterns.** Brainstorming, plan writing, plan execution, TDD, systematic debugging.
- **External integrations.** Slack, Jira, Google Calendar, Obsidian.

### Superpowers

Skills are part of a plugin system called superpowers. Plugins bundle related skills together and can be shared across towns. A plugin installed in one workspace makes its skills available there. This means a team could share a set of workflow skills without each person reimplementing them.

Skills defined as markdown files with frontmatter is a Claude Code convention. The superpowers layer adds packaging, distribution, and the ability to compose skills from multiple plugins.

---

## Projects as Epicenters

The [main companion's Projects section](README.md#projects) covers the basic layout: `projects/<name>/` with subdirectories for plans, design, brainstorming, and handoffs. In practice, projects have grown into something more central than that minimal structure suggests.

### Scale

Pickletown has 20 active project folders. They range from focused efforts (a single gem upgrade) to sprawling multi-month initiatives (the claudifying-eng project that tracks AI tooling adoption across an engineering org). The town-charter project is a meta-example: this very document was planned and designed in `projects/town-charter/`.

### Extended Patterns

Beyond the basic subdirectories, some patterns have emerged:

**Scratch areas.** `projects/<name>/.scratch/` holds working artifacts that are gitignored. Draft scripts, intermediate data, generated output that is useful during a session but not worth versioning. The scratch area lets you work messily without polluting the project's committed history.

**Task files.** `projects/<name>/tasks/<issue-number>-<slug>.md` holds issue-specific working notes that span multiple sessions. Different from handoffs (session-specific) and plans (implementation proposals). A task file accumulates understanding of a problem over time: initial investigation, attempted approaches, findings, eventual resolution.

**The manifest as hub.** Each project's `README.md` lists the epic bean, related beans, key artifacts, and current status. When you start a session on a project, the manifest is the first thing to read. It is the project's table of contents, kept (ideally) in sync with reality.

### Projects as Organizing Centers

For multi-week efforts, the project folder becomes the center of gravity. Beans reference the project via tags (`project-town-charter`). Handoffs accumulate in `handoffs/`. Design decisions get captured in `design/`. The project folder is where you go to understand the full picture of a body of work, across repos and across sessions.

---

## Bean Ecosystem

The [main companion's Work Tracking section](README.md#work-tracking) covers beans as tracking items tied to branches, worktrees, and PRs. In practice, the bean system has grown beyond simple issue tracking into something more like a lightweight project management layer.

### Epic Beans

Every project has an epic bean (type: `epic`). The epic tracks the overall effort. Related beans are child work items. The connection is through tags: the epic for the town-charter project is `gt-rp12`, and related beans carry the tag `project-town-charter`.

```text
$ pt beans show gt-rp12

gt-rp12  in-progress   epic  [normal]  project-town-charter
Town Charter
──────────────────────────────────────────────────
```

`pt beans list --tag project-town-charter` shows everything associated with the project. The epic is the anchor. Individual tasks, bugs, and spikes are the branches.

### Tagging Conventions

Tags follow conventions that enable filtering:

- **`repo-<name>`** associates a bean with a repository. `repo-zenpayroll`, `repo-gusto-karafka`. When you create a bean from within a repo's worktree, `pt beans create` auto-tags it with the repo name.
- **`project-<name>`** associates a bean with a project folder. Manual tagging, but it ties the bean ecosystem to the project system.
- **`code-review`** and **`pr-NNN`** tag beans created from code review findings.

### Cross-referencing with Jira

For work that also lives in Jira, bean bodies include ticket references:

```markdown
## Related Jira Tickets
- [ASYNC-123](https://jira.example.com/browse/ASYNC-123): Consumer migration (In Progress)

## Related PRs
- [PR #789](https://github.com/org/repo/pull/789): Add DLQ routing
```

Beans are the local source of truth. Jira is the team-visible source of truth. The cross-references keep them connected without requiring synchronization.

### The Ready Filter

`pt beans list --ready` is the "what should I work on?" filter. It surfaces beans that are actionable: in-progress items, items with no blockers, items whose dependencies are resolved. The sanitation worker uses this same filter to identify stale beans (ready items that have not seen activity in a while).

The `--ready` flag turns the bean list from an inventory into a work queue. Combined with `pt resume` to pick up a specific item, it forms a lightweight daily workflow: check what is ready, pick something, resume it.

---

## Closing

These extensions are not in the Town Charter spec yet. Some may make it in. Citizens and skills are strong candidates: they solve real problems (maintenance automation, workflow capture) and the patterns have stabilized enough to describe generally. The gazette is probably too specific to Pickletown's personality to spec, but the underlying pattern (generated workspace summaries) might generalize.

The foundation matters here. The spec's seven concepts (workspace, work tracking, projects, AI conventions, session tracking, session continuity, CLI patterns) are what make these experiments possible. Citizens work because the workspace has a consistent structure to operate on. The gazette works because session logs, beans, and git activity are all queryable. Skills work because the AI conventions system gives them a place to live and a way to trigger.

That is the point of a good spec: it creates a platform you can build on.

<!-- markdownlint-enable MD036 -->
