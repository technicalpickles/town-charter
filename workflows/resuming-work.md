# Resuming Work

## Scenario

You worked on something a few days ago. Maybe a week. You got pulled away mid-stream, or you finished a review cycle and are now acting on feedback. The work is not done. You need to pick it back up.

You remember the rough shape of it. Not the details.

---

## Without the Town

You open a terminal. You need to find the branch.

You run `git branch` in the repo you think it is. The list is long. You scan for something that matches your memory of what you called this. You are not sure which of two names it might be. You check both. One looks right.

Now: where is this repo on disk? You cloned it at some point. You check a few likely places. You find it, but this is the clone you use for `main`. The feature branch was in a different clone, the one you made specifically for this work. You find that too, eventually, in a directory you named something sensible at the time but now requires a second of decoding.

You need to know what state the work is in. You open GitHub, find the repo, find pull requests. You search for the branch name. It comes up. You read the comments. There was a review round. Someone left three comments. You read them to figure out what was addressed and what was not.

You need to check the tracking item. You open your issue tracker. You search for the ticket. You find it, read the description, read the comments. The status says "in progress," which tells you nothing about where in progress.

You have now spent five to ten minutes on orientation. You have a rough picture. You still do not know exactly where you left off in the code.

You open the files you think are relevant. You scan for something that looks like a stopping point. Maybe you left a comment. Maybe the failing test is the breadcrumb. You piece it together.

You are finally ready to work. This whole process will happen again next time you pick this up.

---

## With the Town

You give the town a reference to the work: a tracking ID, a branch name, a PR number. Any of those works. The town resolves it to the full work unit.

The response tells you: the path to your working area, the current state of the tracking item, the PR status (open, approved, waiting on CI, whatever it is), and the most recent handoff if one exists. Everything you need to orient is in one place, ready immediately.

You navigate to the working area. If a handoff exists, you read it. It tells you exactly where you were: which file, which problem, what the next step is. If no handoff exists, the tracking item and PR comments give you enough.

You are oriented and working in under a minute.

---

## Concepts Involved

**[Work Tracking](../concepts/work-tracking.md)**: The core operation here is "resume." You provide any reference to the work unit, and the town finds the working area and surfaces the combined status of all four components: tracking item, branch, working area, PR. Without this, you are manually correlating four separate sources.

**[Workspace](../concepts/workspace.md)**: Working areas are in predictable locations because the workspace defines where they live. There is no hunting for the right clone or directory. The structure carries the answer.

**[Session Handoffs](../concepts/session-handoffs.md)**: When you stopped last time, you (or your AI assistant) may have written a handoff. It is a structured document with current state and next steps, written for exactly this moment. A good handoff turns resuming work from reconstruction into reading.

**[CLI Patterns](../concepts/cli.md)**: The resume operation is a single deterministic call. It does not improvise. It finds the working area, loads the context, and gives you a path. That consistency is what makes it fast and reliable, whether you are doing it yourself or an AI assistant is doing it on your behalf.
