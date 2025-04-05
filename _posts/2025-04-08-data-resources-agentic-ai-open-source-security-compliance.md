---
layout: post
permalink: /2025/04/data-resources-agentic-ai-open-source-security-compliance.html
title:  "Data Resources for Agentic AI in Open Source Security and Compliance"
author: oscar
categories: [ oss, security, compliance, supplychain, ai, n8n, sbom ]
tags: [sbom, compliance, oss, supplychain, openchain, ai, n8n]
image: images/posts/agenticai_sboms.png
description: "Learn how to use data sources like ScanOSS & NexB with Agentic AI to improve open source security audits and compliance risk management"
featured: true
comments: false
---

Following my previous post, I want to expand on the topic with some ideas for data feeds that support open source compliance audits and risk assessments when using an Agentic AI approach. These resources are available today, and I’ve had the chance to test and demonstrate them in a recent talk.

## SCANOSS / Software Transparency Foundation

The Software Transparency Foundation offers a public API that connects to the Open Source Software Knowledge Base (OSSKB), which is produced by SCANOSS and licensed through the STF for public use. This system uses code fingerprints to identify code components, even at the snippet level. In addition to the main scanning API, SCANOSS offers specialized open datasets focused on regulatory and legal risks, including the *Crypto Algorithms Open Dataset* and the *Geo-Provenance dataset*.

Key data provided by SCANOSS includes:

* Identified component identifiers (e.g., `PURLs`) from code fingerprint matches
* Detected licenses associated with the identified code
* Hashes of matched code snippets or files
* A catalog of known cryptographic algorithm implementations
* Jurisdictional mappings of software components by country of origin

This data enables agents to detect reused open source code, verify license attribution, trace code origins, identify components that might require export licenses due to cryptography, and flag software sourced from regions under trade restrictions or subject to data regulations.

## NexB / AboutCode

The NexB / AboutCode initiative encompasses several open datasets, tools, and APIs designed for in-depth analysis of software components concerning licenses, security posture, and provenance. Key offerings include:

* ScanCode.io, a self-hosted service with a RESTful API for scanning codebases
* VulnerableCode, a public API that maps open source packages to known vulnerabilities
* [ScanCode LicenseDB](https://github.com/aboutcode-org/scancode-licensedb), a comprehensive license database hosted on GitHub

These tools provide valuable data, including:

* Detailed scan results from ScanCode.io in `JSON` or `SPDX` format, including detected licenses, package metadata, and file-level insights (Note: this service must be self-hosted.)
* Vulnerability mappings from VulnerableCode, linking package identifiers (e.g., `PURLs`) to `CVEs`, along with references and fixed version details
* An extensive collection of license texts, `SPDX` identifiers, and detection rules from ScanCode LicenseDB

Agents can leverage these tools to automate large-scale code scanning within CI/CD pipelines, query vulnerability data by package, and enrich license validation workflows or AI training sets with high-quality licensing metadata.

## ClearlyDefined

ClearlyDefined is a community-driven effort focused on aggregating and curating licensing and security metadata for open source packages. This information is available via a public REST `API`, offering structured data across many popular ecosystems (`npm`, `Maven`, `PyPI`, `NuGet`, etc.).

The data includes:

* Declared and discovered software licenses (`SPDX` identifiers)
* URLs pointing to source code repositories
* Copyright holder information
* Full texts of detected licenses
* Confidence scores indicating metadata quality
* Per-file license information (accessible through specific query options)

The ClearlyDefined project is key in enriching SBOMs and compliance workflows by filling in missing or incomplete attribution data. To retrieve structured license insights, agents can query the `API` using standard package coordinates (`type/provider/namespace/name/version`).

## deps.dev (Open Source Insights by Google)

deps.dev (Open Source Insights by Google) provides structured metadata and dependency information for software packages across significant ecosystems, including `npm`, `Maven`, `PyPI`, `Go`, and `Rust`. Its public `API` returns comprehensive metadata such as:

* Complete dependency graphs, including direct and transitive relationships
* Known vulnerabilities from the OSV database, mapped to affected version ranges
* License data for each package version
* Version history with release dates
* Cryptographic hashes and links to source repositories
* Integrated security signals, such as OpenSSF Scorecard results

This service helps agents analyze how dependencies are interconnected, track the entry points of vulnerabilities, verify licensing, monitor version freshness, and assess overall risk and health using standardized metrics.

## OpenSSF Security Scorecards

OpenSSF Security Scorecards is a project by the Open Source Security Foundation that automatically evaluates open source repositories against security best practices. It assigns numeric scores to a set of defined checks, and makes results available through a public `API`.

The system evaluates aspects such as:

* Use of code review and branch protection
* Dependency pinning practices
* Use of fuzzing and static analysis tools
* CI/CD integration
* Overall maintenance activity

Each check is scored from 0 to 10, and includes detailed explanations of what was detected and how the score was derived.

Scorecards offer a standardized, automated view of a project’s security maturity. This data can complement vulnerability and license scanning by flagging projects with weak security practices. Agentic systems can use this to prioritize high-risk components, identify gaps in development hygiene, and include security posture in broader risk assessments across the software supply chain.


## Final Takeaways

These data sources are available and mature enough to support agent-based compliance tooling today. You can automate large parts of OSS license validation, vulnerability triage, and legal risk assessment by connecting them into your pipelines or AI systems.

If you're exploring how to build or extend agentic systems for open source compliance, now is a great time to start integrating these feeds. The infrastructure is ready — it’s just connecting the dots.
