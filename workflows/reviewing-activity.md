# Reviewing Activity

## Scenario

You want to understand what happened. Maybe it is the end of the week and you are writing a status update. Maybe you are picking up a work unit that has been quiet and want to know when it last got attention. Maybe a colleague asks what has been happening on a branch and you do not remember off the top of your head.

You need to answer questions like: what did I work on this week? Which sessions touched this work unit? When was this branch last active?

---

## Without the Town

You try to reconstruct activity from the artifacts it left behind.

`git log --since="last monday"` across each repo, one at a time. You scan commit messages and try to remember what session produced them. Some sessions did not result in commits. Those are invisible.

You check your PR activity on GitHub. That covers the review side but not the exploration, debugging, or planning sessions that produced no PRs.

You open your issue tracker and scan for recently updated items. Some updates were yours, some were automated, some were other people. Filtering takes time.

For anything that did not leave a commit, a PR, or a tracker update, you are relying on memory. Sessions where you explored an approach and abandoned it, where you investigated a bug without fixing it, where you helped a colleague debug something in their repo: those are gone unless you wrote them down somewhere.

The picture you assemble is incomplete and it took twenty minutes.

---

## With the Town

You query session history for the last week. The output shows every session: when it started, which repo it was in, what work units it touched, what the topic was. Sessions that produced commits sit alongside sessions that were pure exploration or investigation. Nothing is invisible.

You want to know about a specific work unit. You query sessions associated with its tracking ID. Four sessions come back, spanning three days. You can see when attention shifted away and when it came back.

A colleague asks about a branch. You query sessions associated with that branch name. The answer: last touched two days ago, in a session that also referenced a related work unit and a PR. Context that would have taken several minutes of git archaeology comes back in seconds.

The activity record is comprehensive because session tracking captures what happened at the session level, not just what left artifacts in git or GitHub.

---

## Concepts Involved

**[Session Tracking](../concepts/session-tracking.md)**: The foundation. Every query in this workflow runs against the session history: lifecycle events and associations recorded over time. Without session tracking, activity review falls back to reconstructing from artifacts.

**[Work Tracking](../concepts/work-tracking.md)**: Session associations connect to [work units](../concepts/work-tracking.md). The ability to query "which sessions touched this work unit" depends on both concepts working together.

**[Workspace](../concepts/workspace.md)**: The workspace structure makes session-to-repo associations automatic. A session started in `repos/api/worktrees/add-oauth/` is associated with the `api` repo and the `add-oauth` branch by convention, not by manual tagging.
