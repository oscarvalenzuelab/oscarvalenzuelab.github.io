---
layout: post
permalink: /2025/11/why-traditional-sca-fails-when-you-audit-ai-generated-code.html
title: "Why Traditional SCA Fails When You Audit AI Generated Code"
author: oscar
categories: [ ai, security, compliance ]
tags: [ai, sca, security, compliance, code-analysis, gpl, patents, scanning]
image: images/posts/sca-ai-code-audit.png
description: "Traditional SCA scanners were built for a world where code came from packages and human copy-paste. They break when AI rewrites code, hiding license and patent risks that text-based tools cannot detect."
featured: true
published: false
comments: false
---

AI coding assistants changed how developers write software. They help you move faster, but they also reshape the code in ways that break the tools you rely on to keep projects clean. Traditional SCA scanners were built for code that came from packages, open source repositories, and human copy and paste. They were not built for code rewritten by a model.

Your risk did not change. Your code did.

SCA tools only see the text in front of them. When that text changes enough, the scanner stops recognizing the license or patent risk inside the logic. If you depend only on classic SCA, you could miss the exact type of contamination AI produces.

## AI rewrites code in ways SCA cannot match

AI does not paste code from GPL kernels or patented codec libraries. It rewrites it. The logic stays the same while the text changes. This breaks scanners that only compare strings or syntax.

AI applies many transformations at once. It renames variables and functions, changes formatting, restructures loops, or builds helper functions that did not exist before. It often switches languages. The same algorithm can move from C to Python or from JavaScript to Go. The meaning survives the rewrite. The text does not.

Here is a small example from the Linux kernel. The original GPL code:

```c
static void __rb_rotate_left(struct rb_node *node) {
    struct rb_node *right = node->rb_right;
    if ((node->rb_right = right->rb_left))
        rb_set_parent(right->rb_left, node);
    right->rb_left = node;
}
```

This is what a model may produce:

```python
def rotate_left(self, n):
    r = n.right
    if r.left:
        r.left.parent = n
    n.right = r.left
    r.left = n
```

The behavior is the same. The structure is the same. Only the shape of the text changed. A human sees the match. A text based scanner does not. Once the text shifts, string matching, block hashing, winnowing, and language specific AST rules all lose their signals.

Here is another example showing how AI changes both language and structure. The original C code from a GPL library:

```c
// GPL licensed quicksort partition
int partition(int arr[], int low, int high) {
    int pivot = arr[high];
    int i = (low - 1);
    for (int j = low; j <= high - 1; j++) {
        if (arr[j] < pivot) {
            i++;
            int temp = arr[i];
            arr[i] = arr[j];
            arr[j] = temp;
        }
    }
    int temp = arr[i + 1];
    arr[i + 1] = arr[high];
    arr[high] = temp;
    return (i + 1);
}
```

AI rewrites this to JavaScript using modern idioms:

```javascript
function partitionArray(data, start, end) {
    const pivotValue = data[end];
    let partitionIndex = start;

    for (let current = start; current < end; current++) {
        if (data[current] < pivotValue) {
            [data[partitionIndex], data[current]] =
                [data[current], data[partitionIndex]];
            partitionIndex++;
        }
    }

    [data[partitionIndex], data[end]] =
        [data[end], data[partitionIndex]];
    return partitionIndex;
}
```

The algorithm is identical. The control flow matches step by step. The variable names changed. The loop syntax changed. The swap logic moved from explicit temp variables to destructuring. A traditional scanner sees no connection between these two functions.

The problem extends beyond simple utilities to specialized algorithms. Consider this FFT butterfly operation from a signal processing library with patent concerns:

```c
// Butterfly computation (patent encumbered in some jurisdictions)
void fft_butterfly(complex *x, complex *y, complex w) {
    complex temp = mult_complex(*y, w);
    y->real = x->real - temp.real;
    y->imag = x->imag - temp.imag;
    x->real = x->real + temp.real;
    x->imag = x->imag + temp.imag;
}
```

AI generates this in Python for a streaming application:

```python
def apply_butterfly_transform(sample_a, sample_b, twiddle):
    """Apply FFT butterfly operation to two samples"""
    weighted = complex(
        sample_b.real * twiddle.real - sample_b.imag * twiddle.imag,
        sample_b.real * twiddle.imag + sample_b.imag * twiddle.real
    )
    return (
        complex(sample_a.real + weighted.real, sample_a.imag + weighted.imag),
        complex(sample_a.real - weighted.real, sample_a.imag - weighted.imag)
    )
```

The mathematical operations are the same. The patent risk is the same. The text based signature is completely different. Traditional SCA cannot connect these implementations because the language changed, the function signature changed, the variable names changed, and the structure moved from mutation to return values.

## Why scanners fail on AI output

AI generated code looks clean. There are no headers, no license markers, and no obvious links to upstream projects. This leads many teams to assume it is original. Yet the model may rebuild known algorithms from GPL, AGPL, or patented sources. It may recreate parts of H.264 or H.265. It may generate DCT, FFT, or motion compensation routines. It may rebuild functions from FFmpeg. It may reproduce data structure logic from old GPL kernels.

Traditional scanners compare today's code with older public samples. They only detect shared text. When a model rewrites that text, the scanner no longer sees the source behind it. The tool cannot follow the algorithm across languages or follow control flow when the order of operations moves. It cannot detect when two functions share the same logic but not the same structure. It cannot detect semantic matches or code rebuilt from common algorithm patterns. It cannot identify a patent encumbered routine that was reconstructed in a different style.

You ship the code. Your SCA report says there is no problem. The logic inside the file tells a different story.

This gap grows as AI models get better at rewriting code.

## Circular validation hides the real failure

Accuracy claims for traditional SCA often rely on public test sets. That worked before AI. It does not work today because models were trained on the same public code that scanners use as their reference.

This creates a loop.

* AI assistants learn public code.
* SCA tools compare against public code.
* Both systems anchor to the same corpus.
* Tests show strong results.
* Detection collapses when the code is private, new, or rewritten.

This same pattern appeared in the evaluation of CopycatM (SEMCL.ONE tool that detects contamination risk on source code generated by AI). When tested on public datasets, matching looked strong. When tested on private code written after the training cutoff, performance dropped eight points. Tests based only on public corpora are inflated because they overlap with the training data of the models you are trying to audit.

Any scanner that depends on public text suffers from the same problem.

## A better approach checks the behavior, not the text

Auditing AI generated code requires a different view. You need to check what the code does, not only how it looks. The key question is simple. Does this function behave like a known GPL or patented routine. If the behavior matches, the risk is present even when the text is new.

This requires analysis of control flow, data flow, algorithm structure, and operation sequences. It requires semantic fingerprints that survive rewrites. It requires checks that work on small and large snippets. It requires understanding how functions interact across a file. These signals survive the transformations that models apply.

Approaches that survive these rewrites are built on this idea. They follow the logic through the rewrite instead of trusting the surface text.

## About permissive corpora and why they are not enough

Some vendors try to lower risk by training models only on permissive codebases. A few commercial offerings claim to follow this approach. If the training data contains only permissive content, the chance of reproducing GPL code goes down. This addresses one part of the problem, but it does not remove the need to review the code.

Developers still combine model output with legacy code. Applications still include old functions that do not appear in public samples. Teams still add code from unknown sources without checking it. Models still rebuild common algorithms that appear in many forms across many projects. Patent encumbered logic is not filtered by license type because patents do not care about source licenses.

The risk may be lower when training data is filtered, but there is no way to prove the absence of all non permissive material. There is also no public dataset that shows the exact difference between permissive only training and mixed training in real world cases. To measure this properly, you would need a large set of model outputs, ground truth provenance, and verified training exclusions. None of this exists today. Because of that, the strongest claim you can make is that the risk is reduced in theory, not eliminated in practice.

## When traditional SCA still works

Traditional SCA still has value. It works when the code you check matches public code as-is. If you want to find package level dependencies, validate manifests, or track imports, classic tools do their job. They also work when you expect developers to copy code without rewriting it.

They break when models generate the code, because models rewrite everything while keeping the original algorithm.

## Why this matters for your daily work

Your SCA tools still tell you a story based on text. AI generated code does not follow that story. The logic may match GPL or patented algorithms even when the text looks new. If your scanner cannot detect logic level matches, you are trusting a signal that no longer matches the code you ship.

A modern code audit must track rewritten functions, code produced across languages, and semantic matches that survive heavy rewriting. This is the only way to catch the real contamination patterns that AI produces.

## What you can do next

Start by reviewing how your team uses AI. Ask where AI generated functions enter your codebase. Check how your build system mixes old and new code. Identify the critical routines that influence licensing and patents. Add semantic checks to your audit path. Review algorithms that matter in codecs, crypto, compression, and data structures. Monitor the patterns that survive rewrites, not the text that changes.

This gives you a safer view of your code and a more accurate picture of the real risk.

*The views expressed in this document are solely my own and do not represent those of my current or past employers. If you identify an error, please contact me, and I will make the necessary updates.*
