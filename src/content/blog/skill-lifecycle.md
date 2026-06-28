---
title: 'Composable Skills: Building an AI Assistant That Chains Capabilities'
description: 'How to design modular skills that trigger from other skills, creating compound workflows without fragile orchestration.'
pubDate: 'Jun 28 2026'
---

An AI assistant that can do one thing well is useful. An AI assistant that chains capabilities together, where one skill triggers another without manual intervention, is transformational.

The difference is composability. A skill system where each capability is a standalone script works for simple tasks. But real workflows span multiple capabilities: a weekly review reads project notes, updates priorities, generates a focus document, and syncs tasks to a reminder system. If each step requires the user to invoke a separate command, the system is just a menu of tools. If the skills chain together, the user says "do my weekly review" and the system handles the rest.

## What a skill is

A skill is a self-contained prompt with a trigger condition, a set of instructions, and an expected output. In practice, it looks like a markdown file that the assistant loads when a trigger matches:

```markdown
# Weekly Review

Trigger: /weekly-review or Monday morning session-start

## Steps
1. Read last week's active_focus.md
2. For each project in 1_Active/Projects/, check for updates
3. Capture what happened, what shifted, what stalled
4. Draft the new week's active_focus.md
5. Surface any overdue tasks
```

The skill file defines what the assistant should do, in what order, and what to produce. It does not contain the logic itself, just the instructions. The assistant's language model handles execution by reading files, making decisions, and writing outputs.

## The chaining rule

A skill that can only fire via manual invocation is a dead end. The design rule: every new skill must be woven into an existing skill's trigger chain. If a skill cannot chain from anything, reconsider whether it should exist as a standalone.

In practice, this means skills reference each other:

- **Session start** checks energy level, surfaces priorities, and can trigger **triage** if overdue tasks exceed a threshold.
- **Weekly review** reads project status notes and triggers **capture** for any insights worth saving.
- **Wrapup** reviews the conversation for anything worth persisting to memory, updates relevant files, and flags follow-up tasks.
- **Close project** audits tasks, captures lessons learned, evaluates artifacts for reuse, and updates the project dashboard.

The chain creates compound behaviors: starting a Monday session could automatically surface last week's review, flag stale projects, and queue the most important task for the current energy level, all from a single trigger.

## Lifecycle stages

Skills mature through a predictable lifecycle:

**1. Manual invocation.** The skill exists as a slash command. The user types it, the assistant executes it. This is the testing phase: does the skill produce useful output? Are the instructions clear enough?

**2. Triggered by another skill.** The skill gets woven into an existing workflow. Session start calls it, weekly review references it, or the pipeline orchestrator schedules it. The user no longer needs to remember it exists.

**3. Self-triggering.** The skill monitors a condition and fires when the condition is met. A triage skill that activates when overdue tasks exceed five items. A memory grooming skill that fires during monthly review when memory count exceeds a threshold.

**4. Retired or merged.** Skills that serve the same purpose get merged. Skills that no longer fire get archived. The skill library should shrink over time as patterns consolidate, not grow indefinitely.

Most skills never reach stage 3. That is fine. The goal is not full autonomy for every capability but seamless handoffs between capabilities that together cover the user's workflows.

## Designing for low friction

The hardest problem in a skill system is activation energy. A skill that requires the user to remember its name, know when to invoke it, and provide the right arguments will not get used. Design for zero-input invocation:

**Default to action.** A session start skill should not ask "what do you want to work on?" It should read the priority list, check the calendar, assess available time, and recommend. The user redirects if needed.

**Match tasks to energy.** If the system tracks energy levels (even a simple low/medium/high self-report), skills can filter their suggestions. Low energy: surface quick tasks. High energy: propose the deep work item at the top of the priority stack.

**Size everything.** Every task should carry an effort tag: quick (under 15 minutes), medium (15 to 60 minutes), or deep (over an hour). When a skill surfaces work, it includes the size so the user can make a fast decision without reading the full task description.

## The skill file structure

A production skill file includes:

```markdown
---
name: weekly-review
trigger: /weekly-review, Monday session-start
chains_from: session-start
chains_to: capture, triage
---

# Weekly Review

[Instructions...]
```

The frontmatter makes dependencies explicit. `chains_from` documents what triggers this skill. `chains_to` documents what this skill can invoke. When auditing the skill library, you can trace the full chain from any entry point.

## Avoiding the wrong abstractions

The temptation is to build a skill framework: a registry, a dispatch layer, a dependency resolver. Resist this. Skills are prompts, not code. The assistant's language model is the runtime. A skill "executes" when the assistant reads its instructions and follows them.

What you need is simpler:

1. A directory of skill files the assistant can read.
2. A naming convention so the assistant knows which skill matches a trigger.
3. Instructions in each skill that reference other skills by name when chaining.

The assistant handles routing ("the user said /weekly-review, let me load that skill"), execution (reading files, making decisions, writing output), and chaining (the weekly review instructions say to run triage if overdue count is high). No orchestration framework required.

This keeps the system maintainable. Adding a skill means adding a markdown file. Modifying a skill means editing prose. Debugging a skill means reading the conversation where it executed and seeing what instructions the assistant followed. Compare this to debugging a plugin system with hooks, middleware, and event buses.

## When skills fail

Skills fail when their instructions are ambiguous, when they reference files that have moved, or when they chain to skills that no longer exist.

The fix is the same for all three: audit the skill library during periodic reviews. Check that every `chains_to` reference points to a skill that still exists. Check that file paths in instructions still resolve. Check that the skill's output matches what downstream skills expect as input.

A monthly review that includes a skill audit catches these problems before they compound. A quarterly review should question whether each skill still earns its place in the library.

The goal is a small set of well-maintained skills that chain smoothly, not a large catalog that mostly goes unused. Five skills that cover daily workflows are worth more than fifty that each handle one edge case.
