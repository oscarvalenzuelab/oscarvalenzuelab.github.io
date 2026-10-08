---
layout: post
published: false
draft: true
permalink: /2026/10/many-machines-one-queue.html
title:  "Many Machines, One Queue"
author: oscar
categories: [ ai, development, agentic, automation ]
tags: [ai, agentic, orchestration, ctrlrelay, telegram, mattermost, automation, workflow]
image: images/posts/many-machines-one-queue.png
description: "Running coding agents across several machines in parallel. The hard part is not making them work, it is knowing what state each one is in."
featured: false
comments: false
---

This is the last of three posts about how I work with AI agents. The [first]({{ site.baseurl }}/2026/10/stop-prompting-write-rules.html) was about writing rules instead of prompts. The [second]({{ site.baseurl }}/2026/10/never-trust-the-output.html) was about not trusting the output. Both work on one laptop and both are worth doing.

This one is about running several machines at once, and it is genuinely optional. I will say why at the end.

The thing nobody tells you up front: once more than one agent is working, the hard problem stops being the code and becomes **knowing what state each one is in**. Everything below is a consequence of that.

## The twelve-hour wedge

Two of my sessions were found stuck, twelve hours apart, on the same dialog.

Both had finished their work. Both were holding a background shell they had started, read the output of, and never closed. When the exit command was given, the tool stopped on *"Background work is running. The following will stop when you exit"* and waited for a keypress that never came.

From the outside, a pane holding a modal is neither busy nor idle. It looks exactly like a session working quietly on something hard. One of the two was noticed because a person happened to glance at it. The other was only found once the health check was taught to see it.

Three rules came out of that, and the third is the one that actually caused it.

**Close a background shell in the same step that reads its output.** A finished shell is not a closed one. Reading its output does not reap it. The cost of leaving it is zero and invisible until the moment somebody types exit, which is the worst possible moment to discover it.

**Before exiting, verify at the process level, not from memory.** And know the limit of what you can check: a session cannot read its own status bar. If you believe you stopped everything, say so and ask someone outside to confirm. That is the right shape for any report. Name what you verified, name what you cannot see, ask for the part you cannot see.

**Every wait loop gets a bound.** A deadline, an iteration cap, or both. I covered the two ways that goes wrong in [post 1]({{ site.baseurl }}/2026/10/stop-prompting-write-rules.html): one loop that could never end, one that ended instantly, both reporting confidently.

## What the fleet actually is

The scheduler is [ctrlrelay](https://github.com/AInvirion/ctrlrelay), which is Apache-2.0 and public. It is a small daemon that runs on a laptop. No server, no queue, no multi-tenant anything. My agent credentials, my GitHub credentials, my repos, my machine.

What it does:

```
              GitHub API
                  |
         (poll: issues assigned to me,
          or carrying an opt-in label)
                  |
            poller daemon  <----  scheduler (daily secops job)
             /         \
      new issue      blocked session
          |                 |
    dev pipeline      chat bridge
    (own worktree)    (Telegram or Mattermost)
          |                 |
      opens a PR      relays the question, resumes on your reply
          |
      PR watcher ---> tells you when it merges
```

An issue appears, the daemon spawns a session in its own git worktree, the session works, a pull request opens, and a watcher follows it to merge. If the session hits a question it cannot answer alone, the chat bridge asks me and resumes when I reply.

The machines are not interchangeable, and pretending they were cost me a day:

| Machine | Role | Why |
| --- | --- | --- |
| alkapro, alkaturbo | dev and QA | Suites, builds, containers, review rounds. The real work. |
| alkaspire | coordinator only | 3.7 GB of RAM. The full suite was killed by the OOM reaper twice, the second time with nothing else running on the box. |
| neo | out of the fleet | Used directly by me, occasionally. Not capacity. |

That alkaspire row is the whole lesson about heterogeneous hardware. It is a perfectly good machine for shell queries, `git` reads, API calls and coordination. It cannot run a test suite, and a dispatcher that does not know that will send it one, twice, and believe the second failure is a flaky test.

## The question you are not there to answer

The chat bridge is the part I would keep if I had to throw everything else away.

An interactive agent that stops and asks you something is helpful. An agent running headless on another machine that stops and asks you something is **stopped**, and it will stay stopped for as long as it takes you to walk past that terminal.

So the blocking question goes to my phone. Telegram as a direct message from a bot, or Mattermost as a channel on my own server. One or the other, never both at once.

This changes what the fleet is for. Without the bridge it is a batch system that works until it has a question. With it, work continues while I am away from the desk, which is most of the day.

## Who decides what

With several agents working you need an escalation chain, or every small fork in the road comes to you and you become the bottleneck you were trying to remove.

Mine is three levels: worker, then a coordinating session, then me. Never worker straight to me.

**The coordinator decides:** design forks inside a ticket, merge order, sizing, priority, scope, which reviewer runs, whether something gets filed, whether to park or proceed.

**I decide, and only these:** money, customer-visible risk I have not already accepted, anything irreversible or outward-facing, licensing and ownership, and work outside the current milestone.

Everything else is somebody else's call, and the point of writing the list down is that it stops being renegotiated every time.

There is one exception and it is not an escalation. **If a coordinator instruction contradicts a written rule, the worker says so and does not comply.** That is a correction to make, not a question to route. This has happened repeatedly and the worker has been right every time, which tells you something about how much weight a coordinating session's opinion should carry against a rule written down after an incident.

## State has to live where you can see it

This is the part I got most wrong, and the fix is almost embarrassingly simple.

On 27 September four sessions were actively working. The GitHub issue tracker showed four issues with nobody assigned, no label, and no comment. Every one of those sessions had announced itself in a chat message to the coordinator, which is invisible to anyone looking at the repository and gone the moment the conversation is cleared.

So the rule now is: **claim the issue on the issue.** A comment naming the machine and the branch, plus a working label, and remove the label when you stop.

Two details that are not obvious:

**Do not use the assignee field.** Every session pushes as the same identity, so assignment cannot distinguish them. A comment can.

**An unpushed branch is invisible to the forge, and so is a worktree.** That is exactly how two sessions end up editing one file. If the only evidence of your work is on a local disk, nobody else can route around you.

The general shape: the forge is the shared memory. Chat is not. A message is a pointer to work; the work itself has to live somewhere that survives a restart.

Which is also why I cap messages at about 150 words in both directions, and ask for reports in a fixed shape:

```
STATE:  <ticket> building | suite-running | PR-open | merged | blocked
SHA:    <sha>
COUNT:  <the command> -- N passed / M skipped
HOLDS:  <files you have uncommitted, or none>
NEEDS:  <the one decision you want, or nothing>
```

The `COUNT` line is [post 2]({{ site.baseurl }}/2026/10/never-trust-the-output.html) showing up in the plumbing. Never "the suite passed." The command and the numbers, or it does not count.

`HOLDS` and `NEEDS` exist because of the two states that are invisible from outside: a session sitting on uncommitted work, and a session waiting for an answer. **A session that asked a question and is waiting looks exactly like a finished one.** If you are blocked on a person, the last line of your report has to say so, or your pane reads as free capacity.

## What it costs to leave a session open

A measurement that surprised me.

I checked what each live session was carrying in context. Six of them were holding an average of **400,651 tokens** each. A seventh was on **808,481**.

Every one of those tokens is read again on every single call that session makes. A session that finishes a task and stays open drags the whole finished task through every later turn, forever, for nothing.

Resetting those to around 150k would have cut roughly **26% of the fleet's total usage**, at no cost in capability whatsoever.

So: when a task is done, clear or compact before starting the next one. Clear when the next task is unrelated, because none of the old context helps and all of it is being paid for. Compact when the thread still matters.

One warning that cost me real work before I wrote it down: **before clearing, write down anything that exists only in the conversation.** A decision, a constraint somebody gave you, a measurement you took. Those go to a file, an issue, or a pull request body. Anything still only in the transcript when you clear is gone, and afterwards it will not be obvious that it ever existed.

## Should you build this?

Probably not, and I want to be straight about it.

The fleet is worth it for me because I have a specific situation: several repositories under active development, work that arrives as GitHub issues, long test suites, and a habit of being away from the desk. The parallelism pays for the coordination overhead.

If you have one project, this is strictly worse than one agent and good rules. The coordination cost is real and it is paid in a currency you will not expect: not compute, but **the work of knowing what is happening**. Every mechanism in this post exists because a session's true state was not visible from outside. The twelve-hour wedge, the four unlabelled issues, the OOM on the small box, the 800k-token session. None of those were about code quality. All of them were about observability.

So the honest ordering is the order of these posts.

1. Write the rules down. One text file, costs nothing, helps immediately.
2. Build the verification. This is where the actual return is, and it needs no extra machines.
3. Add parallelism only when you are repeatedly waiting on work that could have run while you slept.

If you do get to step 3, the thing to budget for is not the agents. It is the reporting.
