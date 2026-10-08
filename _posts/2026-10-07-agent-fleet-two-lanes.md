---
layout: post
permalink: /2026/10/agent-fleet-two-lanes.html
title:  "An Agent Fleet Has Two Lanes, and They Fail Differently"
author: oscar
categories: [ ai, agents, automation, engineering ]
tags: [ai, agents, automation, tmux, ctrlrelay, devops, llm]
image: images/posts/ai-practical-use.png
description: "Part one of five on running coding agents across several machines: the interactive lane, the headless lane, the one file they share, and the failure mode each one has."
featured: true
comments: false
---

Part 1 of five on running an AI agent fleet.

A friend asked me how I actually use AI day to day. I started answering in a chat window and gave up four paragraphs in, because the honest answer has configuration files in it, and a chat window turns configuration files into advice. So here are five posts instead.

What runs right now: 21 long-lived agent sessions configured across three machines, and a daemon on one of those machines watching 87 repositories for work that nobody is sitting in front of. Those are two separate systems. They share one git repository, and they break in opposite directions.

## The interactive lane

Each machine starts a set of tmux sessions at boot, one per folder. An org-level folder gets a session, and the repositories I work in most get their own. The launcher is a bash script that systemd calls on the Linux box and launchd calls on the laptops:

```bash
# fleet-up: start this machine's Claude sessions inside tmux, each with Remote Control.
#
#   fleet-up            start any missing sessions, then report ones waiting for input
#   fleet-up status     show every configured session and its state
#
# Config: ~/.config/fleet/sessions, one per line:   <name>  <path>
```

One session per folder matters because `--continue` resumes the most recent conversation in that directory. Two sessions in one folder means two agents racing for the same conversation history and the same working tree.

Sessions are not distributed evenly. The coordinator box has 3.7 GB of RAM, and its sessions file carries the reason in a comment: roughly 290 MB per session, so org level only, no per-project sessions. That machine also has a hard rule against running test suites, which I will come back to in part three.

A human is in the loop in this lane, which sounds like it makes supervision easy. It does the opposite. The failure mode here is a session that goes quiet, and from outside a tmux pane there is no difference between an agent thinking for ninety seconds, an agent waiting for an answer to a question it asked, and an agent frozen on a modal dialog. All three render as a static screen. The launcher's own comments say so:

```bash
# Whether claude is still running in the window. When it exits the window stays open
# on a shell (and its old screen), so the window existing proves nothing.
```

On 20 September we found two sessions wedged on the same dialog, twelve hours apart. Both had finished their work. Both were holding background shells they had started, read the output of, and never closed, so when `/exit` was typed Claude Code stopped on "Background work is running" and waited for a keypress that nobody was going to press. One was noticed because somebody happened to look at the pane. The other was found only after the health check was taught to read the dialog footer. Twelve hours of a machine looking busy while holding nothing.

Every interactive-lane problem I have had has that shape. The work was finished and the output was fine. The pane stopped, and a stopped pane looks identical to a working one.

## The headless lane

The other lane has no human in it at all. A daemon polls GitHub, and when an issue is assigned or carries the right label it takes a per-repository lock in SQLite, cuts a worktree from a bare mirror, runs an agent against the issue, and opens a pull request. The agent reports by writing a small JSON checkpoint with one of three statuses, `DONE`, `BLOCKED_NEEDS_INPUT`, or `FAILED`. The file is written to a temporary path and renamed into place, because the orchestrator has to be able to restart and still read where everything got to.

That daemon is [ctrlrelay](https://github.com/AInvirion/ctrlrelay), Apache 2.0, currently 0.12.2. Part two is entirely about it.

The headless lane fails by looking busy, which is worse. The clearest example is in the changelog for version 0.5.0, dated 11 May 2026. A security sweep ran across 86 repositories. Twenty-one of those sessions hit `BLOCKED_NEEDS_INPUT`, meaning the agent stopped, decided it needed a human, and said so in its checkpoint. The number of messages that reached me was zero. Twenty-one agents asked for help into a bridge that was not wired up on that path, and the sweep reported as a sweep that ran.

Getting the messages flowing was the easy half. The hard half showed up four months later, in 0.9.0, dated 7 September 2026, when I measured what happened to the questions after they arrived. On one deployment: 46 questions posted, 4 ever answered, 16 still sitting live, and five repositories re-asking the same question on consecutive days. Rows that recorded a pending question were deliberately built to outlive the session that asked, so a late reply could still restart the work. Nothing ever retired one, so the table only grew.

The code fix was a time-to-live on unanswered questions, defaulting to 48 hours, so an expired question stops being offered as the routing target for a stray reply. That part was small. The finding was that I had built a careful escalation path to a person who was not reading it, and the system had no way to tell me, because from the daemon's side a question asked and a question answered look the same until somebody counts the rows.

## The one file both lanes share

A single git repository holds the rules file, the skills, and the settings that every session on every machine reads. It is applied to `~/.claude/` by the same daemon that runs the headless lane:

```yaml
schedules:
  personalization_cron: "*/15 * * * *"

personalization:
  repo: "<owner>/<policy-repo>"
  paths:
    - source: "global/CLAUDE.md"
      target: "~/.claude/CLAUDE.md"
    - source: "global/skills/gh-dashboard/"
      target: "~/.claude/skills/gh-dashboard/"
```

Edit in the repository, push, and every machine where the daemons are loaded picks it up within fifteen minutes. On machines where they are not loaded, a restore script does it by hand, and knowing which machine is in which state is part of the job rather than a detail.

The reason policy is centralized is that the alternative is three machines with three slightly different rulebooks and no way to tell which one an agent was following when it did something surprising. Policy changes go in by pull request, same as code.

The reason it is a trigger table of skills rather than one long document is cost, and that is a measurement rather than a preference. The always-resident rules file reached about 2,600 lines, read on every call by every session, and most of it did not apply to most calls. Part four has the numbers and the redesign.

Centralizing the rules introduced its own failure, which is worth saying now because it is unintuitive: a running session holds what it read at startup. A fix pushed to that repository reaches the disk on every machine in fifteen minutes and reaches a live session only when the session restarts. On 19 and 20 September this misled us twice in two days, once with a credential that had already been renewed still being reported as expired, and once with a skill reported as stale an hour after the fix was pushed and pulled. From inside a session there is no way to tell a stale copy from a current one, so the rule now is to read the file on disk and quote what is actually there before reporting it as out of date.

## What the rest of the series covers

Part two is the headless pipeline end to end: the checkpoint protocol, why a CI wait had to become a first-class command with real exit codes, and two fetch bugs that sat on opposite sides of the same line of code, one noisy and one silent.

Part three is the interactive lane: detecting a wedged session without false positives, routing work by machine capability, and the measurement that showed six sessions carrying an average of 400,651 tokens of context each.

Part four is the policy repository: the trigger table, the escalation model, the report format that every session uses, and what happened the day issue count went from 20 to 38.

Part five is about trusting numbers. Four separate cases in one day where a measurement was confidently wrong, a test suite that was testing the wrong source tree, and why "is my work merged" has no correct answer in git alone.

Both lanes fail by being silent, and silence is the one state that reads as healthy from outside. Nearly everything I have built around these agents exists to make a stopped one say so.

---

*The views expressed in this document are solely my own and do not represent those of my current or past employers. If you identify an error, please contact me, and I will make the necessary updates.*
