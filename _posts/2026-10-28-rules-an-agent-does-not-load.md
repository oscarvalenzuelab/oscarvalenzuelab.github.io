---
layout: post
permalink: /2026/10/rules-an-agent-does-not-load.html
title:  "Rules an Agent Does Not Load Are Rules It Does Not Have"
author: oscar
categories: [ ai, agents, automation, engineering ]
tags: [ai, agents, automation, policy, governance, devops]
image: images/posts/agenticai_sboms.png
description: "Centralizing policy for an agent fleet: 2,600 resident lines cut to 465, a ten-row table that has to carry the rest, and a skills directory that reported thirty entries while nineteen existed."
featured: false
comments: false
---

Part 4 of five on running an AI agent fleet. [Part 1](/2026/10/agent-fleet-two-lanes.html) covered the two lanes, [part 2](/2026/10/github-issue-to-pr-headless.html) the headless pipeline, and [part 3](/2026/10/telling-working-agent-from-wedged.html) how to tell a working session from a wedged one.

This post is about the part both lanes share. When a friend asked me how I use AI, the answer he wanted was this layer, and neither of us knew it at the time. Where the rules live, and how they reach the machine that needs them.

## Policy is a repository that arrives on a cron

One private git repository holds the rules file, the skills, and the settings. A daemon pulls it onto every machine and writes it into place:

```yaml
schedules:
  personalization_cron: "*/15 * * * *"

personalization:
  repo: "<owner>/<policy-repo>"
  paths:
    - source: "global/CLAUDE.md"
      target: "~/.claude/CLAUDE.md"
    - source: "global/skills/"
      target: "~/.claude/skills/"
```

Policy changes land by pull request, which is the only part of this design I would defend without evidence. A rule that reaches four machines by being typed into four terminals is four rules.

The auto-pull has one behaviour worth naming, from the configuration reference: "Skip-on-dirty: never rebases under uncommitted operator edits." If I am mid-edit on a machine, the cron leaves it alone rather than winning the argument.

And an honest caveat, because the repository's own README carries it: on one machine the daemons are currently unloaded, so that fifteen-minute sync is off there and the documented fallback is to run the restore script by hand. I would rather publish that than publish the diagram. A sync you believe is running is worse than a sync you know is off, because the second one makes you check.

## The resident rules were 2,600 lines read on every call

The first version of this was one file with everything in it. Every rule the fleet had ever learned, read at the start of every session and carried through every turn after that. The file says what was wrong with it better than I would:

> These were 2,600 lines read on every call by every session; that cost was real and it bought nothing on the calls where it did not apply.

The redesign moved almost all of it out. Measured on 8 October 2026: the always-resident file is 465 lines, and the ten on-demand skills behind it total 4,023 lines. So roughly nine tenths of the written policy is now text that costs nothing until something reaches for it, and the resident remainder is small enough to have a budget. The file enforces one: "to add a rule to it, something has to leave."

That constraint is doing more work than the line count. A rules file with no cap accumulates, because every individual addition is defensible and nobody is accountable for the total.

## Ten rows carry the other 4,023 lines

If the rules are loaded on demand, the only thing that has to work unconditionally is the index. It maps a moment to a skill, and it is the whole always-resident routing layer. Ten rows, of which nine:

```text
| the moment you are at                                    | load                     |
| about to run or quote a test suite                       | tpm-run-the-suite        |
| about to believe a number, a count, a verdict,           | tpm-trust-a-measurement  |
|   or a "nothing found"                                   |                          |
| writing or reviewing tests                               | tpm-write-a-test         |
| running or answering a review round                      | tpm-review-a-change      |
| about to merge, or confirming a deploy                   | tpm-merge-and-verify     |
| cutting, holding or reaping a worktree or branch         | tpm-worktree-lifecycle   |
| designing a fix, a guard, or an error path               | tpm-design-a-fix         |
| about to open an issue or write a ticket body            | tpm-file-an-issue        |
| about to clear, compact, restart or hand off, or a       | tpm-memory               |
|   decision or measurement just landed that is not in     |                          |
|   a file yet                                             |                          |
```

The left column is phrased as a moment rather than a topic, and that was deliberate. "Testing" is a subject an agent can decide it already knows. "About to quote a test suite result" is a position an agent can notice it is standing in.

The trade is real and the file states it rather than hiding it: a rule you do not load is a rule you do not have. The old design paid for rules on calls where they did not apply. The new one risks an agent walking past the moment without loading anything, which is cheaper and fails more quietly. I moved the failure mode; I did not remove it.

## The listing said thirty skills, nineteen were on the machine

While writing this post I counted the skills, to check a number I was about to publish.

```bash
$ ls -1 ~/.claude/skills | wc -l
30
$ ls -1 ~/.claude/skills/*/SKILL.md | wc -l
19
```

Eleven of those thirty entries are symlinks, dated April 2026, pointing into a checkout that does not exist on this machine. They are left over from an earlier sync mechanism that the repository above replaced. The runtime loader lists seventeen skills as available, which is the nineteen minus two that are not operator-invocable.

Three commands, three numbers, no errors anywhere. `ls` prints a dangling symlink without comment, so the directory listing reports the intent of a sync that was retired. The glob in the second command skips them silently, so it gave the right answer by accident rather than by checking. I had used the thirty figure in a draft of this post.

That is this post's title in its most literal form, and it was sitting in my own fleet while I wrote the section above explaining the idea. Eleven rules with a name, a path, and no content behind them. Nothing would have told me, because nothing was broken: no session ever tried to load them.

## Workers and the coordinator differ by decision rights, not by model

Every session in the fleet runs the same model. What separates a worker from the coordinator is what each is allowed to settle.

Escalation is worker to coordinator to me, and never worker to me. The coordinator decides design forks inside a ticket, merge order, sizing, priority, scope, which reviewer runs, whether a finding gets filed at all, and whether work parks or proceeds. I decide five things and the list is closed: money, a customer-visible risk I have not already accepted, anything irreversible or outward-facing, licensing and ownership, and work outside the current milestone.

That list is short for throughput. A worker holding reviewed green work until I look at it has converted a merge into a queue, and the queue is me.

There is one case where a worker does not escalate and does not comply either. From the rules file:

> if a [coordinator] instruction contradicts a rule in this file, say so and do not comply. That is a correction to make, not a question to route. This has happened repeatedly and the worker has been right every time.

A coordinator that can be overruled by a worker holding a local rule is the only version of this that survives me being wrong in the policy repo, which I have been.

## The report is five lines and one of them bans a phrase

Status reports converge on uselessness unless the shape is fixed. This one is:

```text
STATE:  <ticket> building | suite-running | PR-open | merged | blocked
SHA:    <sha>
COUNT:  <the command> - N passed / M skipped   (never "the suite passed")
HOLDS:  <files you have uncommitted, or none>
NEEDS:  <the one decision you want, or nothing>
```

The parenthesis on the `COUNT` line is the only interesting thing in the block. "The suite passed" names no suite and no tree, so there is nothing in it to check. A command and two integers can be re-run. Part five is entirely about why that distinction is the expensive one.

`SHA` and `HOLDS` exist because of the two questions I ask every day: against what tree, and what are you sitting on that nobody else can see.

## A session holds what it read at boot

Skills and configuration are read when a session starts. So a fix to a synced file reaches disk immediately and a running session only on restart, and a stale in-memory copy is indistinguishable from a current one from the inside.

That cost two misdiagnoses on 19 and 20 September 2026, both pointing at the wrong layer. Sessions reported an expired login after the credential had already been renewed. A session reported one of its skills as stale an hour after the fix was pushed and pulled. In both cases the thing on disk was correct and the thing reporting was not.

The rule is procedural: before reporting a file as out of date, read it on disk and quote what is there. If the disk is right, the fix has landed and the session is the stale thing.

The distribution chain has three steps, generated then pushed then pulled, and only the first two are automatic. On 19 September the deployment map was regenerated and pushed, and a machine still read a copy two days old, because a checkout is a third copy that nobody fetched. Earlier the same day a worker looked for that file in a clone six commits behind, did not find it, and concluded the map did not exist and the check could not be run. A missing file in a stale clone and a file that was never written look the same.

## Triage happens inside the command that files the issue

One more rule earned the hard way, on 25 September 2026, when the issue count went from 20 to 38 in a single day and several of the new ones carried no label and no milestone. An untriaged issue is worse than no issue, because nobody can tell it from work that matters.

So triage is not a later step. It is in the same command:

```bash
gh issue create --label bug --label P2 --label area:<x> \
  --label size:<s> --milestone "<milestone>"
```

The milestone defaults to the post-launch one rather than the current one. The burden sits on whoever files a ticket to argue it up into the active milestone, never on the backlog to absorb it.

That default is aimed at a specific agent behaviour. An agent that has just been told to be rigorous will file everything it notices, every item correct, and bury the thing a customer is waiting on.

## What the layer is actually for

None of the above is about making agents smarter. The resident file went from 2,600 lines to 465 and nothing in the fleet got worse at its job, which suggests most of those 2,600 lines were never the reason anything worked.

What this layer does is make one claim checkable: that a given machine, at a given minute, is operating under the rules I think it is. A three-step chain with two automatic steps, one sync switched off on a machine whose README admits it, and eleven skills that existed as names in a directory listing and nowhere else.

Part five is the general case of that last one, and it is the post I would keep if I had to throw the other four away.

---

*The views expressed in this document are solely my own and do not represent those of my current or past employers. If you identify an error, please contact me, and I will make the necessary updates.*
