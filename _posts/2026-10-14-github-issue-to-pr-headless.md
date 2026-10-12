---
layout: post
permalink: /2026/10/github-issue-to-pr-headless.html
title:  "My AI Agent Fleet, Part 2: From a GitHub Issue to a Pull Request Without Watching a Terminal"
author: oscar
categories: [ ai, agents, automation, engineering ]
tags: [ai, agents, automation, ctrlrelay, github, ci, devops]
image: images/posts/agentic_sboms_datafeed.png
description: "The headless half of an agent fleet: a poller, a worktree, a checkpoint file written by rename, and the two fetch bugs that sat on opposite sides of the same line of code."
featured: false
comments: false
---

Part 2 of five on running an AI agent fleet. [Part 1](/2026/10/agent-fleet-two-lanes.html) covered the two lanes and why each one fails differently.

This post is about the headless lane. An issue gets assigned on GitHub, and some time later a pull request appears with tests and a description. Nobody watched a terminal for any of it. If a decision is needed, the agent escalates a question to me, the human in the loop, and once I answer, the work continues from that decision.

The headless lane runs on [ctrlrelay](https://github.com/AInvirion/ctrlrelay), open source under the Apache 2.0 license. I'm the author, and it keeps changing: much of that happens automatically, based on what my team and I learn along the way. Most of the work it handles is DevOps in the broad sense: bug fixes, documentation and editorial changes, releases, CI and workflow improvements, Dependabot updates, security fixes, and version upgrades. The reasons behind each change go in its changelog, which is where most of what follows comes from.

## How it is put together

ctrlrelay is a few small parts running on one Linux machine, with no web server and no queue:

- **The poller** is a background service. Every five minutes it asks GitHub for new issues across the repositories in its config, and it also runs scheduled jobs, like the daily security sweep.
- **The pipelines** do the work. The dev pipeline turns an issue into a pull request. The secops pipeline handles Dependabot pull requests.
- **The agent** is Claude Code running headless (`claude -p`), in its own git worktree, one per issue.
- **The state database** is a single SQLite file. It holds the repository locks, the sessions, and the questions waiting for me.
- **The bridge** is a separate service that carries those questions to me on Telegram or Mattermost and brings my answers back. It can restart without touching work in progress.

## What the agent is allowed to pick up

Before any agent runs, the poller decides what it may take. That check is code, not prompt, so nothing written in an issue can talk its way past it. On every repository, an issue needs two things: it is assigned to me, and it carries the `ctrlrelay:auto` label. Assignment alone was too loose: old self-assigned reminders, like "look into pricing", would go into the pipeline the first time the poller saw a repository. A `manual`, `operator`, or `instruction` label always wins and keeps the agent off the issue.

Per-repository policy is a short config block:

```yaml
repos:
  - name: "your-org/your-app"
    local_path: "~/Projects/your-app"
    automation:
      dependabot_patch: auto
      dependabot_minor: ask
      dependabot_major: never
      accept_foreign_assignments: false
      exclude_labels: ["manual", "operator", "instruction"]
      require_labels: ["ctrlrelay:auto"]
      ci_wait_timeout_seconds: 600
```

The Dependabot keys work differently. They are not checked in code; they become instructions in the agent's prompt. `auto` tells it to merge a patch bump once CI passes, `ask` sends me the question, and `never` leaves the pull request alone. That is an instruction, not a lock, which is why the decision about what enters the pipeline at all lives in code.

Credentials are referenced by environment variable name, never by value. The config holds `bot_token_env: "CTRLRELAY_TELEGRAM_TOKEN"` and the process reads the name. That is a small thing that makes the config file committable, which is what makes a new machine a ten-minute job.

## The pipeline

From my side it looks like this. I assign an issue to myself on GitHub and add the `ctrlrelay:auto` label. ctrlrelay comments on the issue to say it took it, so anyone else looking knows. If the issue is unclear, I get a question on Telegram or Mattermost, and the work waits for my answer. Otherwise, some time later a pull request shows up with tests and green CI. The agent doesn't merge it. That part stays with me.

Under that, the agent is given a numbered workflow, and the wording matters more than it looks:

```text
1. Validate the issue still applies to the current codebase
2. If anything is unclear, signal BLOCKED (see below) to ask for clarification
3. Plan and implement the fix using TDD
4. Push the branch and open a PR that references the issue
5. Before signaling DONE, verify the PR is mergeable:
   - Wait for CI by running EXACTLY this command (do NOT improvise bash):
     `ctrlrelay ci wait --pr <PR> --repo {repo} --timeout {ci_wait_timeout_seconds}`
   - Run `gh pr view <PR> --json mergeable,mergeStateStatus`
6. Signal DONE only when the PR is green AND conflict-free, with the PR URL.

Do NOT merge the PR - wait for human review.
```

Step 1 is there because an issue filed three weeks ago may already be fixed. Step 2 is there because an agent that guesses at an ambiguous ticket produces a pull request that looks finished. The capitalized "EXACTLY" in step 5 has a story behind it, below.

Around that, the dev pipeline takes a per-repository lock in SQLite, cuts a worktree from a bare clone, spawns the agent, and reads its checkpoint. Two issues in the same repository can be in flight at once, and both write to the same bare clone on disk, so the lock makes them take turns. It is a single row with a uniqueness constraint, so acquiring it is an `INSERT` that either succeeds or raises `IntegrityError`. The agent gives the lock back while it waits for CI, because waiting doesn't touch git, so one issue's ten-minute CI run doesn't hold up the other. Getting it back is the interesting part:

```text
# Reacquire policy for the repo lock after the unlocked verification phase.
# Verification releases the lock because CI polling doesn't touch git; when
# we need to run `request_fix` (which spawns claude + writes worktree) we
# must get it back. Contention here is NOT necessarily brief - a peer
# session may be running its own full claude pass on another issue in the
# same repo, which can take tens of minutes. A short retry budget would
# spuriously fail healthy same-repo parallelism; bound the wait at roughly
# one hour, long enough to outlast a normal peer run and short enough to
# escape a stuck/SIGKILL'd peer whose lock row is lingering.
```

Both failure directions are named in that comment, which is the only reason the number is defensible. A short budget breaks working parallelism. No budget inherits a dead peer's lock forever.

## The checkpoint is a file, and it is written by rename

The agent reports by writing a small JSON file with one of three statuses, `DONE`, `BLOCKED_NEEDS_INPUT`, or `FAILED`. It is written to a temporary path and renamed into place, so a reader either sees the previous complete state or the new complete state. ctrlrelay has to survive its own restart and still know where every session got to, which rules out keeping that state in memory.

The protocol exists because its absence is invisible. An agent that finishes its work, exits 0, and writes nothing looks exactly like an agent that crashed. That is not hypothetical, and the source records the case:

```text
Measured case, 2026-09-16:
resumed after an "Approved", ran 27.4s, exited 0 and wrote no checkpoint
- having in fact merged three of four pull requests. The operator was
told the resume failed, which was wrong in both directions: most of
the work had landed, and the one thing outstanding was never named.
```

The cause was length. The initial prompt spent about thirty lines establishing the checkpoint contract. The resume prompt was 58 characters and said only "continue". So a resumed agent had never been told the one thing it had to do before exiting, and the report I got was wrong in both directions at once: it said nothing had landed when three merges had, and it did not name the fourth.

That is the general shape of a headless failure. A wrong answer I would catch. What I get instead is a report in the correct format, delivered on time, about something other than what happened.

## Why waiting for CI became its own command

Step 5 says "do NOT improvise bash" in capitals because improvised bash is how agents lie about CI. The prompt spells out the reason:

```text
Exit codes: 0 = all checks passed, 1 = a check failed (investigate,
fix, push, then re-run the wait), 2 = hard timeout while CI is still
pending (treat as acceptable, hand off and let the orchestrator
re-verify). Do NOT write your own `until` / `while` loops around
`gh pr checks`; those have been miswritten in the past (inverted
semantics, pipes swallowing exit codes) and burned the whole session
timeout on PRs that were already green.
```

Three exit codes, and the third one is the design decision. A timeout while checks are still pending is not a failure, it is an unfinished question, and collapsing it into exit 1 would have agents "fixing" green pull requests.

Then the command itself turned out to have the same class of bug it was built to prevent. From the 0.8.1 changelog, dated 4 September 2026: the implementation shelled out to `gh pr checks --json ...`, and `gh` 2.45.0, the current Ubuntu 24.04 package, does not support `--json` on that subcommand. Every poll failed with `unknown flag: --json`. The polling layer treats a failed poll as transient and retries. So on that platform every `ci wait` call was guaranteed to consume its entire timeout and then report failure, with no relationship to the state of the pull request.

The retry logic was correct. The thing it was retrying could never succeed. I will come back to that pattern in part five, because it is the single most expensive category of bug in this whole system.

One measurement on cost, from 0.8.0. Two `ci wait` calls accounted for 12 minutes of a 26-minute run, while the whole session generated under 35,000 output tokens. The wall clock is CI runtime, not model work, which is worth knowing before anyone optimizes the wrong half.

## Two fetch bugs on opposite sides of one line

This is my favorite pair in the codebase, because the second one is the correction to the first and they are both right.

The bare-repo fetch deliberately uses a non-force refspec, so it can never discard a previous session's unpushed commits. Dependabot rebases its branches onto a moved base, which rewrites their tips, so every rebase became a `non-fast-forward` rejection. The detail that turns that into an outage is this: `git fetch` exits 1 on a declined rewind even when every other ref applied cleanly. The setup code read the exit code as failure and aborted the session.

The 0.11.2 changelog, dated 10 September 2026, records a site-builder app whose sweep died reporting a rejection in the same fetch that had just successfully advanced `main`. Twelve bare repositories were holding a `dependabot/*` ref at that moment, each one primed to fail as soon as Dependabot rebased it.

The fix fetches `dependabot/*` with a force refspec first, because those branches are bot-authored and never carry local work, so rewinding one destroys nothing. Surviving rejections are logged and tolerated rather than raised. Real failures still raise, and the changelog is specific about why:

> Failing open there would turn an unreachable remote into a silent no-op and run pipelines against a stale tree.

Two days later, in 0.11.3 dated 12 September 2026, that fix got narrowed. Tolerating every non-fast-forward rejection also covered the default branch, which is the branch sessions are cut from. After a force-push to `main` on GitHub, the bare repo would keep its old `main`, and every session would run against a tree the remote no longer had, with a single log line to show for it. A rejection of the default branch now raises `default branch diverged from origin`.

Put those side by side. The first bug was loud: a sweep aborted and told me so. The second would have been silent, and it was the dangerous one. I caught it two days later, and only because I went looking.

## What the operator has to do

Back to the number from part one. On one deployment, 46 questions were posted to the chat bridge, 4 were ever answered, and 16 were still sitting live. Every piece of machinery above is there so an agent can ask. None of it makes anyone read.

Writing the code was never the part that needed engineering. The lock, the rename, the three exit codes, and the two refspecs are all machinery for one job: making sure that when an agent stops, something says so.

Part three is the other lane, where the same question is harder because the thing that stops is a session with a person attached to it.

---

*The views expressed in this document are solely my own and do not represent those of my current or past employers. If you identify an error, please contact me, and I will make the necessary updates.*
