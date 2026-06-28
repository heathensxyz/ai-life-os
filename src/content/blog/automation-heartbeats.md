---
title: 'Self-Healing Automation with Heartbeat Monitoring'
description: 'How to build automations that detect their own failures and fix themselves, using heartbeat files and an orchestration layer.'
pubDate: 'Jun 28 2026'
---

Every automation fails eventually. Scripts crash, APIs change, authentication tokens expire, mounts disconnect. The question is not whether your automations will break but how long they will stay broken before you notice.

Most personal automation systems fail silently. A cron job that syncs files to a NAS stops running after a system update changes a path. A scheduled data pull returns empty results because an API key expired. A task sync daemon hits a rate limit and stops processing. Nothing alerts you. You discover the failure days or weeks later when you need the thing that was supposed to be syncing.

Heartbeat monitoring fixes this by inverting the problem. Instead of checking whether each automation ran, you check whether each automation reported that it ran.

## The heartbeat pattern

A heartbeat is a file that an automation touches every time it completes successfully. The file's modification timestamp is the proof of life. A monitoring layer reads the timestamps and flags any heartbeat that is stale, meaning the automation has not reported success within its expected interval.

```
heartbeats/
  task-sync.heartbeat        # touched every 30 minutes
  vault-backup.heartbeat     # touched daily
  data-pull.heartbeat        # touched daily
  content-pipeline.heartbeat # touched weekly
```

Each heartbeat file is just a timestamp. The automation's last line, after all work is done, is:

```bash
touch "$HEARTBEAT_DIR/task-sync.heartbeat"
```

If the script crashes before completing, the heartbeat does not update. If the script runs but produces bad output, you can add a validation check before the touch. The heartbeat only fires on verified success.

## The monitoring layer

A monitoring check runs on a schedule (every few hours, or at the start of each work session) and compares each heartbeat's age against its expected interval:

```bash
MAX_AGE_HOURS=26  # daily automation gets 26h grace
HEARTBEAT="heartbeats/data-pull.heartbeat"

if [ ! -f "$HEARTBEAT" ]; then
    echo "MISSING: $HEARTBEAT never created"
elif [ $(($(date +%s) - $(stat -f %m "$HEARTBEAT"))) -gt $((MAX_AGE_HOURS * 3600)) ]; then
    echo "STALE: $HEARTBEAT last updated $(stat -f %Sm "$HEARTBEAT")"
else
    echo "OK: $HEARTBEAT"
fi
```

The grace period matters. A daily automation that runs at 8am should not be flagged as stale at 8:01am the next day. A 26-hour window accounts for slight timing variations. A weekly automation gets an 8-day window.

## Session-start integration

The highest-value integration point is the start of a work session. When you begin working with your AI assistant, the session-start routine scans heartbeat files and surfaces any staleness:

```
Automation health: 6 OK, 1 WARN
WARN: content-pipeline (last: 9 days ago, expected: weekly)
```

This catches failures at the moment you are most likely to act on them: the beginning of a focused work block, when context is fresh and energy is available. Compare this to email alerts that arrive at 3am and get buried in the morning inbox.

## Self-healing: from detection to repair

Detection without repair is just a fancier way to discover problems. The next level is an orchestration layer that can attempt fixes automatically.

A self-healing loop works in tiers:

**Tier 1: restart.** If the automation failed because a process died or a service was unreachable, try running it again. Most transient failures (network timeouts, temporary API errors, mount reconnection) resolve on retry.

**Tier 2: known fix.** If the automation fails with a recognized error pattern, apply the known fix. Mount disconnected? Reconnect and retry. Auth token expired? Refresh and retry. Path changed after system update? Check the expected paths and update.

**Tier 3: escalate.** If the automation has failed multiple times and no known fix resolves it, create a task for the human. Not a notification (those get ignored) but a concrete task with the error details, the remediation attempts that failed, and a suggested investigation path.

```python
checks = {
    "task-sync": {
        "heartbeat": "task-sync.heartbeat",
        "max_age_hours": 2,
        "restart_cmd": "~/bin/task-sync.sh",
        "max_retries": 3
    }
}

for name, config in checks.items():
    age = heartbeat_age(config["heartbeat"])
    if age > config["max_age_hours"] * 3600:
        for attempt in range(config["max_retries"]):
            result = run(config["restart_cmd"])
            if result.success:
                break
        else:
            create_task(f"Fix {name}: failed after {config['max_retries']} retries")
```

## The status dashboard

Heartbeat data is simple enough to display on a dashboard. Each automation gets a row with its name, last heartbeat timestamp, expected interval, and current status (OK, WARN, FAIL). Color-coded for quick scanning.

A more sophisticated dashboard tracks trends: how often each automation fails, whether failure frequency is increasing, and which automations are the most fragile. This data informs where to invest engineering effort. An automation that fails weekly is worth hardening. One that has run reliably for six months needs no attention.

## Practical patterns for heartbeat systems

**Touch the heartbeat last.** The heartbeat should be the final action after all work is complete and verified. If you touch it before validation, a script that runs but produces garbage still reports success.

**Use separate heartbeats for separate concerns.** A sync script that both pulls and pushes should have two heartbeats if either direction can fail independently. One heartbeat for a compound operation masks partial failures.

**Give new automations short intervals at first.** When deploying a new automation, set the expected interval tight (a few hours) so failures surface quickly during the shakeout period. Loosen to the real interval once the automation proves stable.

**Log alongside heartbeats.** A heartbeat says "I ran successfully." A log says "here is what I did." When something goes wrong, you need both: the heartbeat to know when it stopped working, and the log to know what happened in the last successful run.

**Do not alert on every check.** A heartbeat that is 5 minutes past its interval is probably fine. One that is 3x past its interval needs attention. Set alert thresholds at meaningful multiples, not at the edge of the window.

The underlying principle is that reliable automation requires reliable monitoring, and the simplest reliable monitor is a file timestamp. No message queue, no monitoring SaaS, no infrastructure dependency. Just a file that gets touched on success and a check that reads the timestamp. When the check fails, you know something is broken. When you pair that with a self-healing layer, most failures resolve before you ever see them.
