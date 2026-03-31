# AI Conventions

## Motivation

Every AI assistant session starts cold. The assistant has no memory of previous sessions, no knowledge of your preferences, and no awareness of the patterns your team has developed. So you correct it. "Don't use `git add .`." "Check the rules directory before starting." "Use absolute paths in bash commands." The assistant follows the correction for the rest of the session, then forgets it by the next one.

These corrections are not one-time fixes. They are recurring costs. Each correction burns tokens and attention that could go toward actual work. And the corrections accumulate: as you develop sharper intuitions about how to work effectively with AI, the gap between what the assistant does by default and what you actually want grows wider.

AI conventions solve this by making your preferences durable. Instead of correcting the assistant, you write the correction down once. The assistant reads it at the start of every session, everywhere, automatically.

## Mental Model

An AI conventions system is a directory of rule files inside the workspace. Each file covers one topic. The assistant loads them at session start, before doing any work.

The directory lives in the workspace rather than in any individual repository. That placement is deliberate: conventions apply across all tracked repos. A rule about git staging behavior applies whether you are working in an API repo, a frontend repo, or a shared library. You write it once in the workspace; it applies everywhere.

Rules accumulate incrementally. You do not design them upfront. You write a new rule when you notice friction: when you correct the assistant for the second or third time on the same thing, that pattern is worth capturing. Over time the rule set reflects your actual working style, built from real experience rather than speculation.

This is the town's institutional memory for AI collaboration. The workspace carries the rules into every session, into every repository, across every tool that reads from the same directory.

## Capabilities

An AI conventions implementation provides:

- **Automatic loading at session start**: The assistant reads conventions before doing any work. You do not invoke them explicitly. They are always active.
- **Workspace scope**: Conventions apply across all tracked repositories. A rule written once applies everywhere.
- **Incremental authoring**: Rules can be added or edited at any time. New conventions take effect at the next session without any reconfiguration.
- **Per-topic files**: Each file covers one concern. The structure keeps rules findable and makes it easy to update one rule without touching others.

## In Practice

Say you notice that the AI keeps using `git add .` when staging changes. You have corrected it twice in the past week. The third time, instead of correcting it again, you create a file in the workspace conventions directory: `git-staging.md`. In it, you write that `git add .` is prohibited, that you stage files individually by name, and why: it avoids accidentally including sensitive files or large binaries.

The next session, the assistant reads that file. It stages files individually without being asked. You never make that correction again.

A month later you notice the assistant sometimes creates documentation files without being asked. You add a rule: only create documentation when explicitly requested. A week after that, you formalize a rule about using absolute paths in bash commands after a confusing session in a deeply nested directory.

The rule set grows to reflect your working style. Each rule is a correction you made once and will never need to make again. The compounding effect is real: a set of twenty rules represents twenty recurring costs that are now zero.

When you work in a new repository, the conventions are already there. The assistant arrives oriented rather than blank.

## Design Considerations

**Rules should be prescriptive and actionable, not philosophical.** "Prefer clarity" is not a rule; the assistant cannot act on it. "Use absolute paths in all bash commands, never relative paths" is a rule. Good conventions read like standing instructions: specific enough that following or violating them is unambiguous.

**The backing mechanism is AI-tool-specific.** Claude Code reads conventions from `.claude/rules/` in the workspace. Other AI tools have their own mechanisms: system prompts, configuration files, context injection at startup. The pattern (a directory of rule files, one file per topic, loaded at session start) is portable as a concept even when the exact path or format varies.

**One file per topic makes maintenance tractable.** A single large conventions file becomes hard to update and hard for the assistant to navigate. Separate files let you edit one rule without touching others, see at a glance what topics are covered, and add new rules without deciding where they fit in a monolith.

**Write rules when you feel friction, not before.** Upfront rule design produces generic advice. Rules written after repeated corrections are specific and grounded. The best signal for a new rule is correcting the assistant on the same thing twice.

**Conventions degrade if the workspace does.** Rules written for one workflow can become stale or contradictory as the workflow evolves. Treat the conventions directory as a living system: review and prune when a rule no longer reflects what you actually want.
