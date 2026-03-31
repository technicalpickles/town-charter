# Town Charter

A spec for AI-assisted multi-repo development workspaces.

---

## What This Is

A town is an AI-assisted development workspace: a single directory that tracks multiple repositories, organizes work, and provides conventions that help AI assistants collaborate effectively with humans.

Town Charter documents the concepts and workflows that make this work. These are patterns that reinforce each other, not a framework or template. You can adopt them incrementally. Each concept is useful on its own, and they compound when combined.

This spec targets anyone building or configuring a town: humans designing their workspace, and AI assistants operating within one.

---

## Origin

Gastown (Steve Yegge's framework for AI-assisted development) introduced the idea of a git-managed workspace directory alongside your code repositories, with task tracking and structured session management as first-class concerns. Pickletown rebuilt that foundation with a simpler focus: convention-driven AI collaboration over infrastructure complexity, letting the directory structure and naming conventions carry most of the weight. Town Charter extracts the patterns that proved durable into a standalone spec that any implementation can follow.

---

## Philosophy

**Human-directed AI.** The human decides what work to do and when it is done. AI executes, suggests, and drafts. The workspace structure keeps that boundary visible: work units have statuses the human controls, and session handoffs preserve context without making decisions.

**Convention over configuration.** Where you put things matters more than configuration files. A file in the right place with the right name communicates intent to both humans and AI without any additional setup.

**Determinism over improvisation.** AI assistants perform better when they have clear, consistent conventions to follow. Ambiguity leads to drift. This spec defines conventions precisely so implementations can be predictable.

**The town as the unit of organization.** Code lives in repositories. Work, context, and conventions live in the town. The separation keeps repositories clean and makes the workspace itself a durable artifact.

---

## Concepts

Concepts are the building blocks. Core concepts have no dependencies. Extensions build on the core.

### Core

- **[Workspace](concepts/workspace.md)**: The root directory that contains everything: repository references, working areas, work tracking, and AI configuration. This is the town itself.

- **[Work Tracking](concepts/work-tracking.md)**: Knowing what you're working on, why, and what state it's in. A work unit ties together four things: a tracking item, a branch, a working area, and a review. Operating on work units as a whole is the key insight.

### Extensions

Each extension has its dependencies noted.

- **[Projects](concepts/projects.md)** (depends on: Workspace): When work spans multiple repositories or stretches across sessions, it needs a home. A project is a folder with structured subdirectories that lives in the town.

- **[AI Conventions](concepts/ai-conventions.md)** (depends on: Workspace): Teaching your AI assistant how the town works through rules files and project-level instructions. Write conventions once; the AI applies them at every session start.

- **[Session Handoffs](concepts/session-handoffs.md)** (depends on: Workspace, optionally Projects): Structured documents that capture context when work stops mid-stream so you or an AI can resume later.

- **[CLI Patterns](concepts/cli.md)** (depends on: Workspace, Work Tracking): The town benefits from a CLI that operates on work units as a whole. Covers why a CLI exists (determinism, efficiency, single-entrypoint operations) without prescribing specific commands or syntax.

---

## Workflows

Workflows show concepts in action. Each workflow describes a complete, concrete scenario. Implementations can use these as acceptance criteria.

- **[Starting New Work](workflows/starting-new-work.md)**: From "I need to work on X" to editing code. Create a work unit, set up a working area, start developing.

- **[Resuming Work](workflows/resuming-work.md)**: Return to in-progress work. Find the working area, check the status, restore context.

- **[Tracking a Repo](workflows/tracking-a-repo.md)**: Add an external repository to the town with conventions applied automatically.

- **[Switching Context](workflows/switching-context.md)**: Move between different pieces of work without stashing or losing state.

- **[Closing Out Work](workflows/closing-out-work.md)**: PR merged, time to clean up. Verify completion, update tracking, remove the working area.

---

## How to Use This Spec

### For AI assistants

Start with the core concepts: Workspace and Work Tracking. These define the structure you will operate in. Then read the concepts relevant to the current task. If you are writing a handoff, read Session Handoffs. If you are helping design a CLI, read CLI Patterns.

Use the workflows as a checklist. When a user asks you to start work, follow the Starting Work workflow. When they ask you to wrap up, follow Finishing Work.

When in doubt, prefer the conventions in this spec over improvised approaches. Consistency matters more than cleverness.

### For humans

Read the Philosophy section first. The rest of the spec implements those four ideas.

Browse the concepts that interest you. Each one is self-contained. You do not need to adopt all of them to get value from the spec.

If you are building a town from scratch, start with the Workspace concept and the Starting Work workflow. Those two will show you enough structure to make good decisions about the rest.
