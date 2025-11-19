---
layout: post
permalink: /2025/11/why-traditional-sca-fails-when-you-audit-ai-generated-code.html
title:  "Why Traditional SCA Fails When You Audit AI Generated Code"
author: oscar
categories: [ ai, security, compliance ]
tags: [ai, sca, security, compliance, code-analysis, gpl, patents, scanning]
image: images/posts/sca-ai-code-audit.png
description: "Traditional SCA scanners were built for a world where code came from packages and human copy-paste. They break when AI rewrites code, hiding license and patent risks that text-based tools cannot detect."
featured: true
published: false
comments: false
---

AI coding assistants changed how developers write software. They help you move fast, but they also change the shape of your code in ways that break the old tools you depend on to keep your projects clean. Traditional SCA scanners were built for a world where code came from packages, open source projects, and human copy and paste mistakes. They were not built for code rewritten by a model.

Your risk did not change. Your code did.

The scanner only sees the text in front of it. When the text changes enough, it stops recognizing the original license or patent risk hidden inside the logic. If you rely only on classic SCA, you are missing the exact type of contamination AI produces every day.

## AI rewrites code in ways SCA cannot match

AI does not paste GPL or patented code. It rewrites it. The logic stays the same. The text does not. Classic scanners break because they rely on text, not behavior.

You see this in common transformations.

- AI renames variables and functions.
- AI changes indentation and formatting.
- AI moves blocks, splits functions, or merges them.
- AI translates code between languages.
- AI adapts the algorithm to match a framework.
- AI rewrites control flow with a new style.
- AI rebuilds the same algorithm from scratch with different syntax.

These changes hide the source of the code even when the algorithm is identical.

A simple example shows how fast this breaks text based SCA. Here is a small part of a GPL function from the Linux kernel:

```c
static void __rb_rotate_left(struct rb_node *node) {
    struct rb_node *right = node->rb_right;
    if ((node->rb_right = right->rb_left))
        rb_set_parent(right->rb_left, node);
    right->rb_left = node;
}
```

This is the same logic after an AI rewrite in Python:

```python
def rotate_left(self, n):
    r = n.right
    if r.left:
        r.left.parent = n
    n.right = r.left
    r.left = n
```

The control flow, steps, and meaning match the GPL version. Only the text changed. A human can see it. A traditional scanner cannot.

Once the text and layout shift, string matching, block hashing, winnowing, and AST rules lose their signals.

## When traditional SCA still works

Traditional SCA is not useless. It still works when you expect to match public code as-is. If your concern is package level dependencies, manifests, or files that appear in open source repositories unchanged, classic scanners do their job.

They are still useful when:

- You look for known packages.
- You expect to find the same files that appear on the internet.
- You need SBOM level metadata.
- You track what your build system imports.

They break when the code is produced by a model that rewrites the shape while keeping the original logic.

## Why AI generated code is easy to get wrong

Developers trust AI because the output looks clean. The code has no GPL headers, no obvious license clues, and no package references. Many teams believe the code is original. It is not original if the model reproduces a known algorithm.

The problem is not limited to GPL or AGPL. Many AI assistants rebuild parts of H.264 or H.265. They generate FFT and DCT routines that match patented code. They recreate well known algorithms from FFmpeg and other codec stacks. They reproduce database routines from AGPL projects. They rebuild common data structures from GPL kernels.

The risk is the same even when the output looks new.

You ship the code. Your SCA report says everything is clean. The logic says something else.

## Why scanners fail on real AI output

Text based scanners do one thing. They check if today's code looks like yesterday's code. They depend on shared text. When AI models rewrite the text, there is nothing left to match.

- They cannot compare across languages.
- They cannot detect algorithm structure.
- They cannot compare control flow.
- They cannot follow data flow when steps move around.
- They do not detect semantic similarity.
- They do not detect code rebuilt from the same logic.
- They cannot catch a rewritten patent encumbered routine.

This gap grows every year because models keep improving their rewrite ability.

## Circular validation hides the real failure

The most misleading part of SCA accuracy today comes from circular validation. Many SCA tools test themselves using public datasets. That worked in the past. It does not work now because AI models were trained on the same public code.

This creates a loop.

- AI assistants are trained on public code
- SCA tools compare code against public code
- Both systems share the same data
- Tests show high accuracy
- Real world detection collapses when the code is new or private

This same pattern appeared in the evaluation of CopycatM. Matching performance on private code dropped eight points compared to public datasets because public tests were inflated. Public code appears in model training sets, so it is easy to match. Real code written after the training cutoff does not match the public reference. Tests on that code matter more than anything else.

This same problem hits every traditional scanner that depends on public corpora.

## A better approach checks the behavior, not the text

To audit AI generated code, you need something beyond text. You need to check what the code does.

The question is not "Does this file look like a file from the internet".
The question is "Does this function behave like a known GPL or patented algorithm".

If the behavior matches, the risk is present even when the text does not.

This requires a mix of techniques.

- Check the control flow.
- Check the data flow.
- Check the structure of the algorithm.
- Check the operations sequence.
- Check semantic fingerprints that survive renaming and rewrites.
- Check both small and large snippets.
- Check the full file graph when functions interact.

This is the reason transformation resistant approaches like the three tier model in CopycatM were built. The scanner must survive the same transformations the AI applies. Text alone cannot survive these changes.

## What about models trained only on permissive content

Some vendors try to reduce license and patent risk by training models on permissive corpora. Amazon's Nova is one example. When a model is trained only on permissive codebases, the chance of reproducing GPL code or other restrictive licenses goes down.

This helps, but it does not remove the need for deeper analysis.
- Developers still mix model output with their own code.
- Applications still embed legacy functions.
- Teams still pull in code from unknown sources.
- Models still reconstruct algorithms from common knowledge.
- Patent encumbered logic is not avoided by license filtering.

Training only on permissive corpora reduces one class of risk but does not replace the need for code level review.

## Why this matters for your daily work

Your SCA tools tell you a story based on text. AI generated code lives in a different world. The logic you ship might match GPL or patented algorithms even when the text looks new.

If your scanner cannot detect logic level matches, you are trusting a signal that no longer reflects the actual risk. AI changed the shape of the code. You need tools that follow the logic, not the characters.

A code audit today must handle rewritten functions, cross language output, and semantic matches that survive heavy rewrites. This is the only way to catch the real contamination patterns that AI produces.

*The views expressed in this document are solely my own and do not represent those of my current or past employers. If you identify an error, please contact me, and I will make the necessary updates.*
