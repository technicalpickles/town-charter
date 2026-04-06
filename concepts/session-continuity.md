# Session Continuity

## Motivation

AI sessions are stateless. When a session ends, whether from a context limit, a closed terminal, or a task switch, the context disappears. The next session starts cold: what was I working on, where did I leave off, what decisions were already made? Reconstructing that state takes time, and details get lost every time.

Some AI tools offer native continuity mechanisms. Resuming a previous session rehydrates the full conversation history. Context compaction summarizes earlier exchanges to free up space. These help for short interruptions, but they have limits. A resumed conversation carries all its accumulated context: false starts, abandoned approaches, tangents that no longer matter. A compacted context is a lossy summary the tool produces automatically, not a curated capture of what actually matters.

Session continuity in the town is deliberate. You decide what to carry forward and what to leave behind. The result is a fresh session that starts with exactly the context it needs, clean and free of the noise that accumulated in the previous one.

## Mental Model

Continuity operates through three mechanisms:

- **Handoffs**: structured documents written when work stops mid-stream. A handoff captures what you were doing, what state things are in, what to do next, and what files or locations matter. It is written for a fresh pair of eyes: not a log of everything that happened, but a focused summary of where things stand. Handoffs live in the town as versioned files, not as chat messages or local notes that disappear.

- **Parking**: marking a session as stopped-but-not-done. A parked session has a handoff attached and a clear signal that the work is incomplete. Parking is sometimes forced (you hit the context limit, or it is the end of the day) and sometimes intentional (the context has gotten noisy and you want a fresh start with just the essentials).

- **Continuing**: starting a new session that picks up from a parked one. The new session reads the handoff, inherits the associations from the previous session (which [work units](work-tracking.md) were involved, which repos were active), and starts oriented rather than cold. If [Session Tracking](session-tracking.md) is in use, the continuation event creates a traceable chain from old session to new.

These three mechanisms form a cycle: work, park with a handoff, continue into a fresh session, keep working. Each cycle preserves context across the stateless boundary while shedding accumulated noise.

## Capabilities

A session continuity implementation supports:

- **Capture context**: Given a topic and current state, produce a structured handoff document with a date prefix. The writing can be done by an AI assistant as one of its last acts in a session, or by a human. The structure should be consistent enough that a new session can locate the next steps without reading the entire document.

- **Park a session**: Record that a session has stopped, with a reference to the handoff document. If [Session Tracking](session-tracking.md) is available, this writes a park event to the session history.

- **Continue from a parked session**: Start a new session that picks up from a previous one. Read the handoff, orient to the working area and [work unit](work-tracking.md) state, and proceed. If Session Tracking is available, the continuation creates an explicit link between the old and new sessions.

- **Find handoffs**: Locate handoffs for a given project or [work unit](work-tracking.md). Given a project name or a work tracking ID, find the relevant handoff documents sorted by date.

## In Practice

You are three hours into implementing a feature. The AI assistant's context is heavy: exploration of three approaches (two abandoned), a debugging tangent, several files read but not relevant to the final direction. The code is in good shape. The conversation is not.

Rather than pushing through with degraded context, you park. The assistant captures the current state into a handoff:

```text
Status: In progress
Work unit: gt-ab12

What we were doing: Implementing token refresh in AuthService.
The expiry checking logic is done. The part that writes
the new token back to the session store is not.

Current state:
- refresh_token method exists but is incomplete (line 84)
- Tests are failing (expected, method raises NotImplementedError)
- Token storage interface in session_token.rb, already read

Next steps:
1. Finish refresh_token: call session_token.rotate! with new expiry
2. Make failing tests pass
3. Add expiry edge case test

Decisions made:
- Using existing rotate! method rather than creating a new one
- Expiry window is 5 minutes, set in config/auth.yml
```

You continue into a fresh session. The new session reads the handoff. In under a minute, the assistant knows exactly where to start: open `auth_service.rb` at line 84, finish the method, run the tests. No reconstruction. No inherited noise from the abandoned approaches or the debugging tangent.

Another scenario: you hit the context limit mid-task. The assistant writes a handoff as its last act. You continue the next day. The handoff tells the new session exactly where you left off. The gap between sessions is invisible.

For a concrete implementation of this concept, see the [Pickletown example](../examples/pickletown/README.md#session-continuity).

## Design Considerations

**Optimize for fast creation.** Handoffs are written often and read occasionally. The format should be easy to produce quickly. A rushed but complete handoff is more valuable than a polished one that never gets written. When in doubt, capture more rather than less: the next session can skip context it does not need, but it cannot recover context that was never written down.

**Consistent structure makes handoffs machine-readable.** An AI assistant parsing a handoff should be able to locate the next steps without reading the entire document. Consistent section headings ("Current state," "Next steps") make this reliable. The structure in the example above is a reasonable default, but any consistent format works.

**Handoffs degrade over time.** A handoff from three months ago describes a context that has likely changed: files moved, decisions revisited, next steps already completed. Date them and treat them as perishable. When continuing after a long gap, read the handoff for orientation but verify its claims against the current state before acting.

**Parking is distinct from capturing.** A handoff is a document. Parking is a state change: it marks a session as stopped and connects it to the handoff. You can capture context without parking (a summary for your own reference). You cannot meaningfully park without a handoff (there would be nothing to continue from). The separation keeps both ideas clean.

**Continuity compounds with tracking.** Without [Session Tracking](session-tracking.md), continuity still works: you capture context, you continue from it. With Session Tracking, the continuation event creates an explicit chain, and the associations from the previous session carry forward. The two concepts are independent but reinforce each other.

**Town continuity complements native tool continuity.** AI tools that offer resume or compaction handle within-session and light between-session continuity. Town-level continuity handles the heavier case: intentional context resets, multi-day gaps, transitions between different AI tools or environments. They work at different layers and do not conflict.
