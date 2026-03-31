# Starting New Work

## Scenario

You have a bug to fix or a feature to build. You know what you need to do. Now you need to go from "I need to work on X" to actually editing code.

---

## Without the Town

You open your issue tracker and create a ticket. You fill in a title, maybe a description. You assign it to yourself. You copy the ticket number somewhere so you can reference it later.

Now you need a branch. You find the repo, make sure you are on main, pull the latest. You create a branch. You pick a name. The name should match the ticket somehow but you are doing this from memory because you already closed that tab. You push the branch to set the upstream.

Now you need a working area. If you use worktrees, you run `git worktree add` with a path you are inventing on the spot. If you clone, you clone into some directory and immediately forget whether you named it for the ticket or the feature or the repo. You make a note somewhere that this directory is for this ticket, because the connection is not obvious and you will not remember.

You have three things that belong together, created in three different places, connected only by the notes you remembered to take.

You open the directory. You are finally ready to write code.

---

## With the Town

You tell the town what you are working on and which repo it belongs to. One operation.

The town creates the tracking item, the branch, and the working area together as a single unit. It names the branch from the tracking ID so the connection is structural, not a note you have to maintain. The working area lands in the predictable location the workspace defines for that repo.

You are dropped into the directory, and the code is in front of you.

---

## Concepts Involved

**[Work Tracking](../concepts/work-tracking.md)**: The tracking item, branch, and working area are created together as a work unit. The relationship between them is not something you have to manage; it is encoded in how they are named and where they live.

**[Workspace](../concepts/workspace.md)**: Working areas land in the standard location the workspace defines for that repo. You do not invent a path; you navigate to a predictable one.

**[CLI Patterns](../concepts/cli.md)**: A single call creates all three components and returns you a path. You do not have to run three commands, copy IDs between them, or make notes connecting the pieces.
