---
layout: post
permalink: /2025/08/bsa-security-scanning-tool-for-ai-models.html
title:  "BSA Security Scanning tool for AI Models"
author: oscar
categories: [ oss, security, compliance, supplychain, ai, models, sca ]
tags: [sbom, compliance, oss, supplychain, openchain, ai]
image: images/posts/BSA_Model_Scanner.png
description: "Security vulnerabilities and legal compliance gaps in AI model distribution, analyzing real-world attacks and licensing violations."
featured: false
comments: false
---

Six months ago, I wrote about the massive security blind spot in AI adoption. Organizations download ML models from the internet. They deploy them in production. They trust them completely.

The response was overwhelming. Security teams reached out, asking the same question: "How do we scan these models?"

The truth was uncomfortable. No existing tools could do it properly.

## The Problem is Real
A quick internet search reveals that over 60% of HuggingFace models lack license information. Traditional security scanners miss pickle files entirely. Supply chain attacks, such as PoisonGPT and the PyTorch compromise, have proven that a threat exists.

However, pointing out problems without solutions felt incomplete, and even a proof-of-concept tool could make a significant difference when the gap is enormous.

### A Security Scanner for AI Models
I spent the last weeks extending BinarySniffer (an existing project I use for Binary Static Analysis of Linux Packages, firmware, and Mobile Apps) with ML security capabilities (file parsing, signature matching, reporting, etc).

The tool does what I wished existed when I wrote that first post. It analyzes ML models without executing them. It understands pickle files, PyTorch models, ONNX formats, and SafeTensors. It maps threats to MITRE ATT&CK frameworks to produce "features" that later match with signatures that I can generate using an AI Agent. Most importantly, it catches real attacks.

### Testing Against Real Threats
I validated the tool against every known ML attack I could find. Results speak louder than marketing claims. 100% detection rate on malicious pickle exploits. Detects command execution patterns used in supply chain attacks. 94% confidence in detecting PyTorch model threats and 56% confidence in identifying suspicious XGBoost models. All of that can be achieved using simple string searching and signatures, which is pretty cool for a proof-of-concept approach.

The tool identified over 50 different attack patterns across various ML formats, but it's not perfect. The tool is a proof-of-concept that makes progress but requires significantly more time to address the necessary features.

## Making It Practical
The implementation is straightforward. You need to "pip it" and use it. Signatures and calibration will happen automatically as you use it:

```
pip install semantic-copycat-binarysniffer
binarysniffer ml-scan suspicious_model.pkl
```

The output integrates with GitHub Actions. It generates SARIF reports for CI/CD pipelines. Therefore, security teams can obtain actionable intelligence that they can act on immediately or execute on demand. The tool can also be integrated as a Python Library into another system. Simple as it gets.

### What This Means for Your Organization
Every organization adopting AI faces the same choice. Wait for the first ML supply chain attack to hit your systems. Or start scanning your models today, and share your experience with others so we can improve tooling and research.

<img src="images/posts/bsa_results.png">

BinarySniffer is Open Source under Apache-2.0 and available on GitHub. Feel free to submit comments, recommendations, examples, new signatures, and even feature requests.

What it catches: Basic attacks, known malware patterns, unobfuscated pickle exploits, and obvious command injection attempts.
What it misses: Sophisticated obfuscation, weight-based backdoors, novel attack patterns, and steganographic payloads. Think of it as your first line of defense, not a complete solution.

## Moving Forward
The question isn't whether ML supply chain attacks will happen. They already are. The question is whether your organization will be protected when the next one hits. Don't wait to find out; you don't need to use this tool. You can use any tool that works for your use case. However, the time to start doing something is today.

I welcome ideas to improve scanners or produce proof-of-concept tools. Many more are coming after this one.

### Reference: 
* Pepe, F., et al. (2024). "How do Hugging Face Models Document Datasets, Bias, and Licenses? An Empirical Study." 32nd IEEE/ACM International Conference on Program Comprehension (ICPC). https://mdipenta.github.io/files/icpc2024.pdf