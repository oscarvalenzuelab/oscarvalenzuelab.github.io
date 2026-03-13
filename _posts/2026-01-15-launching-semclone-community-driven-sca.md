---
layout: post
permalink: /2026/01/launching-semclone-community-driven-sca.html
title:  "Launching SEMCL.ONE: Community-Driven Software Composition Analysis"
author: oscar
categories: [ oss, security, compliance, supplychain, ai, sbom ]
tags: [sca, compliance, oss, supplychain, ai, sbom, semclone, opensource]
image: images/posts/semclone-launch.jpg
description: "Introducing SEMCL.ONE, a community-driven SCA platform that automates open source compliance using AI and shared infrastructure."
featured: true
comments: false
---

After years of building compliance automation inside large organizations, I kept running into the same problem: the tools that exist are either too expensive, too rigid, or too disconnected from how software is actually built today.

So I built something different.

## What is SEMCL.ONE?

[SEMCL.ONE](https://community.semcl.one) is a community-driven Software Composition Analysis platform. It combines open-source tools, shared infrastructure, and AI automation to make compliance management accessible and scalable.

The core idea is simple: compliance should be automated, not manual.

That means detection, analysis, validation, reporting, and continuous monitoring—all flowing through your development pipeline without constant human intervention.

## Why now?

Traditional SCA tools were built for a world where developers copied libraries from known sources. But that world is changing fast. AI-generated code doesn't always match existing packages. It transforms, adapts, and creates variations that slip past hash-based detection.

## What's inside?

The platform includes several specialized tools:

- **PURL2SRC & SRC2PURL** — Convert between package identifiers and source code across 13+ ecosystems
- **OSSLILI** — License detection supporting 700+ SPDX identifiers with multiple detection methods
- **Semantic CopyCat** — Advanced IP contamination detection targeting AI-generated code transformations
- **MCP-SEMCLONE** — IDE integration that brings conversational compliance to AI assistants like Cursor, Cline, and VS Code
- **OSPAC** — Policy engine with 712 licenses and compatibility checking

Some tools are production-ready. Others are still in development. The project is about 79% complete, with OSSLILI, OSPAC, UPMEX, and MCP-SEMCLONE already live.

## Community-driven by design

This isn't a closed product. Everything is open source. The reference databases are community-contributed. The output formats follow industry standards—SPDX, PURL, CycloneDX.

If you want to contribute, there's room for everything: bug reports, feature suggestions, code, documentation, detection patterns. The goal is to build something that works for everyone, not just those who can afford enterprise licenses.

## Why I'm doing this

SEMCL.ONE is compliance automation built on agentic AI principles, designed for the way software is actually written today.

If you're working on open source compliance, software supply chain security, or just curious about what AI-powered SCA looks like, come check it out.

[Join the community](https://community.semcl.one)

---

*The views expressed in this document are solely my own and do not represent those of my current or past employers. If you identify an error, please contact me, and I will make the necessary updates.*
