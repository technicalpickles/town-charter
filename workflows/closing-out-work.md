# Closing Out Work

## Scenario

Your PR got merged. The work is done. Time to clean up.

---

## Without the Town

You delete the local branch. You delete the remote branch, if you remember to. You find the working area and remove the directory. You open the issue tracker and update the ticket status. You check whether there are any related branches or directories you spun up along the way.

Each step is a separate command, a separate tool, a separate context switch. The steps are not hard, but they are easy to do partially. You forget the remote branch. You forget the working area. You update the ticket but miss changing the status. Six months later you have a graveyard of stale branches and orphaned directories that all made sense at the time.

---

## With the Town

You tell the town to close the work.

The town checks that the work is actually merged before doing anything. Then it updates the tracking item, removes the working area, and cleans up the branch. Everything that was created together gets removed together.

Nothing lingers.

---

## Concepts Involved

**[Work Tracking](../concepts/work-tracking.md)**: The close operation updates the tracking item as part of cleanup. The tracking item, branch, and working area were created as a unit; they are removed as a unit.

**[Workspace](../concepts/workspace.md)**: Because working areas live in predictable locations, the town knows exactly where to look. There is nothing to hunt down.

**[CLI Patterns](../concepts/cli.md)**: A single call handles the full cleanup sequence, including the verification step that confirms the work is merged before removing anything. You do not have to remember the order, and you do not have to worry about cleaning up prematurely.
