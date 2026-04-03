# CLAUDE.md

This is a prose spec, not application code. All content is markdown. There is nothing to build or run.

## File Organization

- `spec.md` is the hub. It links to every concept and workflow, contains the philosophy and origin story, and provides reading guidance. When you add or rename content, update spec.md.
- `concepts/` contains the building blocks. Each file is self-contained. Core concepts (Workspace, Work Tracking) have no dependencies. Extensions note their dependencies at the top of their spec.md entry.
- `workflows/` contains concepts in action. Each file describes a complete scenario that shows how concepts work together.
- `README.md` is the GitHub landing page. It is short and points to spec.md.

Non-content files: `hk.pkl` (pre-commit hooks), `mise.toml` (tool versions), `.markdownlint-cli2.yaml` (lint config), `.lychee.toml` (link checker config).

## Adding a Concept

1. Create `concepts/<name>.md`
2. Use these sections in order: Motivation, Mental Model, Capabilities, In Practice, Design Considerations
3. Add an entry to `spec.md` under Core or Extensions, noting dependencies
4. Link to related concepts using relative paths with the concept's proper title as link text: `[Work Tracking](work-tracking.md)`

## Adding a Workflow

1. Create `workflows/<name>.md`
2. Use these sections in order: Scenario, Without the Town, With the Town, Concepts Involved
3. Add an entry to `spec.md` under Workflows
4. Reference concepts using relative paths from workflows/: `[Session Tracking](../concepts/session-tracking.md)`

## Pre-commit Hooks and Tooling

hk runs two checks on pre-commit: markdownlint (with auto-fix) and lychee (link checking). mise manages tool versions for both.

After a fresh clone or worktree, run `mise trust` before hooks will work.

If a hook fails, fix the issue before committing.

## Writing Voice

Read `concepts/workspace.md` and `concepts/work-tracking.md` for the tone and style. Match them.
