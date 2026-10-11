---
layout: post
permalink: /2026/10/agent-fleet-two-lanes.html
title:  "Building and Managing an AI Agent Fleet: Orchestrating Autonomous Intelligence at Scale"
author: oscar
categories: [ ai, agents, automation, engineering ]
tags: [ai, agents, automation, tmux, ctrlrelay, devops, llm]
image: images/posts/ai-practical-use.png
description: "Part one of five on running coding agents across several machines: the interactive lane, the headless lane, the one file they share, and the failure mode each one has."
featured: false
comments: false
---

A friend asked how I use AI day to day. I started replying in chat, but four paragraphs in, I realized I needed more room, and it's a very good excercise as backup in case I forget how everything is wired. The answer involves the machines I use, how I assign work, and what happens when an agent gets stuck. So I'm writing a series about the setup I actually run.

I don't run Hermes, Grok bot, or heavy multi-agent frameworks. Working in security taught me to pay attention to what I centralize, because it becomes a target. Complex infrastructure also takes time to maintain. My setup is simpler and cheaper: regular agents on Linux and Mac machines, with prompts that give each agent a role, and a single entry point for my interactions that act as program manager and mission control.

Each machine keeps its own keys, local clones, and agent sessions. The agents run on those machines. One coordination agent assigns the work, but it holds no secrets. If it goes down, work stops. If it gets compromised, it still doesn't have my GitHub tokens.

Here’s my hardware setup right now:

- Four machines:
  - Linux laptop 4 GB RAM (SSH access to other nodes, fleet coordination, and 3 of the agent sessions)
  - MacBook Pro M4 Pro 24 GB RAM (coding, UI/CX testing, iOS/Apple development, model fingerprinting, Software Composition Analysis)
  - MacBook Pro M1 Max 64 GB RAM (the fleet's Ollama host: serves local models to every other machine over the LAN, and runs nothing else)
  - Linux PC 32 GB RAM / RTX 5060 8 GB VRAM (local models, GitHub Actions, security analysis, software testing)

 The coding and review loop:

I use two LLM coding providers: Claude Code (Anthropic) with a Max subscription, and Codex (OpenAI) with a PRO subscription. Claude handles coding and planning, while Codex handles adversarial testing: looking for ways the changes might fail.

I mostly code with Sonnet and Opus. When a problem gets more complex, or the review loop goes past five iterations, the fleet coordinator escalate to Fable or Astra. For testing, I use Sol or Terra, depending on the work.

The review loop covers selected files, formats, and artifacts. This is part of each iteration, and I'll describe the full loop in a future post.

The work runs in two lanes:

- The interactive lane: 21 long-running agent sessions spread across three of the four machines, working as a team. A coordination agent called "TPM" handles:
  - Each agent's role, skills, and plugins, and communication between machines.
  - Providing context, managing memory, and measuring and clearing the context window.
  - Assigning work based on what each machine can handle.
- The headless lane: ctrlrelay runs on the Linux PC and watches 85 repositories across three organizations. It handles GitHub Dependabot alerts, security fixes, blocked upgrades, research, feature design, triage, releases, and assigned GitHub issues, following the configured policy and rules. When it needs a decision from me, it asks through Telegram or Mattermost.

Both lanes use the same Git repository for configuration, memory, and fleet data, but they run as separate systems. Keeping the rules together avoids maintaining a different copy on every machine, and changes go through pull requests. Machines running the sync daemon check for updates every 15 minutes; on the others, I run an update script by hand. A failed pull leaves the old rules in place. I've seen a node log 418 failed pulls, including five days without a configuration update.

## The interactive lane

On each machine I cloned the repos I work on, grouped by organization into their own folders. At boot, each machine starts a tmux session per folder: one for the org-level folder, plus separate ones for the repos I edit most. A bash script does the setup, started by systemd on Linux and launchd on macOS:

```bash
# fleet-up: start this machine's Claude sessions inside tmux, each with Remote Control.
#
# fleet-up start any missing sessions, then report ones waiting for input
# fleet-up status show every configured session and its state
#
# Config: ~/.config/fleet/sessions, one per line: <name> <path>
```

I have some rules based on how each of the agents operate, for example the fleet keeps one session per folder, because `--continue` picks up the latest conversation in whatever directory you are in. Two sessions in the same directory will fight over the same conversation history and the same files. I do sometimes have two agents working on the same project, but never on the same machine. In some cases the agent could spin additional agents working on separate worktrees, but once the work is done, the agent clean up and consolidate the memory, to avoid future conflicts.

Sessions are not spread evenly. The coordinator is an old Linux laptop with 4 GB of RAM, and each session uses a few hundred megabytes, so it runs only the three org-level sessions, with no per-project ones. It also has a hard rule against running test suites, which I'll cover in part three of this series. That machine is my interface to the fleet and keeps the other nodes talking to each other.

Updating the files on disk doesn't refresh the rules a running session loaded at startup. Long-running sessions can therefore keep using old instructions until they restart. Part four covers that failure and why I now ask agents to read the file on disk before reporting a configuration error.

Most of my tuning time has gone into making sure the fleet keeps working in coordination. Having me in the loop doesn't solve the oversight problem: the fleet still has to know when to ask for help, and I have to know when something is waiting on me. From outside a tmux pane, an agent thinking for ninety seconds looks the same as one waiting for an answer or stuck on a prompt.

Early on, I found two sessions stuck on the same dialog, twelve hours apart. Both had finished their work, but both were still holding background shells they had started, read from, and never closed. When the exit command came, the tool stopped on "Background work is running" and waited for a keypress from a user who never arrived. I caught the first one by chance. I found the second only after I taught the health check to read the pane's footer; by then, it had spent hours looking busy while doing nothing at all.

The fleet has a heartbeat, or tick, mechanism that checks each agent's state and the changes being made to the code. The health check now recognizes that stuck state and tells the coordinator about it, but leaves the dialog alone: the session might be waiting on a real question, and clearing it could lose that question before I answer. The fleet can clear a session with nothing outstanding; if a dialog is open, it asks me to look.

Getting those checks and handoffs right has taken more of my time than making the fleet faster.

What surprised me is where the problems actually live. The agents do the work fine. They write the code, run the tests, and open the pull requests. What I had to engineer was the handoff: knowing when one is done, when it needs me, and when it has quietly finished and forgotten to say so. That is the part nobody warns you about, and it is the part worth building.

## The headless lane

The headless lane handles work without someone sitting at a terminal. It still asks me when a decision needs human input. I'll cover the setup in part two; here, I want to explain how it works.

The headless lane runs on [ctrlrelay](https://github.com/AInvirion/ctrlrelay), an open source project that runs as a background daemon. It polls GitHub for issues assigned to it or tagged with specific labels. When it picks up an issue, it adds a comment to let others know it has taken the task, creates a worktree, and runs an agent on it. Once the work is done and the verification and testing loop is complete, it opens a pull request.

Each task reports whether it finished, failed, or needs my input. That lets me keep track of work across repositories without opening a terminal for every agent.

My part in this lane is to set the rules, answer questions from the coding agent through Telegram or Mattermost, and review the resulting pull requests. The agents handle the coding and testing between those points. I still need to make time for the questions they send: a task that needs my decision stays blocked until I answer.

In this lane, ctrlrelay handles the scheduled sync, copying the shared rules, skills, and configuration into `~/.claude/`.

## What the rest of the series covers

Part two follows the headless pipeline from an incoming task to a pull request. It covers the checkpoint protocol, the CI wait commands and their exit codes, and two fetch bugs in the sync layer.

Part three covers how I spot stuck terminal sessions, assign work based on what each machine can handle, and keep the agents' context windows from growing too large.

Part four covers the policy repo: the skill trigger table, when agents ask for help, the format they use to report their work, and what happened when the number of assigned issues suddenly grew.

Part five is about checking the numbers I get from the fleet. It covers misleading measurements, test suites running against the wrong source tree, build environment mismatches, and why Git status alone can't tell me whether a pull request was merged.

In both lanes, an agent can stop working while everything around it still looks normal. Most of the tooling I've built is there to catch that moment and tell me something needs attention.

The views expressed in this document are solely my own and do not represent those of my current or past employers. If you identify an error, please contact me, and I will make the necessary updates.
