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

- **[Work Tracking](concepts/work-tracking.md)**: A file-based system for tracking tasks, features, and bugs as work units. Work units live alongside the code they describe, in the workspace, under version control.

### Extensions

Each extension has its dependencies noted.

- **[Projects](concepts/projects.md)** (depends on: Workspace, Work Tracking): A project is a named subdirectory for work that spans multiple repositories or sessions. It holds plans, designs, and session handoffs for a coherent body of work.

- **[AI Conventions](concepts/ai-conventions.md)** (depends on: Workspace): The files and directory conventions that give AI assistants context: where to find instructions, how to discover active work, and what patterns to follow.

- **[Session Handoffs](concepts/session-handoffs.md)** (depends on: Work Tracking, AI Conventions): Structured documents that preserve context across sessions. A handoff captures where you are, what you were doing, and what comes next, so work can resume without reconstruction.

- **[CLI Patterns](concepts/cli-patterns.md)** (depends on: Workspace, Work Tracking): Conventions for building a CLI that wraps common workspace operations. Covers command naming, ref resolution (accepting work unit IDs, PR numbers, or branch names interchangeably), and composing operations that span repositories.

---

## Workflows

Workflows show concepts in action. Each workflow describes a complete, concrete scenario. Implementations can use these as acceptance criteria.

- **[Starting Work](workflows/starting-work.md)**: Create a work unit, check out a branch, and open a working area ready for development.

- **[Resuming Work](workflows/resuming-work.md)**: Return to an in-progress work unit, locate its working area, and restore context from a handoff.

- **[Finishing Work](workflows/finishing-work.md)**: Verify a work unit is complete, create a pull request, and clean up the working area after merge.

- **[Daily Orientation](workflows/daily-orientation.md)**: Survey active work units, open pull requests, and recent changes to understand the current state of a town.

- **[Parking Work](workflows/parking-work.md)**: Write a session handoff for in-progress work so it can be resumed cleanly in a future session.

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
