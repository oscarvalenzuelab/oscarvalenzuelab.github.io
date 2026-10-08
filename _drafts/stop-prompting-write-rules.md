---
layout: post
published: false
draft: true
permalink: /2026/10/stop-prompting-write-rules.html
title:  "Stop Prompting. Write Rules."
author: oscar
categories: [ ai, development, bestpractices, agentic ]
tags: [ai, agentic, claude, development, bestpractices, workflow, automation]
image: images/posts/rules-not-prompts.png
description: "Most advice about AI coding agents is advice about prompting. The thing that actually changed my results was writing down the rules of my workshop in a file the agent reads first."
featured: false
comments: false
---

Most advice about getting good work out of an AI coding agent is advice about prompting. I think that is the wrong layer. The thing that changed my results was not better prompts. It was writing down the rules of my workshop in a file the agent reads before it touches anything, and then being ruthless about what earns a place in that file.

This is the first of three posts about how I actually work. This one costs you nothing but a text file. The second is about not trusting the output. The third is about running several machines at once.

## A rule that was correct, and still broke

One of my rules is that no AI tool name appears in a commit message, a pull request or a code comment. The work should read as mine, because it is. So the rule says: before you push, scan what you wrote and remove any hits.

The prose said scan the diff. The command that got written scanned whole files.

On 23 September it fired four times on one site project. All four were real text in the product, none of it written by an agent: a page about AI crawler policy that mentions `Microsoft Copilot` and `ClaudeBot` by name, and a line of CSS containing `cursor:pointer`.

Four false positives is not the interesting part. This is: the fix for a false positive is always to relax the pattern. Do that twice and the scan no longer scans. A check that cries wolf gets quietly disarmed by the person it keeps interrupting, and then it is still there, still green, still checking nothing.

So the rule now carries the exact command, and the command names the diff:

```
git diff origin/main...HEAD -U0 | grep '^+' | grep -vE '^\+\+\+' \
  | grep -nIE "[Cc]odex|[Cc]laude|[Cc]opilot|[Cc]ursor|AI-(reviewed|generated|assisted)"
```

That is the whole lesson of this post in one incident. A rule written as an intention gets implemented differently every time. A rule written as the exact command gets implemented the same way every time, including by me at midnight.

## Why a file beats a prompt

A good prompt and a good rules file contain roughly the same words. The difference is not the words. It is three properties the file has and the prompt cannot.

**It survives you forgetting.** A prompt is typed by a person who currently remembers the problem. The whole reason a rule exists is that at some point nobody will remember. My file says to return the shared checkout to a clean state when a ticket lands, "so the checkout you leave behind is current for whoever picks it up, including you tomorrow, with no memory of today." That clause is the entire argument for files over prompts.

**It applies when you are not watching.** Scheduled jobs run against my repositories overnight. Nobody is typing a prompt at 3am. The file is the only thing present.

**It is reviewable.** A prompt is gone after the conversation. A file can be read, argued with, and corrected. Mine is in git, which means I can see when a rule was added and go back and read the incident that added it. Several of my rules were wrong when written, and the file is where that got caught.

There is a fourth property that only shows up later. Because the file is a shared artifact, it can be wrong *about the world* and not just about style. I once had a line of reasoning in my own documentation that said two pieces of work going to the same customer meant a copyleft obligation did not apply between them. That is false. Copyleft binds whoever receives the code, not whoever ends up holding the product. It was written down, which is why it got found and corrected, and correcting it changed a design. You cannot catch that in a prompt, because a prompt is never read twice.

## The file has a cap

Here is where most people go wrong, and I went wrong for months. Every lesson gets added, nothing ever leaves, and the file grows.

Mine reached about 2,600 lines. Every one of those lines was read on every call by every session. The cost was real, and on the calls where a rule did not apply it bought exactly nothing. A rule about how to reap a git worktree was being paid for during a conversation about a CSS bug.

So I split it. The file now holds only what must always be true, and everything else became a skill that loads on demand. The always-loaded part is a table of moments:

| The moment you are at | Load |
| --- | --- |
| about to run or quote a test suite | `run-the-suite` |
| about to believe a number, a count, or a "nothing found" | `trust-a-measurement` |
| writing or reviewing tests | `write-a-test` |
| running or answering a review round | `review-a-change` |
| about to merge, or confirming a deploy | `merge-and-verify` |
| cutting or reaping a worktree or branch | `worktree-lifecycle` |
| designing a fix, a guard, or an error path | `design-a-fix` |
| about to open an issue or write a ticket body | `file-an-issue` |
| about to clear, compact or hand off | `memory` |

Note what the left column is. It is not topics. It is **moments**, written as the thing you are about to do. "About to run a test suite" is recognisable from the inside. "Testing best practices" is not, because you never notice yourself entering it.

The trade is honest and worth saying out loud: a rule you do not load is a rule you do not have. On-demand loading means you have to reach for it, and sometimes you will not. I accept that, because a 2,600-line file that gets skimmed is not better than a 60-line file that gets read.

And the cap has teeth. To add a rule to the always-loaded file, something has to leave. That forces the question nobody asks otherwise: is this rule still earning its line?

## How to write a rule that holds

Four properties. Every rule of mine that works has all four, and every rule that got ignored was missing one.

**1. It names the moment, not the topic.** "Stage files by name. Never `git add -A`, `git add .` or `git commit -a`." You know when you are staging. You do not know when you are doing version control hygiene.

**2. It carries the incident.** The staging rule continues: "Checkouts are shared and scheduled jobs fire while someone is mid-edit." That clause is why the rule exists, and it is the difference between a rule you follow and a rule you work around. A rule with no reason attached gets overridden the first time it is inconvenient, because nothing tells you what you are risking.

**3. It is executable where it can be.** Not "check the deployment map is current" but the command that fetches it from the remote, plus the instruction to read the `Generated` timestamp at the top. I learned this one the hard way: the map was regenerated and pushed, and a machine still read a version two days old, because a local checkout is a third copy that nobody fetched. Another session looked in a stale clone, found no such file, and concluded the map did not exist. Absence of a file in your clone is a fact about your clone.

**4. It says what to do when the check cannot answer.** This is the one almost everybody misses, and it is the one that has cost me the most. A check whose failure mode is silence is indistinguishable from a healthy one.

My worst example of the fourth: a loop waiting for a build to finish.

```
until [ "$(gh run list --branch <b> --limit 1 --json status -q '.[0].status')" = "completed" ]
do sleep 25; done
```

It ran from a scratch directory, which is not a git repository. The command failed with "failed to determine base repo", printed nothing, and the comparison became `"" = "completed"`. Forever. It wedged a machine for twelve hours, and from the outside it looked exactly like a session patiently working.

The mirror image bit me two days later. `gh pr checks` exits non-zero *while checks are still pending*, so a loop that breaks on a non-zero exit leaves on the first poll and prints a pending result that reads like a settled one. One loop could never end, the other ended instantly, and both produced a confident wrong answer about the same thing.

So the rule is now three clauses instead of one: a deadline and an iteration cap always; break on a non-zero exit only when non-zero genuinely means "I could not answer"; and know how many answers you expect, because right after a force-push the checks do not exist yet, and a loop waiting for zero pending results will find zero results and report success against an empty answer sheet.

## Your first rules file

Do not copy mine. Almost every line in it is the scar of something that happened to me, and a rule without its incident is cargo cult.

Start with this instead. It is short on purpose.

```markdown
# Rules for working in this repo

## Before you change anything
- Branch first. Never commit to main.
- Stage files by name. Never `git add -A` or `git commit -a`.

## Before you tell me something passed
- Quote the command and the counts: "pytest -q gave 412 passed, 3 skipped".
- Never "the tests pass". I cannot check that sentence.
- "Not found", "zero results" and "command not found" are facts about
  the search, not about the world. Say which one you got.

## Before you push
- <your own scan command here, pointed at the diff, not at whole files>

## When you are unsure
- Do the parts that do not depend on the answer, then ask one question.
```

Then add one rule per incident, and only after an incident. Write the date and what happened next to it. In six months that annotation is the most valuable thing in the file, because it tells you which rules you can delete.

Two lines in that starter are worth more than the rest combined, and they are both in the "something passed" section. Making the agent quote the command and the numbers is what turns a claim into evidence. And treating a failed search as a statement about the search rather than about the world is the single habit that has saved me the most wasted work.

Which is the whole subject of post 2.

## What I left out

This post is deliberately the cheap one. Everything in it works on one laptop with one agent and a text file, and none of it requires my setup.

Three things I skipped on purpose:

- **Verification.** Rules tell the agent what to do. They do not tell you whether it did it. That is post 2, and it is the one that matters most.
- **The fleet.** I run several machines with agents working in parallel, coordinated by one of them, with work reaching me on my phone. That is post 3, and it is useless until posts 1 and 2 are in place.
- **One rule that only applies to me**, but it is my favourite, so here it is anyway. A running agent holds whatever it read when it started. So a fix to a rules file reaches the disk immediately and a running session only on restart, and a stale copy is indistinguishable from a current one from the inside. Twice in two days I had sessions reporting a credential as expired after it had been renewed, and a skill as stale an hour after the fix was pushed. The rule that came out of it: before reporting anything as out of date, read the file on disk and quote what is actually there. If the disk is right, the fix landed and you are the stale thing.

That last one generalises further than agents, and I think about it more than I expected to.

If you want the version of this that is about expectations rather than mechanics, I wrote [How to Use AI Without Wasting Time and Money]({{ site.baseurl }}/2025/11/how-to-use-ai-without-wasting-time-and-money.html) last year. This series is the other half: not what to expect, but what to actually set up.
