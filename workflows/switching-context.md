# Switching Context

## Scenario

You are mid-task on one thing when something urgent comes in on a different repo. You need to drop what you are doing and address it, then come back to where you were.

---

## Without the Town

You have uncommitted changes. You either stash them or make a "wip" commit you will have to remember to undo later. You note where you were, because when you come back this will not be obvious.

You find the other repo. You find the branch, or create one. You do the work.

Now you have to come back. You find your original repo and branch. You un-stash or un-commit. You figure out where you were. Some context has leaked out of your head in the meantime. You spend a few minutes reconstructing.

If the urgent thing stretched across more than one session, you have probably lost more than a few minutes.

---

## With the Town

Each piece of work has its own isolated working area. The urgent work has one. Your in-progress work has another.

Switching means navigating to a different directory.

Your in-progress changes stay exactly where they are. Uncommitted, unstashed, exactly as you left them. The directory does not know or care that you went somewhere else for a while.

When you come back, you navigate back. Nothing to reverse. Nothing to reconstruct.

---

## Concepts Involved

**[Workspace](../concepts/workspace.md)**: Each working area is isolated. Changes in one do not affect others. The workspace structure is what makes it safe to leave work in place while you go handle something else.

**[Work Tracking](../concepts/work-tracking.md)**: Each working area is tied to a tracked work unit. When you come back, you are not hunting for the right branch or remembering which directory held which feature. You navigate by the work unit.
