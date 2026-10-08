---
layout: post
published: false
draft: true
permalink: /2026/10/never-trust-the-output.html
title:  "Never Trust the Output"
author: oscar
categories: [ ai, development, bestpractices, agentic ]
tags: [ai, agentic, verification, testing, codereview, vid, workflow]
image: images/posts/never-trust-the-output.png
description: "Generating code got cheap. Verifying it did not. This is the part of my AI setup that actually earns its keep: risk scoring, negative controls, and reviewing the reviewer."
featured: false
comments: false
---

This is the second of three posts about how I work with AI agents. The [first one]({{ site.baseurl }}/2026/10/stop-prompting-write-rules.html) was about writing rules instead of prompts. Rules tell the agent what to do. They do not tell you whether it did it.

That gap is the whole subject here, and it is the part of my setup that actually earns its keep.

One sentence carries the rest of this post: **AI made code generation cheap and left verification exactly as expensive as it always was.** Every problem below is a consequence of that asymmetry.

## A migration that did nothing and said it worked

Some of my database tables run under PostgreSQL row-level security, with `FORCE` enabled so even the table owner is subject to the policies.

A migration needed to convert data in one of those tables. It ran. It reported success. It had matched zero rows.

The migration role owns the table but is not guaranteed `BYPASSRLS`, so the `UPDATE` was silently scoped to nothing. No error. An `UPDATE` that matches no rows is a perfectly successful `UPDATE`. The failure surfaced several statements later as a check constraint violation, and that error named the constraint, which is the wrong place to look.

Here is the part that changed how I think. I added a guard: count the rows that should have been converted and are not, and raise if the count is above zero.

The guard read zero and passed.

It was querying the same table, so it was filtered by the same policy that had hidden the rows in the first place. The check and the bug shared a blind spot. Two layers of green on a migration that did nothing.

The fix was to ask a different source entirely, `pg_class` and `pg_roles`, which carry no row policies. And to prove the whole thing by running the migration as a role without `BYPASSRLS`, against a database that actually holds the row the ordering is about. Reading the code catches none of this.

**A check that shares a blind spot with the bug is worse than no check**, because it converts "I do not know" into "it is fine."

## Not everything deserves the same scrutiny

If verification is the expensive part, then spending it evenly is the mistake. A typo in a log message and a change to permission checks do not deserve the same hour.

So I score changes. The methodology is called Verified Intent Development, and it is [public](https://github.com/AInvirion/Verified-Intent-Development). The scoring is four dimensions:

```
Risk Score = (Impact x 3) + (Reversibility x 2) + (Exposure x 2) + Compliance
```

| Dimension | Range | The question it asks |
| --- | --- | --- |
| Impact | 1-5 | What happens if this is wrong? Cosmetic, or data loss? |
| Reversibility | 1-5 | How hard is it to undo? Redeploy, or impossible? |
| Exposure | 1-5 | Who sees the effects? Me, or every customer? |
| Compliance | 0-10 | What obligations apply to *this change*, not to the regime's name? |

The score runs 7 to 45 and routes to one of four levels:

| Score | Trust level | What it buys |
| --- | --- | --- |
| 7-10 | High | Read it, check it matches intent, run the automated checks |
| 11-20 | Moderate | Plus edge cases and a basic security pass |
| 21-30 | Guarded | Plus adversarial testing, security audit, peer review |
| 31-45 | Minimal | Plus multiple reviewers, mutation testing, a written rollback plan |

Three things about this that took me a while to get right.

**The names describe the context, not the code.** "High Trust" is not permission to skip reading the output. It means the risk context allows more unverified trust, because being wrong is cheap and reversible.

**Escalation only goes one way.** If Impact, Reversibility or Exposure hits 5, or Compliance is 6 or more, it moves up a level regardless of the total. Judgment can raise a level; the formula can never argue one down. Otherwise the scoring becomes a machine for talking yourself into shipping.

**The score is a routing aid, not an assurance result.** It decides how much verification a change gets. It does not certify anything, and the person accepting the change is still accountable for it. I have seen enough dashboards treated as proof to be careful about this one.

## Prove the test can fail

Three tests of mine once passed with the bug still present. The causes were ordinary:

- a fixture that already contained the expected data
- a set comparison that quietly swallowed a duplicate
- an assertion that some column was populated, instead of which value it held

Each of those tests had a correct name, a green tick, and no power at all.

So: **revert the fix, run the test, and watch it fail. Then check that it failed for the stated reason.** A test that goes red for the wrong reason is still not evidence.

This generalises past tests, into anything that reports an absence. Once, turning off comment skipping in a new CSS reader left all nine of its tests green, which reads as "that setting does nothing." It was not inert. A comment above a rule is ignored either way, so the setting only bites *inside* a rule body, and no test was pointed there. A real change, no test aimed at it, wearing the costume of a no-op.

The hole underneath was worse than the bug being fixed. A commented-out declaration inside a rule still serialises into the rule's text, so an assertion like `"clamp(" in block(...)` is satisfied by the very edit that deleted the declaration. It fails green.

Which gave me the rule I now reach for most: when you assert that one string appears inside another, ask whether the serialised thing can carry something a person wrote that is not the thing. Comments, disabled entries, `#` config lines, `--` in SQL, docstrings. **An `in` against a rendered blob is a substring match wearing an assertion's clothes.**

## A narrow control passes forever

Negative controls are necessary and not sufficient, and this is the subtlest failure I know.

I had a scan with a genuinely good control: one link that must fire and one that must not, the second being the same phone number in two spellings. It passed every run.

The comparison was still wrong. It compared by digit suffix, so `+1 425 555 0123` "agreed" with `tel:+4414255550123`. The email comparison case-folded the entire address, so `sales@` agreed with `Sales@`.

Note the direction of both faults. They **hide** findings. They make the count smaller, which is the answer everybody is hoping for, so nothing in the process pushes back.

The control never caught either, because a control only ever disproves the cases somebody thought to put in it. A merely narrow control passes every run, reads as rigorous, and is indistinguishable from a correct one from inside the tool.

What I do now:

- Ask what the control does **not** contain, specifically in the direction that fails quietly. For any triage or scan, that direction is always whatever makes the count smaller.
- When a review finds a rule wrong, grep for every other copy of that rule before closing it. A comparison written twice is wrong twice.
- When two copies of a rule must agree, make it a check rather than a comment: ask both the same cases in the same process and fail on disagreement. Then prove the check works by moving the other copy and confirming it fires.
- Give a comparison a third answer. A boolean forces every hard case to be rounded, and rounded towards agreement is a customer nobody tells.

## The review loop, and its gate order

Now the mechanics. I use a second model as an adversarial reviewer on the diff, in a loop, and the order of the gates is fixed:

1. Review rounds on the diff until a round comes back with no blockers. Every round that finds something gets fixed and re-reviewed.
2. Only then open the pull request.
3. Merge when CI is green.

One piece of work, one pull request. The important part is what is *not* allowed: I do not get to offer "merge without another round." If a round found something, there is another round. The loop ending clean is the signal. A round count is not.

And here is the measurement that justifies the whole apparatus. Four rounds on one change, tracking not just what was found but **whose code it was in**:

| Round | What it reviewed | Found | Real | In my own patches |
| --- | --- | --- | --- | --- |
| 1 | general correctness | 5 | 3 | 0 |
| 2 | the tests, asked by name | 5 | 5 | 1 |
| 3 | the round 1 patches | 3 | 3 | **3** |
| 4 | verification | 0 | 0 | 0 |

Round 3 found three real defects and all three were in the fixes I had written in round 1.

That is not bad luck. Those patches were written under time pressure, against a reviewer's prose description rather than a reproduction, and **nothing had reviewed them**. The original code had at least one careful pass. The patch had none.

Worse, all three failed in the same direction as the original bug: a stale link reported as *agreeing*, so the fault reads as healthy.

So the rule is: **after you fix review findings, run the next round aimed at the fixes themselves**, named in the prompt. Not a repeat of the same general review. And say which direction of wrongness is dangerous, so findings get ranked by it. Something like: a wrong True hides a live customer fault, a wrong False is only a false alarm, and None is an honest admission of ignorance.

Keep that last column. One round that is entirely your own patches means run another. Two in a row means stop patching and replace the mechanism.

## Review the reviewer

The last layer, and the one I would not have predicted.

Three review rounds on one change were void, and **every output-side check I had passed on all three.**

Twice the reviewer received a bare diff with no prompt, because the heredoc that was supposed to write the prompt file sat at the end of an `&&` chain inside a backgrounded compound command, and never ran. The reviewer dutifully asked "which review should I run?" My checks saw a non-empty output file and the right reasoning-effort header and no access errors. All green. The only tell was a missing `VERDICT:` line.

The third round was handed a diff generated two commits earlier. It reported three defects that were already fixed. What gave it away was that it quoted code no longer in the tree.

**A reviewer that read nothing, and a reviewer that read the wrong thing, both look exactly like a reviewer that found nothing.**

Two habits came out of that, and they are the most portable things in this post:

**Assert the artifact a success produces. Never assert the absence of a failure's symptoms.** Checking for no errors is not checking for success. The `VERDICT:` line is what a real round produces, so that is the thing to require.

**Verify the input, not only the output.** Build the whole reviewer input to one file, then assert that the file is non-empty and contains a marker string from each fix this round is meant to see. Refuse to launch on a zero. It costs one `grep -cF` and it would have saved three rounds.

And when findings quote code you have already replaced, suspect a stale input before you suspect the reviewer.

## What to actually do tonight

If you take one thing, take the first. The rest can wait.

- **Make every claim quote its evidence.** Not "the tests pass" but the command and the counts. You cannot check the first sentence, and neither can the agent that wrote it.
- **Revert one fix and watch its test fail.** Pick the test you are most confident in. I was wrong three times.
- **Add one adversarial review round before you open a pull request**, and point the next round at the patches the first round produced.
- **Before trusting any check, ask what it does when it cannot answer.** If the answer is "reports success," you have a dashboard, not a check.

None of this needs my setup. It needs a shell and the willingness to assume your own verification is broken until it has demonstrably failed once.

## Next

Post 3 is the infrastructure: several machines running agents in parallel, one coordinating them, work reaching me on my phone, and the accounting that tells me what it all costs. That one is genuinely optional. These two are not.

Measurement is the last thing worth saying here. Of everything above, the single highest-return habit has been treating a failed check as evidence about the check. "Not found," "zero results" and "command not found" are statements about the search, not about the world. Most of my expensive mistakes have been a confident sentence built on one of those three.
