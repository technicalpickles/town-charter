# Session Handoffs

## Motivation

AI sessions are stateless. When a session ends, whether from a context limit, a closed terminal, or a task switch, the context disappears. The next session starts cold: what was I working on, where did I leave off, what decisions were made, which files are relevant? Reconstructing that state takes time and is error-prone. Details get lost. Work that was mid-stream has to be rediscovered.

Handoffs are the persistence layer for session context. They are written when stopping and read when resuming. The cost of writing a handoff is a few minutes at the end of a session. The benefit is that the next session, whether tomorrow or next week, starts with full context instead of nothing.

## Mental Model

A handoff is a structured markdown document that captures enough context to resume work. It is written from the perspective of someone explaining a task to a fresh pair of eyes: not a log of everything that happened, but a focused summary of where things stand and what to do next.

A useful handoff contains:

- What you were working on, stated plainly
- The current state: what is done, what is in progress, what is stuck
- What to do next, in enough detail that a fresh session can proceed
- Which files and locations are relevant
- Any decisions made, context gathered, or gotchas discovered

Handoffs are date-prefixed for chronological ordering. `2026-03-31-payment-service-migration.md` is easier to find and sort than `payment-service-migration-3.md`. The date also signals freshness: a handoff from three months ago may describe a state that no longer exists.

A handoff lives either in a project's `handoffs/` directory (if the work belongs to a project) or in a general `docs/handoffs/` directory in the workspace. Either way, it is a versioned file, not a chat message or a local note that disappears when the terminal closes.

## Capabilities

A handoff implementation supports:

- **Create a handoff**: Given a topic and current context, produce a structured handoff document with the date prefix. The writing itself can be done by an AI assistant as one of its last acts in a session.

- **Find handoffs**: Locate handoffs for a given project or work unit. Given a project name or a work tracking ID, find the relevant handoff documents sorted by date.

- **Resume from a handoff**: At the start of a new session, read the most recent handoff for a piece of work and reconstruct the working context. An AI assistant reading a well-structured handoff should be able to orient itself without additional archaeology.

## In Practice

Say you are implementing authentication changes in a backend service. Midway through the session, you hit the context limit. Before the session ends, the assistant writes a handoff:

```markdown
# Handoff: Auth Token Refresh

Date: 2026-03-31
Status: In progress

## What we were doing

Implementing token refresh in `AuthService`. The new `refresh_token` method is
partially written in `app/services/auth_service.rb`. The logic for expiry
checking is done; the part that writes the new token back to the session is not.

## Current state

- `refresh_token` method exists but is incomplete (see line 84)
- Tests in `spec/services/auth_service_spec.rb` are failing because the method
  raises NotImplementedError
- The token storage interface is in `app/models/session_token.rb`, already read

## Next steps

1. Finish the `refresh_token` method: call `session_token.rotate!` with the new
   expiry value
2. Make the failing tests pass
3. Add a test for the expiry edge case (token expires exactly at request time)

## Decisions made

- Using the existing `rotate!` method rather than creating a new one
- Expiry window is 5 minutes, set in `config/auth.yml` under `token.refresh_window`
```

The next session opens this file. In under a minute, the assistant knows exactly where to start: open `auth_service.rb` at line 84, finish the `refresh_token` method, run the tests. No reconstruction. No guessing.

## Design Considerations

**Optimize for fast creation.** Handoffs are write-heavy and read-occasionally. The format should be easy to produce quickly, not beautiful. A rushed but complete handoff is more valuable than a polished one that never gets written. When in doubt, write more rather than less: the next session can skip context it already has, but it cannot recover context that was never written down.

**Consistent structure makes handoffs machine-readable.** An AI assistant parsing a handoff should be able to locate the next steps without reading the entire document. Using consistent section headings, like "Current state" and "Next steps," makes this reliable. The structure in the example above is a reasonable default, but any consistent format works.

**Handoffs degrade.** A handoff written three months ago describes a context that has changed: files may have moved, decisions may have been revisited, the next steps may already be done. Date them and treat them as perishable. When resuming work after a long gap, read the handoff for orientation but verify its claims before acting on them.

**Handoffs complement work tracking, they do not replace it.** A work tracking item captures what needs to happen. A handoff captures the immediate session state: where in the middle of a task you are, what you just found out, what the next concrete step is. Both have value. Use them together.

**Project-scoped versus general.** When work belongs to a project, the handoff lives in that project's `handoffs/` directory. When work is more ad-hoc or does not have a project, a general `docs/handoffs/` directory in the workspace serves as a fallback. Either location works as long as handoffs are findable by date and topic.
