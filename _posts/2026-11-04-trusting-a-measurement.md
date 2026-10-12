---
layout: post
permalink: /2026/11/trusting-a-measurement.html
title:  "Every Number an Agent Reports Is a Claim About Its Instrument"
author: oscar
categories: [ ai, agents, automation, engineering ]
tags: [ai, agents, automation, testing, measurement, devops]
image: images/posts/semantic_copycat.png
description: "Four instrument faults in one day, none of which errored: a suite that tested the wrong tree, a diff that showed merged work as missing, and a review verdict that was clean because the reviewer asked a question."
featured: false
comments: false
---

Part 5 of five on running an AI agent fleet. [Part 1](/2026/10/agent-fleet-two-lanes.html) covered the two lanes, [part 2](/2026/10/github-issue-to-pr-headless.html) the headless pipeline, [part 3](/2026/10/telling-working-agent-from-wedged.html) telling a working session from a wedged one, and [part 4](/2026/10/rules-an-agent-does-not-load.html) where the rules live.

This is the one I would keep. The four posts above it are machinery, and this is the failure the machinery does not catch.

An agent that gives a wrong answer is a manageable problem. I notice, I correct it, we move on. The expensive failure is an agent that returns a plausible value, on time, in the right format, where the value is a fact about the tool rather than about the world.

## Four faults in one day, and none of them errored

On 23 September 2026 four sessions hit four separate versions of this. The skill file records them together, because the shape matters more than the instances:

```text
- `gh pr checks` exits non-zero while checks are pending, so a `|| break`
  bound ended the wait on the first poll and printed a pending result
  that read as settled.
- An empty check list made "zero failures and zero pending" true, so an
  answer sheet with no questions on it read as a full one.
- A foreground `sleep` is blocked in some environments, so a sampler spun
  without delay and produced numbers that measured nothing.
- `pgrep -f pytest` matched the shell running the pgrep; `pgrep -x pytest`
  matched the controller and missed 3.7 GB of workers, because xdist
  workers are named `python`.
```

None of them stopped. None of them errored. Each one handed back a number that a reasonable person would have quoted.

The second is my favourite, and it is the one I now look for first. A wait loop that polls until there are zero pending checks will see zero rows immediately after a force-push, because the checks do not exist yet. Zero pending is true. The loop reports settled. An answer sheet with no questions on it scores the same as a full one, and the arithmetic is correct both times.

The pair in the fourth line is worth keeping. `pgrep -f` sees your own command line, so it matches the shell you ran it from. `pgrep -x` sees only the exact process name, and pytest's parallel workers are named `python`, so it finds the controller and misses every worker. Two flags, two wrong answers, and nobody asked the instrument what it could see.

That is the question the whole post reduces to. Before believing a number, ask what the instrument is capable of seeing, and ask what "nothing" would look like if it were broken. If a broken instrument and an empty result produce the same output, the result is not evidence.

## A suite can test a tree your branch is not in

Found on a Python scanner, 21 September 2026, and it undercuts every test count the fleet produces.

These repositories install themselves editable into the virtualenv, pinned to the main checkout. So `pytest` run from inside a git worktree imports the package from main, not from the branch you are sitting in. The source change under test is not in the tree being tested, and nothing anywhere says so.

The session that found it reported three failing tests on its work. It had none. It was running main's code against the branch's tests. The skill file names the thing that actually matters about it:

> It could as easily have been three *passing* tests over a broken change, which is the direction that ships.

The check is one line, and it is cheap:

```bash
python -c "import <pkg>, sys; print(<pkg>.__file__)"
```

If that path is not inside your worktree, the suite result is about somebody else's tree. This mattered more than it looks, because heavy tests had been moved off pull requests on that repository, which made the local suite the gate. A gate that silently tests the wrong tree is worse than no gate, because it produces a number somebody will quote.

## "Is my work in main" has no git answer on a squash-merging repository

A session almost kept a finished worktree because `git diff --stat main...e4fac60` showed 405 insertions across ten files. That reads exactly like unmerged work. The work was already merged. The three-dot form compares against the merge base, which predates the squash, and the squash commit is not an ancestor of the branch, so everything the branch did still appears absent.

Unlike a bare "not merged" verdict, this one names files and counts lines. It is the most convincing wrong answer in the set.

The obvious correction is the two-dot form, and a day later, on 24 September 2026, that turned out to be wrong in the opposite direction and would have destroyed work. A session about to delete two merged worktrees ran `git diff origin/main..<branch>` and got 71 differing files, again reading exactly like unmerged work. It was main being ahead. Those branches predated four merges, and two dots report the difference in both directions.

Neither form answers the question that was asked. The asymmetry decides how careful to be: a leftover worktree deleted on a wrong reading loses work, and one kept on a wrong reading is clutter. So the answer is to ask the forge whether the pull request merged, or to check whether the content is present in main by path. Both are more boring than a diff and both are correct.

## A clean review verdict is the cheapest thing a broken reviewer produces

The fleet runs an adversarial review round before a merge. On 24 September 2026, on a site-builder app, that round produced a confident nothing four separate ways.

The first ran at the CLI's default reasoning effort, which is `none`. It read the diff genuinely and reasoned about nothing. The same diff at high effort returned a P1.

The second started in a scratch directory holding only the diff, which is the documented bound, then reached outside it with an explicit `..` path and listed other repositories. Every finding was still anchored to the diff, so nothing went wrong that day.

The third never read the diff at all. It went looking for its own review skill, tried to launch a subagent, hit `request_user_input is unavailable in Default mode`, and wrote no output file. From outside it was indistinguishable from a clean pass.

So I added guards: the output file exists and is non-empty, the header reports the effort I set, and the transcript shows tool calls against my files. Then the fourth variant arrived and walked through all of them. The round came back with this:

```text
Which review should I run?
- Bugbot - correctness and regression review
- Security Review - security-focused review
```

It reviewed nothing. Exit code zero. The output file exists and is non-empty. A grep for the error strings a broken round emits, when it emits any, returns zero matches. Every guard passes. A review that asks a question is indistinguishable from a review that found nothing.

The fix was to stop checking for the absence of failure and start checking for the presence of success. The output has to end in a `VERDICT:` line, which is the thing a completed round always produces, rather than lacking the things a broken one sometimes says.

That generalizes past reviews. Guarding on the absence of known error strings only ever catches the failures you have already seen.

## Absence is the hardest claim an agent can make

"Not found", "zero results" and "command not found" are statements about the search rather than about the world. That sentence has been in my rules file for a while and it keeps earning its place.

The sharpest version came from a session on 25 September 2026, reconciling two runs of what everyone believed was the same suite. A test file was missing from its tree entirely. The session's own framing was better than the coordinator's guess:

> they were never collected, which is a cleaner fact than "skipped": a skip at least appears in the summary, and an absent file appears nowhere.

There is a hierarchy there, and the worst case is last. A failure announces itself. A skip appears in the summary, so it can be read and counted. An absent file produces no line anywhere: the total is lower and nothing says why. So reconciling a run against a collect is necessary and not sufficient, because a partial suite reconciles perfectly with itself, and so does a tree missing a whole test file.

The same reasoning applies to any log-watching loop. Make the sentinel something only your own command writes, anchor it to the end of the log, and negative-control it once against a log from a run still going. The words a pipeline emits generically are the ones several of its tools use: `DONE`, `OK`, `finished`, `complete`, `PASS`. A sentinel something other than the event can trigger is evidence about nothing.

## A process's age is not a conversation's age

One last pair, because it is the only one in this post where the measurement was correct and still produced a wrong decision.

On 25 September 2026 the coordinator read a `ps` line for a session on the fast laptop, saw `etime 04-13:29:48` and a hundred minutes of CPU, and told that session it had been carrying context since Sunday and should compact. The session had been cleared that morning. Clearing a session replaces the conversation and leaves the process running, so the process age answers a question nobody asked. The reply is the rule:

> a cleared session looks exactly like the one it replaced from outside

`etime` measures the process and survives every task boundary. Context is the conversation, and nothing in `ps`, in tmux, or on the forge can see it. So the compaction rule from part three cannot be enforced from outside by process age. The session is the only thing that knows, which means the mechanism has to be to ask it.

The same message carried the other half. It also supplied `lstart`, an absolute start timestamp, which becomes a duration only when combined with the current time, and the current time was the one fact nobody fetched. Read as "started 00:45" at what felt like late morning, a 78-second-old test run looked like a seven-hour wedge.

Prefer `etime` over `lstart` for a duration, because `lstart` needs a second fact to mean anything and that is the fact that goes wrong.

## The discipline is one sentence long

Across five posts, the lock and the rename and the exit codes and the refspecs and the trigger table are all the same move: forcing a system to produce different evidence for "working" and "stopped", because by default it produces the same evidence for both.

This post is that problem one level up, where the thing producing identical evidence is the measurement itself. No mechanism fixes it. What works is a habit that costs one clause per claim: say whether you read it or inferred it.

An agent that reports "the suite passed" has told me nothing I can check. One that reports the command, the tree it ran against, and two integers has made a claim I can go and disprove. Every rule in this series is a way of getting the second sentence instead of the first.

---

*The views expressed in this document are solely my own and do not represent those of my current or past employers. If you identify an error, please contact me, and I will make the necessary updates.*
