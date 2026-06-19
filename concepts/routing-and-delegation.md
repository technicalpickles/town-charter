# Routing & Delegation

## Motivation

A session that holds the wide, cross-project view is expensive to maintain and easy to wreck. It knows what is in flight across repositories, which work matters now, and how the pieces relate. That view is the whole reason the session is useful. Every time it dives into a single repository to do the actual digging, it fills its own context with detail that does not serve the next decision: file contents, build output, the back-and-forth of one bug. The coordinating view erodes a little with each dive.

Past a certain volume of active work, the scarcest resource is not compute or parallelism. It is the coordinating session's attention and context budget. The answer is to route bounded work elsewhere instead of doing it in place. The session decides where a piece of work goes, hands it off, and stays at altitude. Delegation here is context economy first and a speed-up second.

## Mental Model

A coordinating session routes bounded work to an isolated execution context, then keeps its own view clean. Three things make this work: the work is bounded, the executor is isolated, and the handoff is self-sufficient.

**The work is bounded.** A unit of delegated work maps to a single [work unit](work-tracking.md): one tracking item, one branch, one working area. Unbounded work cannot be handed off cleanly, because neither side can tell when it is done.

**The executor is isolated.** The delegate runs in its own context so it cannot silt or corrupt the coordinator. Isolation comes in degrees, and the degree you choose is the main decision. The execution modes form a spectrum along one axis: increasing isolation, decreasing supervision.

- **In-process.** Helpers inside the coordinating session itself, sharing its runtime, short-lived. Their results flow straight back into the coordinator's context. Use them for research, parallel reads, and surveys: work whose output the coordinator needs to absorb. Cheapest and least isolated.
- **Local isolated.** A separate session or process on the same machine, with its own scoped environment. Use it for bounded, real fieldwork in one repository. Because it runs in its own context, it can edit, build, and iterate without touching the coordinator's view.
- **Remote.** A detached agent running elsewhere that proceeds without supervision. Use it for work that should run while the coordinator does other things, where you are willing to review the result later rather than watch it happen. Most isolated, least supervised.

**The handoff is self-sufficient.** Whatever a town calls the artifact a coordinator hands a delegate, it must carry two things or the delegation fails: an absolute path to the working area, so the executor lands in the right place rather than somewhere in the repository, and a verification command, so the result can be checked rather than merely asserted complete. Scope and intent travel alongside these.

## Capabilities

A routing and delegation implementation supports three operations:

- **Route**: classify an incoming request. Does it need the wide view (handle it in the coordinating session), is it bounded fieldwork (delegate it), or is it uncertain (survey first, and let cheap recon pick the right executor)? Routing is the decision that keeps the coordinator from grabbing a shovel by reflex.

- **Delegate**: start the chosen execution context against one work unit, with a handoff that carries the working-area path and the verification command. The executor should be able to begin work immediately, without reconstructing where to work or what done looks like.

- **Verify**: confirm delegated work against its verification command before the coordinator treats it as complete. A delegate's report of success is a claim; the verification command is the evidence.

## In Practice

Say a request lands that touches three repositories. The coordinating session does not start editing. It first sends a few in-process helpers to map the work: what exists, where, and how much each repository needs. Two of the three turn out to need real changes; the third is already fine.

For each of the two, the coordinator cuts a handoff carrying the absolute path to that repository's working area and a verification command (the test or check that proves the change works). It delegates each to its own local isolated context. Now two pieces of fieldwork are underway in isolation, and the coordinating session is free: it can answer the next question, route the next request, or simply wait, all without its view silting up with the detail of either job.

When a delegate reports back, the coordinator runs the verification command before closing the work unit. Work that passes is done. Work that does not goes back out. The coordinator never had to hold both jobs' internals in its own context to keep them moving.

For a concrete implementation of this concept, see the [Pickletown example](../examples/pickletown/README.md#routing--delegation).

## Design Considerations

**The coordinator protects its context.** This is the point of the concept, not a side effect. Parallelism falls out of it for free, but the primary good is keeping the wide-view session uncluttered enough to keep making good routing decisions. An implementation that delegates for speed but lets results flood back into the coordinator has missed the motivation.

**The work unit is the unit of delegation.** Bounding delegated work to a single work unit is what makes a handoff clean and a result verifiable. This is why the concept depends on [Work Tracking](work-tracking.md): the work unit already bundles the tracking item, branch, working area, and review that a delegate needs. The [Workspace](workspace.md) provides the isolated working area the delegate operates in.

**The handoff must stand on its own.** A path plus a verification command, or the delegate lands lost and the result cannot be checked. This is the single most common failure mode: a vague handoff produces work in the wrong place, or work nobody can confirm. Treat the handoff as the contract.

**Survey before committing when unsure.** Cheap, low-isolation recon is how you choose the right execution mode instead of guessing. A short in-process survey that reveals the true shape of the work is worth more than a confident delegation to the wrong kind of executor.

**Match isolation to the work, not to habit.** In-process for what the coordinator must absorb, local isolated for fieldwork that would otherwise silt the view, remote for what can run unwatched. Over-isolating wastes setup; under-isolating defeats the purpose.

**This is human-directed routing, not automation.** A coordinating session routing work is not the same as an automated orchestrator that schedules and recovers agents on its own. The coordinator can be a human, or a session acting on a human's behalf, but the routing decisions stay legible and interruptible. Towns can grow toward more automation over time (see the [autonomy spectrum](../autonomy-spectrum.md)); this concept describes the routing itself, wherever you sit on that spectrum.
