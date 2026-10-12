---
layout: post
permalink: /2026/10/telling-working-agent-from-wedged.html
title:  "My AI Agent Fleet, Part 3: Working, Waiting, or Stuck"
author: oscar
categories: [ ai, agents, automation, engineering ]
tags: [ai, agents, tmux, monitoring, observability, automation, llm]
image: images/posts/agent-fleet-part3.png
description: "The interactive half of an agent fleet: how I tell a session that is thinking from one that is waiting on me or stuck, and the rules that keep long-running sessions healthy."
featured: false
comments: false
---

Part 3 of five on running an AI agent fleet. [Part 1](/2026/10/agent-fleet-two-lanes.html) covered the two lanes and the rulebook they share, and [part 2](/2026/10/github-issue-to-pr-headless.html) reviewed in details the headless lane.

This is the lane with sessions I talk to directly (thorught a central coordinator). A session thinking for ninety seconds, a session that asked me something an hour ago, and a session stuck on a dialog it cannot dismiss all look the same on screen: a window with text in it and nothing moving.

This post is about how I tell them apart.

## How it is put together

The interactive lane has four parts:

- **The launcher** is the script from part one. At boot it starts one Claude Code session per folder inside tmux, 21 in all across three machines, and restarts any that have exited.
- **The health check** looks at every session and sorts it: working, waiting for input, stuck on a dialog, or exited.
- **The coordinator** is the TPM session from part one. It hands out work, reads what the health check reports, and tells me when something needs a person.
- **The context tool** watches how much each session is carrying and clears the ones that have finished their work.

Every session also has a Remote Control name, so I can open any of them from my phone and answer it there.

## The window existing proves nothing

When an agent exits, tmux keeps its window open and drops it to a shell, and the shell still shows the agent's last screen. So the window is still there, still showing a conversation, and still looking exactly like work in progress.

So the launcher doesn't trust the window. It asks the operating system which program is actually running in that window, and only counts the session as alive if that program is Claude Code. Even the way it asks matters: on macOS, one common way of listing processes sometimes misses them, and an empty answer looks exactly like "nothing is running". Part five comes back to that idea: a check should look at the thing itself, not at something that only resembles it.

## Match the screen, not the words

Deciding that a session is waiting for me was harder than it looked. An early version looked for phrases like "do you want to", and flagged a session whose reply to me said "What do you want to look at?" An agent that talks about prompts and questions will trip any check built on the words of prompts and questions.

The fix was to stop reading what the screen says and look at how it is built. Real dialogs always end with the same fixed line at the bottom, like "Enter to confirm · Esc to cancel". The check looks only for a short list of those exact lines, and only in the last few lines of the screen. Part one shows the code.

That matters for security too. An agent can print any text at all, including the phrase a check is looking for, so the check stays narrow: a few exact lines, in the one place a dialog puts them.

## A session that finished but cannot leave

A session can finish its work and still be stuck. For example, an agent starts a small background job to keep an eye on a CI run, reads the result, and never stops the job. When it is later told to exit, Claude Code shows "Background work is running" and waits for a keypress that nobody will send. From outside, that window looks busy. It is not working, and it is not waiting on me. It is stuck.

The background job can make it worse. One that kept a session stuck for hours asked GitHub every 25 seconds whether a CI run had finished, and was meant to stop when the answer was "completed". But it ran in a folder that was not a git repository, so the GitHub command failed every time and returned nothing. Nothing is never "completed", so the job could never end, and from outside it looked exactly like one patiently waiting on slow CI. The agent had even seen the GitHub error earlier and decided it was harmless. Nothing hid it; it was read and dismissed.

The health check now spots that exit dialog and tells the coordinator. It does not press the key itself, because a session might be waiting on a real question. The rules that came out of it are simple: stop a background job as soon as you have read its result, give every waiting loop a time limit, and run GitHub commands from inside the repository so they cannot fail quietly.

## Keeping sessions healthy

Two rules keep sessions from failing in slower ways.

The first is about machines. Work goes to a machine because it can handle it, not because it is free. The coordinator is an old laptop with 4 GB of memory, and the rules forbid running test suites or builds there: a full test suite ran out of memory on it twice, the second time with nothing else running. Memory also decides how many sessions a machine gets. At roughly 290 MB each, the coordinator runs three, and the laptop with 24 GB runs ten.

The second is about context: everything a session carries from earlier in its conversation. All of it is read again on every call. When I measured, six of the busiest sessions were each carrying about 400,000 tokens, and one was over 800,000. Trimming them to around 150,000 would have cut roughly a quarter of the fleet's usage, without losing anything they needed.

So sessions clear between unrelated tasks, compact when the thread still matters, and never do either in the middle of a task. Anything that exists only in the conversation, like a decision, a constraint, or a measurement, gets written to a file first, because once it is cleared it is gone. The context tool does this from outside, and it refuses to clear a session that is mid-task, holding a dialog, has text typed in, or has an open pull request. It has one blind spot: a session waiting on my answer looks just like one that is done. So a session that is waiting on me says so in its last line.

## A claim is a pushed branch

When a session picks up an issue, everyone else needs to know, me and the other sessions alike. A chat message saying "I'm on it" is invisible on GitHub and gone when the conversation clears. A branch that only exists on one machine is invisible too. That is how two sessions end up editing the same file.

So a claim goes on the issue itself, as a comment and a label. And a claim alone is not enough. An issue marked as taken with nothing pushed is worse than an unclaimed one, because it stops anyone else from picking it up. A pushed branch is the honest signal: one command, visible to everyone, and it survives a restart. Part four covers the report format every session uses, which is built around the same idea.

## My part

Supervising this lane has little to do with how good the agents are. A stopped session and a working one look the same, and everything above is a way of making them look different.

My part is to answer when the coordinator says a session needs me, often from my phone through Remote Control, and to review what the sessions produce. The health check, the context tool, and the claim rules exist so that I only get pulled in when a person is actually needed.

Part four of this series covers the rulebook all these sessions read, and how it is organized.

---

*The views expressed in this document are solely my own and do not represent those of my current or past employers. If you identify an error, please contact me, and I will make the necessary updates.*
