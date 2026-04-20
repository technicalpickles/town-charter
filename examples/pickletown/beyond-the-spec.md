# Beyond the Spec

> Last updated: 2026-04-06. These are experiments built on the spec's foundation. Some have stuck, some are still evolving.

**[Back to the Pickletown companion](README.md)** if you arrived here directly.

<!-- markdownlint-disable MD036 -->

The [Town Charter spec](../../spec.md) defines seven concepts. Pickletown implements all of them, covered in the [main companion](README.md). This document covers what Pickletown has built on top of that foundation: citizens, a daily newspaper, a skill system, extended project patterns, and an expanded bean ecosystem.

These are not prescriptions. They are experiments that emerged from daily use. Some are genuinely useful. Some are playful. A few are both.

---

## Citizens

A citizen is an automated agent that can run against the town. The concept: define a role with a name, a description, a scope, and a set of skills, then let it operate. There are a handful of experimental citizens today (Engineer, Clerk, Reporter, Paige Turner, Slab Serif), but only one does real load-bearing work: the Sanitation Worker. It was the first citizen built, and it is where the pattern was shaken out.

### Why Sanitation Exists

A working town accumulates cruft. Branches merge but their worktrees hang around. Beans get marked `in-progress`, worked on for an afternoon, and then forgotten when something more interesting shows up. Tool repos drift out of sync with upstream. `main` falls behind. State files pile up in the workspace repo itself. None of this is a crisis on its own, but it compounds. After a month of daily use across ~19 repos, you have a dozen stale worktrees, twenty questionable beans, and a few repos where `main` is two weeks behind.

Any single item takes thirty seconds to handle. The aggregate is the chore nobody gets around to, so the town gets progressively worse to live in. Sanitation exists to make that chore tractable.

### The Role

Sanitation is deliberately narrow. It does not write code, answer questions, or weigh in on design. It sweeps. Its job is to notice the boring problems a human would never prioritize on their own, and either fix them automatically or flag them with enough context that a human can approve or dismiss each one in under a minute.

The checks cover the kinds of cruft that accumulate in a real town:

- **Worktree Health**: worktrees whose branches have merged, worktrees with no commits, and worktrees with uncommitted changes older than a week.
- **Repo Freshness**: tracked repos where `main` is behind upstream, or where `git gc` has not run in a while.
- **Bean Staleness**: `in-progress` beans with no commits, no PR, and no recent activity.
- **Town Hygiene**: uncommitted state files in the pickletown repo itself, or untracked files that should be categorized or gitignored.
- **Claude Hook Health**: broken or missing hooks in the workspace's `.claude/` configuration.

Each finding gets a severity tier (`error`, `warning`, `notice`, `ok`) so a scanning human can ignore anything below `warning` on a quick pass.

### The Two-Pass Workflow

Early sanitation runs had a problem. The citizen would receive raw findings and then spend thirty tool calls re-investigating each one: which worktree is this, what is its PR status, when did this bean last change. Every answer required running `pt` or `gh` or `git` again, which burned context and made a ten-minute sweep take forty.

The fix was to split the work into two passes:

1. **Plan.** A pre-pass runs the checks, enriches each finding with the context the citizen would otherwise have to gather (PR status, git log, bean history), and writes the result as a JSON action plan. Each action has an ID, a proposed command, a confidence level, and a finding block with enough context to make a decision without further investigation.
2. **Decide and execute.** The citizen reads the plan, writes a single `current-decisions.json` that maps each action ID to `approve`, `reject`, or `skip`, and runs `pt sanitation execute`. The executor runs the approved actions and reports back.

Three tool calls instead of thirty. The interesting work (judgment) is where the citizen spends its turns, and the boring work (gathering context, running commands) happens in deterministic code outside the agent. The decisions file also serves as a lightweight audit trail, since every run leaves a record of what the citizen chose to do and what it passed on.

### The Definition

Citizens are defined in YAML files under `citizens/<name>/`:

```yaml
name: Sanitation Worker
description: Maintains town hygiene, triages stale beans, cleans worktrees
model: sonnet
scope:
  default: town
  accepts:
    - town
skills:
  - sanitation
bootstrap:
  - gather-report.sh
```

The `bootstrap` step runs before the citizen launches and produces the enriched action plan. The plan becomes the citizen's starting context, so it begins each run fully briefed.

### The Reality

Sanitation is manually invoked. You run `pt citizen sanitation` and it launches a Claude Code session with the pre-computed plan in hand. It is not scheduled, not triggered by events, not autonomous.

The aspiration is autonomous operation: a citizen that runs on a schedule, files its own beans for problems it finds, and cleans up what it can without asking. The current reality is human-initiated, which is the right starting point. Before autonomy is useful, the workflow has to be boring, predictable, and safe. Sanitation is most of the way there, and the `citizen.yml` format is the interface contract for the autonomous version, even though today it is just configuration for a manually launched agent.

---

## The Daily Gazette

The Pickle Town Gazette is a daily summary of workspace state, formatted as a small-town newspaper. Active beans, recent commits, PR status, worktree health, all presented with headlines, bylines, and the occasional editorial flourish.

### What It Looks Like

Each edition covers:

- **Active work.** Beans in progress, recently completed items, stale items that need attention.
- **Repository activity.** Recent commits and PRs across tracked repos.
- **Community standards.** Workspace health metrics, convention adherence, outstanding reviews.

The format is playful (headlines like "OVERTIME FIX LANDS IN PRODUCTION" with a reporter byline) but the content is genuinely useful for daily orientation. Reading the gazette in the morning tells you where you left off, what landed overnight, and what needs attention.

### The Podcast

Some gazette editions include a podcast episode. The script is generated from the gazette content, then converted to audio using AI text-to-speech. The result is a short audio briefing you can listen to while making coffee.

The podcast production workflow involves the `/daily-gazette` skill for the gazette itself, then the `/podcast` skill for script generation and audio preparation.

### History

The gazette has been running since February 4, 2026. It generates to `docs/daily-gazette/` with date-prefixed filenames. Podcast episodes and scripts live in `docs/daily-gazette/podcast/`. The archive is versioned in the workspace repo, so you can look back at any day's edition.

### Generation

The `/daily-gazette` skill orchestrates the whole process. It gathers workspace state (session log, bean status, git activity across repos), generates the gazette content, and writes the output files. The skill is invoked manually, usually first thing in the morning.

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
