# Projects

## Motivation

Some work does not fit in a single repository. A cross-service migration touches three repos and runs for weeks. An infrastructure initiative generates plans, decision records, and handoff notes that do not belong in any one codebase. When that work is split across repos and individual session notes, it loses coherence: plans live in one place, handoffs in another, and the rationale for decisions disappears into chat logs.

A project folder gives cross-repo work a versioned home inside the workspace. Plans, design docs, and session handoffs accumulate in one place. Because the project lives in the workspace repository, it travels with your other conventions, stays under version control, and is visible to every AI assistant session from the start.

## Mental Model

A project is a directory inside the workspace with a consistent internal structure:

```
projects/<project-name>/
  README.md          # manifest: status, goals, key artifacts
  plans/             # implementation plans
  design/            # architecture docs, specs
  brainstorming/     # explorations, rough ideas
  handoffs/          # session context for resumption
```

The README acts as the manifest: what this project is, what its current status is, and where to find the relevant artifacts. Plans and design docs capture decisions and intent. Handoffs let you (or an AI assistant) resume a session without reconstructing context from scratch.

Projects are cheap. The cost of creating one is a directory and a README. If work turns out to be simple and contained, the folder stays small. If it grows into a multi-week effort, the structure was already there.

A template directory gives you this skeleton in one command. You should not have to remember the subdirectory names or the README format each time.

## Capabilities

A project implementation supports:

- **Create a project**: Given a name, create the directory structure from a template and open the README for editing. The project is immediately available as a home for artifacts.

- **Find active projects**: List projects by name, status, or associated work tracking items. You should be able to see what is in flight without opening every README.

- **Navigate project artifacts**: Given a project name, find plans, handoffs, and design docs. Useful when resuming work after a gap.

## In Practice

Say you are coordinating a migration of three backend services to a new job processing library. The work spans multiple repositories and will take several weeks.

You create a project called `job-processing-migration`. The workspace creates the directory structure and a README stub. You fill in the goal, link the relevant tracking items, and note the three repos involved.

Over the first week, you add implementation plans to `plans/`. When you park a session mid-task, you drop a handoff note in `handoffs/` with the date and what you were doing. The next session, you read that note, get oriented in a minute, and continue.

Three weeks in, a teammate asks about the sequencing decisions. You point them to `design/` where the sequencing rationale lives. No archaeology required.

When the migration finishes, the project folder is a complete record of how it happened: the original plans, the decisions, the course corrections, the handoffs that carried context between sessions. It does not disappear into a chat log.

## Design Considerations

**The subdirectory structure is opinionated, but the names and contents can vary.** `plans/`, `design/`, `brainstorming/`, `handoffs/` are a sensible default. Teams may rename or reorganize to fit their workflow. The key property is that cross-repo work has a single versioned home, not that it follows a precise directory layout.

**Not everything needs a project.** Work that fits cleanly in one repository should stay there: in the repo's own docs, its own issue tracker, its own conventions. A project folder is for work that genuinely spans multiple repos or sessions, where no single repo is the natural home. Creating project folders for everything adds noise without adding clarity.

**The README as manifest is the load-bearing piece.** The subdirectories are helpful containers, but the README is what makes a project navigable. A good manifest captures current status, goals, and pointers to key artifacts. An AI assistant reading the README at the start of a session should be able to understand the project's state without opening every file.

**Handoffs degrade.** A handoff written three months ago describes a context that has changed. Projects work best when handoffs are dated and the README is kept current. Stale handoffs are worse than none because they create false confidence. Treat the README as a living document, not a one-time artifact.
