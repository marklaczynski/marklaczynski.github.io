---
title: "Building My First Claude Code Plugin: Desktop Notifications When AI Needs You"
date: 2026-02-27 18:00:00 -0600
categories: [BUILD]
tags: [claude-code-plugins, developer-tools, hooks]
---

I've been using Claude Code as my daily driver for a while now. It's become the kind of tool where I forget what it was like before — it reads my codebase, writes code, runs commands, and handles the tedious parts of development so I can focus on the interesting stuff.

But there's one friction point that kept bugging me: Claude Code is async by nature. It'll ask me a question or need permission to run a command, and I'm already alt-tabbed into Slack or a browser. Minutes pass. Claude sits idle. I come back and realize it's been waiting for me the entire time.

What if Claude could just tap me on the shoulder?

## Discovering the Extension System

I started by asking Claude itself — right there in the CLI — what extension points exist. Turns out Claude Code has a surprisingly rich ecosystem:

- **Skills** are reusable workflows you invoke with `/slash-commands` that run in your conversation context
- **Agents** are isolated AI sessions that work independently and report back
- **Hooks** are deterministic shell scripts that fire on specific events — no LLM involved

Hooks were clearly the right tool for this job. They're just bash scripts triggered by events like `PreToolUse`, `PostToolUse`, `PermissionRequest`, and `UserPromptSubmit`. Fast, predictable, and simple to reason about.

The plan was straightforward: start a timer when Claude asks me something, cancel it if I respond in time, and send a macOS notification if I don't.

## The First Attempt

I started with the `Stop` hook, which fires every time Claude finishes a response. Two scripts: `notify-if-idle.sh` to start a background timer, and `cancel-idle-timer.sh` to kill it when I respond.

The notification itself was easy — macOS has `osascript` for native notifications:

```bash
osascript -e 'display notification "Claude Code is waiting" with title "Claude Code" sound name "Ping"'
```

Except the sound didn't play. I tried different sound names, checked System Settings, verified notifications were enabled for Script Editor. Nothing. Turns out `display notification ... sound name` is just broken in newer versions of macOS. The notification appears, but the sound parameter is silently ignored.

The fix was to bypass the notification sound entirely and play it directly:

```bash
afplay /System/Library/Sounds/Glass.aiff &
osascript -e 'display notification "Claude Code is waiting" with title "Claude Code"'
```

That worked. I had notifications with sound. But there was a bigger problem: the `Stop` hook fires on *every* response, not just questions. Claude finishes explaining something? Notification. Claude runs a command? Notification. It was like a needy coworker tapping your shoulder every thirty seconds.

## Narrowing the Trigger

I needed to be more specific about *when* to start the timer. The `Stop` event was too broad. What I actually cared about were two scenarios:

1. Claude asks me a question (`AskUserQuestion` tool)
2. Claude needs permission to use a tool (`PermissionRequest` event)

I switched the start timer to fire on `PreToolUse` with a matcher for `AskUserQuestion`, and on `PermissionRequest`. For cancellation, I used `PostToolUse` (when a tool completes) and `UserPromptSubmit` (when I type a message).

This was the right architecture. No more notifications on regular responses.

## The Race Condition

Here's where it got interesting.

The cancel mechanism used marker files. The start script would create a file like `/tmp/claude-idle-marker-12345` (using the process PID for uniqueness), sleep for the delay, then check if the file still existed. The cancel script would delete all marker files. Simple, right?

In testing, I'd answer a question quickly, wait... and still get a notification. The cancel wasn't working.

I added logging and found something odd:

```
15:45:47: START marker=/tmp/claude-idle-marker-72672
15:45:47: START marker=/tmp/claude-idle-marker-72673
15:45:47: CANCEL (found both markers, deleted)
15:45:57: Marker exists, NOTIFYING
15:45:57: Marker exists, NOTIFYING
```

The cancel hook ran at the same timestamp as the start hooks. It found the markers and deleted them. But ten seconds later, both background processes still found their markers. How?

The start hooks were configured with `async: true`, meaning they ran in the background. The cancel hook was synchronous. Even though the log timestamps showed the same second, the actual execution order was:

1. Cancel hook runs (synchronous, immediate)
2. Async start scripts finally create their marker files (slightly after cancel already cleaned up)

The markers were being created *after* the cancel deleted them.

My first fix was to make the start hook synchronous and fork the timer internally:

```bash
touch "$MARKER"
(
  sleep 60
  if [ -f "$MARKER" ]; then
    # notify
  fi
) &
exit 0
```

This guaranteed the marker existed before Claude proceeded. But it introduced a new problem: synchronous hooks **block Claude Code**. The question wouldn't even appear for ten seconds while the hook ran. The cure was worse than the disease.

### The Timestamp Solution

The real fix was to stop thinking about files as coordination mechanisms and use timestamps instead:

```bash
# Start hook (async)
START_TIME=$(date +%s)
sleep 60
if [ -f "$CANCEL_FILE" ]; then
  CANCEL_TIME=$(cat "$CANCEL_FILE")
  if [ "$CANCEL_TIME" -ge "$START_TIME" ]; then
    exit 0  # User responded after we started — don't notify
  fi
fi
# Send notification
```

```bash
# Cancel hook
date +%s > /tmp/claude-idle-cancel
```

The timer records when it started. The cancel hook writes the current time. When the timer wakes up, it compares: if the cancel timestamp is newer than the start time, the user responded. No more race condition — even if the cancel runs before the async script starts, the timestamp comparison handles it correctly.

## One More Bug

With the race condition solved, I ran into one more issue. I was getting notifications after granting permissions, even though I responded immediately.

The logs revealed the problem:

```
17:26:32: CANCEL time=1772234792
17:26:37: START event=PermissionRequest time=1772234797
17:26:47: STALE CANCEL (cancel=792 < start=797) → NOTIFYING
```

`PermissionRequest` fires on *every* tool use, even auto-allowed ones. When I click "Allow" on a permission dialog, `UserPromptSubmit` doesn't fire — that's only for typed messages. So the timer started but nothing cancelled it.

The fix was simple: remove the `AskUserQuestion` matcher from the `PostToolUse` cancel hook. Now *any* tool completing cancels the timer. When you grant permission, the tool runs, `PostToolUse` fires, timer cancelled.

## Packaging as a Plugin

With the hooks working reliably, I wanted to share them. Claude Code has a plugin system that bundles hooks, skills, agents, and MCP servers into installable packages.

The plugin structure is minimal:

```
claude-idle-notifier/
├── .claude-plugin/
│   └── plugin.json
├── hooks/
│   ├── hooks.json
│   └── scripts/
│       ├── notify-if-idle.sh
│       └── cancel-idle-timer.sh
└── README.md
```

I hit one more gotcha during packaging: the `hooks.json` file needs the hook definitions wrapped in a `"hooks"` key. My original file put the events at the top level and Claude Code silently ignored them. I only figured it out by comparing against a working plugin's `hooks.json`.

```json
// Wrong — hooks silently ignored
{ "PreToolUse": [...] }

// Correct
{ "hooks": { "PreToolUse": [...] } }
```

I also added Linux support (using `notify-send` for notifications and `paplay`/`aplay` for sound), and made the delay configurable via a `CLAUDE_IDLE_DELAY` environment variable that defaults to 60 seconds.

## Publishing

For distribution, I went with a separate marketplace approach — one index repo that points to individual plugin repos:

- [**marklaczynski/claude-plugins**](https://github.com/marklaczynski/claude-plugins) — the marketplace (just a `marketplace.json` pointing to plugin repos)
- [**marklaczynski/claude-idle-notifier**](https://github.com/marklaczynski/claude-idle-notifier) — the actual plugin

Installing is two commands:

```bash
/plugin marketplace add marklaczynski/claude-plugins
/plugin install idle-notifier@marklaczynski-plugins
```

## Reflections

The entire thing — from "can Claude notify me?" to a published, installable plugin — was built in a single session with Claude Code. There's something deeply meta about using an AI coding assistant to build a plugin for that same AI coding assistant, debugging the plugin's behavior by asking the assistant to inspect its own hook logs.

The debugging journey was the most instructive part. The race condition between async and synchronous hooks isn't something you'd find in documentation — it's the kind of thing you only discover by building something real and watching it break. The timestamp solution feels obvious in hindsight, but it took three failed approaches to get there.

If you use Claude Code regularly, give the plugin a try. And if you end up building your own plugin, embrace the weird bugs. They're where the learning happens.
