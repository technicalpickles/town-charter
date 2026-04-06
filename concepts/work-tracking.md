# Work Tracking

## Motivation

Every session starts with the same question: where was I? Without tracking, the answer is scattered across git branch names, PR pages, issue trackers, and memory. You piece together current state from four or five different places every time you pick up a task. The overhead compounds: more active work means more reconstruction, more opportunities for things to slip.

Work tracking gives you a single lens onto all active work. The goal is not to add process but to eliminate the cost of context reconstruction.

## Mental Model

A work unit bundles four things:

- **Tracking item**: A description of the work, its current status, and any notes about intent or progress. This lives in whatever issue or task system you use.
- **Branch**: The code for this piece of work, isolated from other branches.
- **Working area**: The directory where the branch is checked out and where you actually edit files.
- **Review**: The pull request or code review associated with the branch, if one exists.

These four things represent one piece of work. In practice, they are scattered across different tools with no shared reference. The key insight is to operate on the work unit as a whole: create all four together, check their combined status in one view, resume by jumping directly to the working area with full context, and close by verifying all four are in a completed state before cleaning up.

The lifecycle maps to four operations: **create**, **status**, **resume**, **close**.

## Capabilities

A work tracking implementation supports:

- **Create a work unit**: Given a description and initial state, create the tracking item, the branch, and the working area in one operation. You should be able to start editing code immediately after.

- **View combined status**: Show the current state of all four components together: what the tracking item says, what branch it is, what working area it maps to, and what the review status is. This replaces manually checking each system.

- **Resume a work unit**: Given any reference to the work unit (a tracking ID, a branch name, a PR number), find the working area and present the context needed to continue. You should land in the right directory with the right information without any manual lookup.

- **Close a work unit**: Verify that the review is merged, update the tracking item to completed, and remove the working area. One operation that covers all four components.

## In Practice

Say you have several pieces of work in flight. You open your terminal and check the status of all active work units. The output shows each one: the tracking item title, the branch, the location of the working area, and the current review status (open, approved, waiting for CI, merged). One of them has an approved PR with CI passing. You want to look at it before merging.

You resume that work unit using its tracking ID. The tool prints the path to the working area and shows you the relevant context: the tracking item description, any open comments on the PR, the current branch state. You are in the working area, ready to act, in one command.

When the PR merges, you close the work unit. The tool confirms the branch is merged, marks the tracking item completed, and removes the working area. Nothing lingers.

The four pieces stay connected throughout: you never need to remember which branch corresponds to which issue or which directory corresponds to which branch. The work unit carries all of it.

For a concrete implementation of this concept, see the [Pickletown example](../examples/pickletown/README.md#work-tracking).

## Design Considerations

**The backing store is not prescribed.** A work unit is an abstraction, not a specific tool. Implementations can use any tracking system: a directory of plain files, GitHub issues, a dedicated issue tracker, or something custom. What matters is that the four components of the work unit stay connected and can be operated on together.

**References should be flexible.** You should be able to refer to a work unit by a tracking ID, a branch name, or a PR number. Any of these should resolve to the full work unit. Requiring knowledge of the specific identifier format adds friction and breaks down when you arrive at a work unit through a different path (a PR notification, a mention in a message, a branch name you remember).

**Status belongs to the work unit, not to individual components.** A tracking item marked "in progress" with no working area is incomplete. A merged PR with a tracking item still marked "open" is misleading. The work unit's status is the composite of all four components. Implementations should surface that composite view rather than requiring you to check each piece separately.

**Isolation is load-bearing.** The working area maps to a single branch, and only that branch. This is what makes it possible to have multiple work units active simultaneously without interference. The branch isolation provided by the workspace concept (separate directories per branch) is what makes work units practical at any scale.
