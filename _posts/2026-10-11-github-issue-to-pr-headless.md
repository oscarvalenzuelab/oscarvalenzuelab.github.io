---
layout: post
permalink: /2026/10/github-issue-to-pr-headless.html
title:  "My AI Agent Fleet, Part 2: From a GitHub Issue to a Pull Request Without Watching a Terminal"
author: oscar
categories: [ ai, agents, automation, engineering ]
tags: [ai, agents, automation, ctrlrelay, github, ci, devops]
image: images/posts/agentic_sboms_datafeed.png
description: "The headless half of an agent fleet: what the agent may pick up, the rules it works under, the review loop, and how a question reaches me and comes back."
featured: false
comments: false
---

Part 2 of five on running an AI agent fleet. [Part 1](/2026/10/agent-fleet-two-lanes.html) covered the two lanes and why each one fails differently.

This post is about the headless lane. An issue gets assigned on GitHub, and some time later a pull request appears with tests and a description. Nobody watched a terminal for any of it. If a decision is needed, the agent escalates a question to me, the human in the loop, and once I answer, the work continues from that decision.

The headless lane runs on [ctrlrelay](https://github.com/AInvirion/ctrlrelay), open source under the Apache 2.0 license. I'm the author, and it keeps changing: much of that happens automatically, based on what my team and I learn along the way. Most of the work it handles is DevOps in the broad sense: bug fixes, documentation and editorial changes, releases, CI and workflow improvements, Dependabot updates, security fixes, and version upgrades. The reasons behind each change go in its changelog, which is where most of what follows comes from.

## How it is put together

CTRLRelay (Control Relay) is a few small parts running on one Linux/MacOS machine, with no web server and no queue:

- **The poller** is a background service. Every five minutes it asks GitHub for new issues across the repositories in its config, and it also runs scheduled jobs, like the daily security sweep.
- **The pipelines** do the work. The dev pipeline turns an issue into a pull request. The secops pipeline runs on a schedule and handles Dependabot alerts and pull requests.
- **The agent** is Claude Code running headless (`claude -p`), in its own git worktree, one per issue.
- **The state database** is a single SQLite file. It holds the repository locks, the sessions, and the questions waiting for me.
- **The bridge** is a separate service that carries those questions to me on Telegram or Mattermost and brings my answers back. It can restart without touching work in progress.

## What the agent is allowed to pick up

Work comes in through two gates, and each one has its own rules.

The first is issues. Before any agent runs, the poller decides which issues it may take. That check is code, not prompt, so nothing written in an issue can talk its way past it. On every repository, an issue needs two things: it is assigned to me, and it carries the `ctrlrelay:auto` label. Assignment alone was too loose: old self-assigned reminders, like "look into pricing", would go into the pipeline the first time the poller saw a repository. A `manual`, `operator`, or `instruction` label always wins and keeps the agent off the issue.

The second is the security sweep, which runs every morning at six on every repository, with no assignment and no label. It looks only at open Dependabot alerts, pull requests opened by Dependabot, and my own pull requests that turn Dependabot on. That is the work the Dependabot keys below are for.

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

The Dependabot keys work differently from the labels. They are not checked in code; they become instructions in the sweep's prompt. `auto` tells it to merge a patch bump once CI passes, `ask` sends me the question, and `never` leaves the pull request alone. That is an instruction, not a lock, which is why the decision about which issues enter the pipeline at all lives in code.

Credentials are referenced by environment variable name, never by value. The config holds `bot_token_env: "CTRLRELAY_TELEGRAM_TOKEN"` and the process reads the name. That is a small thing that makes the config file committable, which is what makes a new machine a ten-minute job.

## The pipeline

From my side it looks like this. I assign an issue to myself on GitHub and add the `ctrlrelay:auto` label. ctrlrelay comments on the issue to say it took it, so anyone else looking knows. If the issue is unclear, I get a question on Telegram or Mattermost, and the work waits for my answer. Otherwise, some time later a pull request shows up with tests and green CI. The agent doesn't merge it. That part stays with me.

Under the hood, the agent is given a numbered workflow, and the wording matters more than it looks:

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

Two issues in the same repository can be worked at the same time, so the dev pipeline uses a lock to make them take turns whenever they touch git. While an agent waits for CI, it gives the lock back, so one issue's ten-minute CI run doesn't hold up the other. The only real decision was how long to wait to get the lock back. Too short, and a healthy session working on another issue makes this one fail. No limit, and a session that crashed while holding the lock blocks the repository forever. I settled on about an hour.

## The checkpoint is a file, and it is written by rename

The agent reports by writing a small JSON file with one of three statuses, `DONE`, `BLOCKED_NEEDS_INPUT`, or `FAILED`. It is written to a temporary path and renamed into place, so a reader either sees the previous complete state or the new complete state. ctrlrelay has to survive its own restart and still know where every session got to, which rules out keeping that state in memory.

The protocol exists because its absence is invisible. An agent that finishes its work, exits 0, and writes nothing looks exactly like an agent that crashed. On 16 September 2026 that is what happened: a resumed session merged three of four pull requests, wrote no checkpoint, and I was told the resume had failed. The first prompt spent about thirty lines on the checkpoint rules; the resume prompt was 58 characters and said only "continue". The resume prompt now carries the same rules.

That is the general shape of a headless failure. A wrong answer I would catch. What I get instead is a report in the correct format, delivered on time, about something other than what happened.

## The rules the agent works under

The numbered workflow is only part of what the agent reads. ctrlrelay also syncs my shared rules and skills into `~/.claude/` from the same Git repository the interactive lane uses, so the headless agent starts with the same rules as the sessions I drive by hand. Part four covers how that rulebook is organized.

The prompt adds a few rules of its own. If the issue's comments disagree with its body, the comments win, because nobody goes back and edits the body after changing their mind in the thread. If the two can't be reconciled, the agent asks instead of picking one. The security sweep is fenced to the three things listed earlier: open Dependabot alerts, pull requests opened by Dependabot, and my own pull requests that turn Dependabot on. A stranger's pull request is not its business, so it never goes looking for one.

## The review loop

The agent waits for CI with one exact command, `ctrlrelay ci wait`, instead of writing its own loop. Its own loops had gone wrong before, with inverted conditions and pipes that swallowed exit codes, and burned the whole session timeout on pull requests that were already green. The command has three exit codes: 0 means every check passed, 1 means one failed and the agent should fix it and wait again, and 2 means the timeout hit while checks were still running. That third one is not a failure. It is an unfinished question, and treating it as a failure would have agents "fixing" green pull requests.

The agent saying `DONE` is not the end. ctrlrelay then checks the pull request itself: CI green, and no merge conflicts. If either check fails, it resumes the same agent session with what is wrong and checks again, up to three times.

Once CI is green, Codex is asked to review the branch in a read-only sandbox and posts what it finds as a comment on the pull request. The comment is labeled as unverified opinion, on purpose. The agent being reviewed wrote every file the reviewer reads. I tried twice to mark a pull request as reviewed, and both times the agent's own files fooled the marker; the simplest was a committed file telling the reviewer to reply "No findings". So the findings get published, and nothing claims a review took place. Part five comes back to why a clean verdict is the cheapest thing a broken reviewer produces.

## How I stay in the loop

When the agent hits something it should not decide alone, it writes `BLOCKED_NEEDS_INPUT` with a question into its checkpoint and exits. The bridge posts the question to Telegram or Mattermost, and I get a notification on my phone. I answer, and using the app's reply feature ties my answer to the right question when several are open. ctrlrelay then resumes the same agent session with my answer, so it picks up with everything it already knew. A session can go around that loop five times before ctrlrelay gives up on it.

A question does not have to be answered right away. It is stored in the state database, so an answer that arrives hours later can still restart the work, for up to 48 hours by default. When the session ends, I get one last message: the pull request is ready, or it failed and why. Reviewing and merging the pull request is still my job.

## What failed quietly

Two bugs from the changelog show how this kind of system breaks.

The first was in `ci wait` itself, the command built to stop agents misreading CI. In 0.8.1, dated 4 September 2026, it called `gh pr checks --json`, and the `gh` package on Ubuntu 24.04 does not support `--json` on that subcommand. Every poll failed, the polling layer treated each failure as temporary and retried, and every call used its whole timeout and then reported failure, whatever the state of the pull request. The retry logic was correct. The thing it was retrying could never succeed. Part five comes back to that pattern, because it is the most expensive kind of bug in this whole system.

The second was a pair of fetch bugs, two days apart. ctrlrelay's fetch refuses to rewrite a branch, so it can never throw away unpushed work. Dependabot rewrites its branches every time it rebases them, and `git fetch` reports the refusal as a failure even when everything else was fetched, so every rebase killed the sweep. The fix in 0.11.2 tolerated those refusals. Two days later, 0.11.3 narrowed it: tolerating every refusal also covered `main`, so after a force-push to `main` every session would have run on a tree GitHub no longer had, with one log line to show for it. Now a refusal on the default branch stops the run. The first bug was loud: a sweep aborted and told me so. The second would have been silent, and I caught it only because I went looking.

## What the operator has to do

Back to the number from part one. On one deployment, 46 questions were posted to the chat bridge, 4 were ever answered, and 16 were still sitting live. Every piece of machinery above is there so an agent can ask. None of it makes anyone read.

Writing the code was never the part that needed engineering. The lock, the rename, the three exit codes, and the fetch fix are all machinery for one job: making sure that when an agent stops, something says so.

Part three is the other lane, where the same question is harder because the thing that stops is a session with a person attached to it.

---

*The views expressed in this document are solely my own and do not represent those of my current or past employers. If you identify an error, please contact me, and I will make the necessary updates.*