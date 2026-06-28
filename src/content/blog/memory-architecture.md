---
title: 'Designing a Memory System for AI Assistants'
description: 'How to build persistent, file-based memory that survives across conversations and stays useful as it grows.'
pubDate: 'Jun 27 2026'
---

Most AI assistants forget everything between sessions. You start fresh every time, re-explaining context that was established hours ago. A memory system fixes this, but the naive approach (dump everything into a file) creates a different problem: a growing blob of unstructured text that consumes context window and becomes impossible to maintain.

Here is an architecture that scales: typed memories stored as individual files, indexed by a lightweight manifest that loads at session start.

## The core structure

Memory lives in a directory with two layers:

1. **An index file** (e.g., `MEMORY.md`) that loads into every conversation. Each entry is one line: a link to a topic file with a short hook describing what it contains. This is the table of contents the assistant scans to decide what's relevant.

2. **Individual topic files**, each containing one coherent piece of knowledge with structured frontmatter.

```
memory/
  MEMORY.md          # Index, loaded every session
  user_role.md       # "User is a data scientist"
  feedback_testing.md # "Use real databases, not mocks"
  project_migration.md # "Auth rewrite driven by compliance"
  reference_api.md   # "Bug tracker is Linear project INGEST"
```

The index stays small (under 200 lines is a good ceiling). Topic files hold the detail. This separation means the assistant gets broad awareness cheaply, then reads specific files only when relevant.

## Memory types

Not all memories serve the same purpose. Categorizing them makes it easier to decide what to save, how to use it, and when to retire it.

**User memories** capture who you are: role, expertise, preferences, constraints. These shape how the assistant communicates and what it assumes you already know. A senior engineer gets different explanations than a first-time coder.

**Feedback memories** record corrections and confirmations. When you say "don't mock the database in tests" or "yes, the single PR approach was right here," that's a feedback memory. These prevent the assistant from repeating mistakes or abandoning approaches you've validated. Structure them as: the rule, why it exists, and when to apply it.

**Project memories** track what's happening: who's doing what, why, by when. These decay fast, so they need context about motivation (not just facts) so future sessions can judge whether the memory is still relevant.

**Reference memories** point to external systems: "bugs are tracked in Linear project X" or "the oncall dashboard is at this URL." These prevent the assistant from asking questions it should already know the answer to.

## The frontmatter format

Each topic file uses a consistent structure:

```markdown
---
name: feedback-real-database-tests
description: Integration tests must hit a real database, not mocks
metadata:
  type: feedback
---

Integration tests must use a real database, not mocks.

**Why:** A prior incident where mocked tests passed but a production
migration failed. Mock/prod divergence masked the broken migration.

**How to apply:** When writing or reviewing test code that touches
database operations, always configure a real test database. Flag any
mock-based database tests for review.
```

The `name` field is a kebab-case slug used for cross-referencing. The `description` is what appears in the index and what the assistant uses to judge relevance. The `type` field drives different handling: user memories inform tone, feedback memories guide behavior, project memories provide context, reference memories answer lookup questions.

## What NOT to save

The biggest risk with memory is saving too much. Bloated memory wastes context window, introduces contradictions, and makes maintenance a chore. Good exclusion rules:

- **Code patterns and architecture:** These can be derived by reading the current codebase. Saving them creates stale snapshots that conflict with reality.
- **Git history:** `git log` and `git blame` are authoritative. Memory copies of "who changed what" rot immediately.
- **Debugging solutions:** The fix is in the code. The commit message has the context. Saving "how we fixed bug X" duplicates what already exists.
- **Ephemeral task details:** In-progress work, current conversation context, temporary state. These belong in task trackers, not long-term memory.

The test: if you could get this information by reading the current state of the system, don't save it. Memory is for things that aren't recorded anywhere else.

## Index management

The index file is the chokepoint. If it grows too large, it consumes context budget that should go to actual work. Strategies:

- **One line per entry, under 80 characters.** The index is a lookup table, not a knowledge base. Detail belongs in the topic files.
- **Merge related entries.** Five separate API integration memories become one entry pointing to a consolidated file.
- **Drop narrow technical details from the index.** A specific CSS bug fix doesn't need to be in the table of contents. The topic file still exists for search.
- **Regular grooming.** A monthly pass to update stale entries, merge duplicates, and remove memories that no longer apply.

## Memory hygiene

A memory system that only grows will eventually drown in its own noise. The harder discipline is knowing when a memory has outgrown its home and should graduate somewhere else, or be retired entirely.

The core principle: memory is for narrow, hard-won knowledge. If a rule applies universally across sessions and domains, it belongs in your instruction file (the persistent config the assistant loads every time). Memory is the wrong place for universal norms.

**The placement test:** When you're about to save a feedback memory, ask: "Would adding this to the instruction file make it longer without helping most sessions?" If the answer is no, if the rule genuinely helps across the board, put it in the instruction file. Memory is for the corrections that only fire in specific contexts.

Three tiers govern where rules live:

1. **Instruction file** (loaded every session): Universal behavioral norms, autonomy framework, file safety rules, design principles that apply across domains. "Always verify file paths before acting" belongs here.

2. **Memory** (loaded on demand): Domain-specific corrections, project-specific gotchas, technical knowledge that matters only in narrow contexts. "This API requires pagination tokens, not offsets" belongs here.

3. **Skills and project files** (loaded only inside one workflow): Rules that only matter during a specific process. A deployment checklist for one service doesn't need to occupy memory at all.

**Self-maintenance through graduation.** When a feedback memory gets referenced during a session and clearly applies universally, migrate it to the instruction file and retire the memory. This is the natural lifecycle: you learn something the hard way, save it as a memory, and eventually recognize it as a general rule. At that point, promoting it and removing the memory entry reduces index noise without losing the knowledge.

Memory count should trend down over time as the system matures, not up. A mature system has fewer, sharper memories because the universal lessons have graduated to the instruction file and the stale project-specific ones have been retired.

## Cross-referencing

Topic files can link to each other using a simple convention (wiki-style links or just naming the slug). This creates a lightweight knowledge graph: a feedback memory about deployment can reference a project memory about infrastructure, and a reference memory about API access can link to the project that uses it.

These links serve two purposes: they help the assistant follow context chains ("this feedback exists because of that project"), and they flag memories that might be affected when one changes.

## Verifying memories before acting

A memory that names a specific function, file, or flag is a claim about what existed *when the memory was written*. Code gets renamed, removed, or refactored. Before recommending something based on memory:

- If the memory names a file path, check that the file exists.
- If the memory names a function or flag, grep for it.
- If the user is about to act on the recommendation, verify first.

"The memory says X exists" is not the same as "X exists now."

## The practical payoff

A well-maintained memory system changes the assistant relationship from transactional to cumulative. Session 50 is qualitatively different from session 1: the assistant knows your preferences, understands your constraints, remembers past decisions and why they were made, and can build on prior work instead of rediscovering it.

The architecture here, typed files with an indexed manifest, handles the growth curve. Early on, everything fits in the index. As memory grows, the type system, grooming rules, and hygiene framework keep it manageable. A production system running this architecture reduced from 87 feedback memories to 40 by applying the graduation framework described above. The reduction actually improved assistant quality: fewer memories meant less noise in every session, faster index scanning, and cleaner signal when a memory did fire. Pruning was additive.

The key discipline: save what's non-obvious and hard to re-derive. Promote universal lessons to the instruction file. Retire what's stale. A memory system that only accumulates is worse than one that actively curates, because it buries signal in noise and makes every session slower.
