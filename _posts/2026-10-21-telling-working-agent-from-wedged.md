---
layout: post
permalink: /2026/10/telling-working-agent-from-wedged.html
title:  "Telling a Working Agent From a Wedged One"
author: oscar
categories: [ ai, agents, automation, engineering ]
tags: [ai, agents, tmux, monitoring, observability, automation, llm]
image: images/posts/small_feet.png
description: "A thinking agent, an agent waiting for an answer, and an agent frozen on a dialog all render the same way. What it takes to tell them apart, and what it costs to get the detector wrong."
featured: false
comments: false
---

Part 3 of five on running an AI agent fleet. [Part 1](/2026/10/agent-fleet-two-lanes.html) covered the two lanes, [part 2](/2026/10/github-issue-to-pr-headless.html) the headless one.

This is the lane with sessions I talk to, and it has the harder supervision problem of the two. A session thinking for ninety seconds, a session that asked me something an hour ago, and a session frozen on a dialog it cannot dismiss all produce the same thing on screen: a window with text in it and no cursor movement.

Everything below is about narrowing that gap, and about the three times I narrowed it wrong.

## The window existing proves nothing

The launcher starts one session per folder inside tmux and is called by systemd on the Linux box and launchd on the laptops. Its first job is deciding whether a session is even alive, which is less obvious than it sounds:

```bash
# Whether claude is still running in the window. When it exits the window stays open
# on a shell (and its old screen), so the window existing proves nothing.
claude_alive_in_window() {
  local id tty
  id="$(window_id "$1")"; [ -n "$id" ] || return 1
  tty="$(tmux display -p -t "$id" '#{pane_tty}')"
  ps -o comm= -t "${tty#/dev/}" 2>/dev/null | grep -qE '(^|/)claude$'
}
```

When the agent exits, tmux keeps the window open on a shell, and the shell inherits the screen the agent left behind. So the window is still there, still showing a conversation, still looking exactly like work in progress. The check has to go to the process table by the pane's tty and ask whether anything named `claude` is attached to it.

The adjacent function uses `ps` rather than `pgrep` for a reason that cost someone an afternoon:

```bash
# ps, not pgrep: pgrep misses processes in some macOS contexts.
```

Part five is entirely about that category: a tool that answers confidently about itself rather than about the world. `pgrep` returning nothing reads identically to no process existing.

## The detector that flagged a conversation

Detecting "waiting for input" is where I got it wrong in a way worth publishing. The honest version, from my own notes:

> **Conversation text can look like a prompt.** An early version matched "do you want to" and flagged a session whose reply said "What do you want to look at?". The check now matches only the dialog footer `Enter to confirm · Esc to cancel` in the last lines of the screen.

An agent whose own output discusses prompts and dialogs will trip any detector built on the vocabulary of prompts and dialogs. The fix was to stop matching meaning and match chrome instead. Dialogs render a fixed footer; conversation never does:

```bash
# Startup dialogs (folder trust, MCP approval, login) end with this footer or show a login menu.
# Match only those, never conversation text, which can say anything.
waiting_for_input() {
  local id; id="$(window_id "$1")"; [ -n "$id" ] || return 1
  tmux capture-pane -p -t "$id" 2>/dev/null | tail -12 |
    grep -qE 'Enter to confirm · Esc to cancel|Select login method|Press Enter to continue'
}
```

Two constraints in five lines. `tail -12` because a dialog footer is always at the bottom, and scanning the whole scrollback reintroduces the bug. The alternation lists specific strings, because "never conversation text, which can say anything" is the actual security property. An agent can emit any string you can think of, including the one you are grepping for.

## Twelve hours on a dialog nobody would press

On 20 September we found two sessions wedged on the same dialog, twelve hours apart. Both had finished their work. Both were holding background shells they had started, read the output of, and never closed. When `/exit` was typed, the CLI stopped on "Background work is running / The following will stop when you exit" and waited for a keypress.

From outside, a pane holding a modal is neither busy nor idle. One was noticed because a person happened to look at it. The other was found only once the health check was taught to see that specific footer.

The shell that held one of them open for twelve hours was this:

```bash
until [ "$(gh run list --branch <b> --limit 1 --json status -q '.[0].status')" = "completed" ]
do sleep 25; done
```

It ran from a scratchpad directory, which is not a git repository, so `gh` could not work out which repo to query. It failed with "failed to determine base repo", printed nothing to stdout, and the comparison became `"" = "completed"`, which is false forever. That loop could never terminate. From the outside it was indistinguishable from one patiently waiting on a slow CI run.

Three things came out of that, and only the third addresses the cause. Close a background shell in the same step that reads its output, because reading output does not reap a shell. Give every poll loop a deadline or an iteration cap. And run `gh` from inside the repository or pass `--repo`, because a command that can fail inside a loop condition is a case you have to handle. An empty string never equals the thing you are waiting for.

The session that wrote that loop had seen the "failed to determine base repo" error earlier in the same session and recorded it as harmless. The error was not hidden and it was not silent. It was read, and dismissed.

## Capability is a hard constraint, not a preference

Routing work by who is free produces a specific kind of outage. One machine in the fleet is a 2011 ultrabook with 3.7 GB of soldered RAM, and it coordinates rather than builds. The rule in my policy file is blunt about why:

> **Do not run test suites, builds, containers or snapshot matrices there.** Measured 25 Sep: the full suite was OOM-killed twice, the second time with nothing else on the box.

The second clause is the one that settles it. The first OOM kill is arguable, since something else might have been running. With nothing else on the box, 3.7 GB is not enough, and no amount of scheduling cleverness changes that.

The same constraint shapes how many sessions each machine gets. That box runs three, one per organisation folder, and its config carries the arithmetic in a comment: roughly 290 MB per session, so org level only. The laptop with 24 GB runs ten, including one per active repository. Heavy work is dispatched by capability, and a machine being idle is not a reason to send it a build.

## Context is a bill you pay on every call

Here is the measurement that changed how I run sessions. Across the fleet on 20 September, six sessions above 4% of usage were each carrying an average of **400,651 tokens of context**, and a seventh was on 808,481. Every one of those tokens is read again on every single call that session makes.

Resetting those sessions to roughly 150k would have cut about **26% of the fleet's usage**, at no cost in capability whatsoever. A session that finishes a task and keeps going drags the entire finished task through every later turn, for as long as it stays open.

So the rule is to clear between unrelated tasks, compact mid-thread when the thread still matters, and never do either mid-turn, because a clear during a task destroys the task. Anything that exists only in the conversation goes to a file first: a decision, a constraint somebody gave you, a measurement you made. Whatever is still only in the transcript when it clears is gone, and afterwards there is nothing to indicate it ever existed.

There is a tool that does this from outside, and the interesting part is what it refuses. It will not clear a session that is mid-turn, holding a dialog, has text typed in its input box, or whose repository has an open pull request. Each of those is a state where clearing destroys something. And it has one blind spot it cannot close: a session that asked a question and is waiting for an answer looks exactly like a session that has finished. Which is why the rule for the session is to say so in its last line rather than falling silent.

## The claim that was only an intention

The last failure in this lane is social rather than technical, and it is the most expensive one.

On 27 September four sessions were working and the forge showed four silent, unassigned, unlabelled issues. Each session had announced itself in a chat message, which is invisible to anyone looking at GitHub and gone as soon as the transcript clears. An unpushed branch is invisible to the forge, and so is a worktree. That is how two sessions end up editing one file.

The variant that does more damage is an issue where the label was taken and a comment was posted, and the last sentence of the comment is an intention rather than a result. Nothing was pushed. That issue is now worse than an unclaimed one, because the claim stops anything else from picking it up while producing no work.

A pushed branch is the cheapest honest signal available. It costs one command, it is visible to every human and every other session, and it survives a restart. The report format every session uses is built around the same idea, and part four has it in full, along with the rule that a test count has to name the command and the numbers rather than saying "the suite passed".

Supervising this lane turned out to have almost nothing to do with the quality of the agents. The whole problem is that a stopped process and a working one produce the same evidence, and every fix above is a way of forcing them to produce different evidence.

Part four is the policy that all of these sessions read, and what it cost when that policy was one long file.

---

*The views expressed in this document are solely my own and do not represent those of my current or past employers. If you identify an error, please contact me, and I will make the necessary updates.*
