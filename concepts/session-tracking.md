# Session Tracking

## Motivation

[Work units](work-tracking.md) track what needs to happen. Sessions are where it actually happens. An AI session starts, work gets done, the session ends. But by default, there is no record of any of it: no log of which sessions existed, what they touched, or how they relate to the work tracked elsewhere in the town.

Over time, this creates a gap. The town knows what work is active and what state it is in. It does not know when that work last got attention, how many sessions contributed to it, or what happened in sessions that did not produce commits or PRs. Sessions that explored an approach and abandoned it, investigated a bug without fixing it, or helped a colleague debug something in their repo: those are invisible.

Session tracking closes the gap. It records when sessions happen and connects them to the [work units](work-tracking.md) they touch. The result is a queryable history of activity across the town.

## Mental Model

A session is a bounded period of work. It starts when you (or an AI assistant) begin working and ends when you stop: close the terminal, hit context limits, switch tasks. Sessions happen inside the town, in a specific working area, on a specific branch. They may touch one [work unit](work-tracking.md) or several.

Session tracking records two things:

- **Lifecycle events**: when a session started, where (which working area, which branch), and how (fresh start, continued from a previous session, context reset). This is the timeline.
- **Associations**: connections between a session and the work it touched. A session might be associated with a [work unit](work-tracking.md), a branch, a review, a project, or a topic. Some associations are known at session start (you are in a working area that maps to a work unit). Others emerge during the session (you reference a work unit in conversation, open a PR, switch branches). Others are discovered after the fact by scanning session artifacts like transcripts.

The combination produces a graph: sessions connected to work units, ordered in time. You can traverse it in either direction: "show me sessions for this work unit" or "show me what this session touched."

## Capabilities

A session tracking implementation supports:

- **Record a session**: When a session starts, capture the timestamp, location (working area, branch), and source (how the session was initiated). This can be automated through hooks that fire at session start, or done manually.

- **Associate a session with work**: Connect a session to work units, branches, reviews, projects, or topics. Support both explicit association (the user or a tool declares a connection during the session) and discovered association (scanning session artifacts after the fact to find references that were not tracked in real time).

- **Query session history**: Given a time range, a work unit, a branch, or a repo, find the relevant sessions. Surface what each session was associated with.

## In Practice

You track three services in your town: an API, a frontend, and a shared library. Over a week, you work across several of them: a feature branch on the API, a bug fix on the frontend, some documentation updates in the town itself.

Each time a session starts, the town records it automatically. A hook fires, captures the session ID from the AI tool, notes the working directory and branch, and appends the event to the session log. When you start a session in `repos/api/worktrees/add-oauth/`, the town knows this session is in the `api` repo on the `add-oauth` branch. If that branch maps to a work unit, the association is recorded too.

During the API session, you mention a related work unit in conversation and open a PR. The assistant explicitly associates the session with the PR. Later, an association discovery pass scans the session transcript and finds a reference to the frontend bug fix work unit that was discussed but never explicitly tracked. That association gets filled in after the fact.

At the end of the week, you query sessions for the last seven days. The output shows each session: when it started, which repo it was in, what work units it touched, what the topic was. The API feature got four sessions across three days. The frontend bug was a single focused session. Two sessions were in the town itself, working on docs. You have a complete picture of where your time went, including sessions that produced no commits.

A colleague asks when a particular branch last got attention. You query sessions associated with that branch name. The answer comes back: three days ago, in a session that also touched two related work units. Context that would have taken several minutes of git archaeology comes back in seconds.

## Design Considerations

**Append-only is the right model.** Session events accumulate over time. They should not be edited or rewritten. An append-only log, whatever the format, keeps the implementation simple and the history trustworthy.

**Association discovery matters as much as explicit tracking.** Not every session will have clean metadata at start time. Sessions started from unfamiliar directories, quick context resets, or background agents may lack explicit work unit associations. The ability to discover associations after the fact, by scanning transcripts or other artifacts, keeps the history useful even when real-time tracking is incomplete.

**The backing store is not prescribed.** A JSONL file, a SQLite database, a directory of markdown files: any append-only store that supports the query patterns works. What matters is that lifecycle events and associations are recorded and queryable.

**Session identity comes from the AI tool, not the town.** Most AI tools assign session IDs. The town records those IDs rather than inventing its own. This keeps sessions traceable back to their source (transcripts, conversation history) without creating a parallel identity system.
