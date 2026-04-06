# The Autonomy Spectrum

How much do you trust your AI assistant? The answer shapes everything about how you work.

Steve Yegge's [Welcome to Gas Town](https://steve-yegge.medium.com/welcome-to-gas-town-4f25ee16dd04) describes eight stages of developer progression in AI-assisted coding. They form a spectrum from "AI as autocomplete" to "AI as orchestrated workforce." The stages are useful framing for understanding where a town sits and what it needs.

## The Stages

**Stage 1: Near-zero AI.** Code completions, maybe asking a chatbot questions. The AI is a reference tool.

**Stage 2: Coding agent in IDE.** A sidebar agent with permissions enabled. You approve every tool use. The AI proposes, you decide.

**Stage 3: Agent in IDE, YOLO mode.** Trust goes up. You turn off permissions. The agent runs freely but you are watching.

**Stage 4: Wide agent in IDE.** The agent gradually fills the screen. You are reading diffs more than writing code.

**Stage 5: CLI, single agent, YOLO.** You have left the IDE. Diffs scroll by. You may or may not look at all of them.

**Stage 6: CLI, multi-agent, YOLO.** You regularly run three to five parallel instances. You are fast.

**Stage 7: 10+ agents, hand-managed.** You are pushing the limits of what you can coordinate manually.

**Stage 8: Building your own orchestrator.** You are automating the coordination itself. Agent colonies, recovery logic, workflow graphs.

## Where Towns Fit

The town patterns in this spec are most useful from stage 4 onward. That is where the volume of work, the number of active branches, and the frequency of context switches start to exceed what you can hold in your head. A workspace directory, structured work tracking, and session continuity become load-bearing infrastructure rather than nice-to-haves.

Gas Town targets stage 7-8. It is a full orchestration system: agent colonies, molecular workflows, crash recovery. The workspace structure is scaffolding for automated agents.

Pickletown started at stage 5 (single human-directed CLI agent) and has grown toward stage 6-7 over time. The workspace structure supports both the human and the agents, with conventions that make each session productive without requiring the previous session's context.

Town Charter captures what works across stages 4-8. The concepts are the same whether you have one agent or twenty. What changes is how much automation wraps around them: session tracking can be a manual note or an automated event log, work tracking can be human-managed or agent-triaged, handoffs can be written by hand or generated as a session's last act.

The spec does not prescribe where you should be on this spectrum. It gives you the building blocks that work wherever you are.
