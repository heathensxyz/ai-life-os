---
title: 'Building a Hook System That Prevents Mistakes at the Tool Level'
description: 'How to use pre and post tool-use hooks to enforce safety rules mechanically instead of relying on prompt instructions.'
pubDate: 'Jun 28 2026'
---

Prompt instructions are suggestions. Hooks are laws.

When you tell an AI assistant "never use em dashes in markdown files," it will follow that rule most of the time. But context windows compress, attention drifts, and eventually a bad character slips through. The fix is not a stronger instruction. The fix is a gate that runs every time the assistant writes to a file, inspects the content, and blocks the write if it violates the rule.

This is a hook system: deterministic code that fires at specific points in the assistant's tool-use lifecycle, enforcing invariants that prompt-level instructions cannot guarantee.

## The three hook points

A useful hook system needs three events:

1. **PreToolUse**: fires before a tool executes. Can inspect the tool name and input, then allow, deny, or modify. This is where write guards live.

2. **PostToolUse** (or Stop): fires after the assistant produces output. Can inspect the response for quality signals, character violations, or behavioral drift.

3. **UserPromptSubmit**: fires when a new user message arrives. Can inject context, set reminders, or prime the assistant's next response.

Each hook is a shell script that receives the event payload on stdin as JSON and communicates back by writing JSON to stdout. The contract is simple: exit 0 with no output means "allow." Exit 0 with a JSON deny payload means "block this action." Exit non-zero means "hook crashed, fail open."

## Fail-open by design

The most important design decision in a hook system is failure behavior. A hook that crashes should never block the assistant from working. A buggy write guard that denies everything would brick the entire system, preventing any file writes at all. That is worse than the problem the hook exists to solve.

Every hook should fail open: if the script cannot parse the payload, if a dependency is missing, if the logic throws an exception, exit 0 and allow the action. The only path to a deny should be a positive match on a banned pattern. This means:

```bash
#!/bin/bash
PYBIN="$(command -v python3)"
[ -z "$PYBIN" ] && exit 0   # no python -> allow
exec "$PYBIN" "$DIR/guard.py"
```

The bash wrapper checks for Python. If Python is missing, it exits cleanly. The Python script itself wraps all logic in try/except and calls `allow()` on any error. Denial requires an explicit, intentional match.

## Write guard: blocking bad characters mechanically

A write guard hook fires on every Write or Edit tool call. It inspects the content being written and blocks patterns that violate the system's rules. In practice, this means:

```python
def allow():
    sys.exit(0)

def deny(reason):
    json.dump({
        "decision": "block",
        "reason": reason
    }, sys.stdout)
    sys.exit(0)
```

The guard scans only additive content: `Write.content` or `Edit.new_string`, never `Edit.old_string`. Removing a banned character via an edit should never be blocked.

It also scopes to document surfaces only: `.md`, `.html`, `.txt` files. Code files legitimately use characters that would be errors in prose (double hyphens in CLI flags, em dashes in string literals, CJK characters in i18n modules).

For prose content, the guard strips code blocks and inline code spans before scanning, so documentation can quote banned characters without tripping the guard:

```python
out, in_fence = [], False
for line in text.split("\n"):
    if line.lstrip().startswith("```"):
        in_fence = not in_fence
        out.append("")
        continue
    out.append("" if in_fence else line)
stripped = re.sub(r"`[^`]*`", "", "\n".join(out))
```

Then it scans `stripped` for the actual violations.

## The reliability canary: detecting drift without testing for it

A different kind of hook solves a different problem: how do you know the assistant is still following instructions after context compression?

A reliability canary works like this: the assistant is instructed to emit a small marker at the start of every response, incrementing a counter. A Stop hook grades each response, checking that the marker is present and correctly incremented.

The canary has three design constraints:

1. **The hook reminds emission but never injects the number.** A UserPromptSubmit hook adds a one-line reminder: "begin your reply with the marker." It does not say what the next number should be. If the hook spoon-fed the digit, the canary would degrade into a copy operation that even a severely degraded session could pass.

2. **The watchdog grades but never blocks.** The Stop hook checks for the marker, logs a miss to a degradation log file, and fires a notification. It never prevents the assistant from responding. A missing canary is a signal, not a failure.

3. **It detects silent degradation.** When context windows compress or attention drifts, the canary is one of the first things to drop. A stalled or missing canary on turn 47 tells you the session is degrading before you notice it in the quality of the work.

The canary is deliberately fragile. That fragility is the feature: it breaks early, so you can hand off to a fresh session before the important work suffers.

## Layered defense

No single hook catches everything. The production system uses three layers:

| Layer | What it catches | Failure mode |
|---|---|---|
| Write guard (PreToolUse) | Bad characters before they hit disk | Blocks the write, assistant retries |
| Canary watchdog (Stop) | Behavioral drift, attention loss | Logs to degradation file, notifies |
| Prompt reminder (UserPromptSubmit) | Canary omission from forgetfulness | Injects reminder, no enforcement |

The write guard is deterministic: it will catch every em dash in every markdown file, every time. The canary is probabilistic: it catches drift that deterministic rules cannot express. The prompt reminder reduces false alarms by nudging before the watchdog has to fire.

Together, they create a system where mechanical rules are enforced mechanically, behavioral expectations are monitored probabilistically, and the human only intervenes when something genuinely novel happens.

## Practical patterns

**Scope hooks narrowly.** A write guard that scans every file write will slow the system and produce false positives in code files. Scope by file extension, by directory, or by content type.

**Log denials visibly.** When a hook blocks an action, the assistant needs to know why. The deny payload should include a reason string that surfaces in the conversation: "Blocked: em dash detected in markdown file." Without this, the assistant retries the same action and the user sees unexplained failures.

**Keep hooks fast.** Hooks run synchronously in the tool-use pipeline. A hook that takes 500ms to execute adds 500ms to every tool call. Shell scripts that exec into Python are fine. Shell scripts that start Docker containers are not.

**Version your hooks.** Hook logic evolves as the system matures. A write guard that blocks em dashes today might also block smart quotes tomorrow. Keep hooks in version control alongside the system's configuration.

The core insight is that hooks transform aspirational rules into enforceable ones. Instead of hoping the assistant follows a style guide, you gate every write against the style guide's rules. The assistant learns from denials (it gets the feedback in-context and adjusts), and the system's quality invariants hold even as context compresses and sessions degrade.
