---
layout: post
permalink: /2025/04/data-resources-agentic-ai-open-source-security-compliance.html
title:  "Data Resources for Agentic AI in Open Source Security and Compliance"
author: oscar
categories: [ oss, security, compliance, supplychain, ai, n8n, sbom ]
tags: [sbom, compliance, oss, supplychain, openchain, ai, n8n]
image: images/posts/agentic_sboms_datafeed.png
description: "Learn how to use data sources like SCANOSS & VulnerableCode with Agentic AI to improve open source security audits and compliance risk management"
featured: false
comments: false
---

Following my [previous post](https://ovalenzuela.com/2025/04/ai-agents-the-missing-piece-in-sbom-compliance.html), I want to expand on the topic with some ideas for data feeds that support open source compliance audits and risk assessments when using an Agentic AI approach. These resources are available today, and I’ve had the chance to test and demonstrate them in a recent talk.

## SCANOSS / Software Transparency Foundation

The [Software Transparency Foundation](https://www.softwaretransparency.org/) offers a public [API](https://docs.osskb.org/) that connects to the Open Source Software Knowledge Base (OSSKB), which is produced by SCANOSS and licensed through the STF for public use. This system uses code fingerprints to identify code components, even at the snippet level. In addition to the main scanning API, SCANOSS offers specialized open datasets focused on regulatory and legal risks, including the [*Crypto Algorithms Open Dataset*](https://github.com/scanoss/crypto_algorithms_open_dataset) and the [*Geo-Provenance dataset*](https://www.scanoss.com/post/understanding-the-geo-provenance-dataset).

Key data provided by SCANOSS includes:

* Identified component identifiers (e.g., `PURLs`) from code fingerprint matches
* Detected licenses associated with the identified code
* Hashes of matched code snippets or files
* A catalog of known cryptographic algorithm implementations
* Jurisdictional mappings of software components by country of origin

This data enables agents to detect reused open source code, verify license attribution, trace code origins, identify components that might require export licenses due to cryptography, and flag software sourced from regions under trade restrictions or subject to data regulations.

## Software Heritage
Software Heritage is an open, non-profit initiative maintaining the world’s largest public archive of source code. Its mission of collecting and preserving all publicly available software makes it a powerful resource for compliance and cybersecurity. By offering a rich API and persistent identifiers (SWHIDs) for every piece of code, Software Heritage enables agentic systems to verify code provenance, integrity, and traceability at scale. In practice, an AI agent can leverage Software Heritage in several ways to bolster open source compliance and security:

* Archive Verification: Agents can quickly check if a given source code release or package (e.g., a tarball) is already archived in Software Heritage’s database. This enables use cases like using the archived version for source compliance instead of publishing redundant tarballs.
* Verify Modifications: Internal forks from open-source packages represent incremental technical debt due to cherry-picking changes when the software needs to be updated to a more recent upstream version, which is still often necessary (at least temporarily). Forks also introduce compliance and IP risks. Software Heritage's API and scanners allow agents to identify whether a local fork has been modified and where those modifications were introduced.
* License Files Dataset: Software Heritage has built the largest dataset of licenses from all the archived projects, a valuable resource for benchmarking compliance tooling and training AI on license generation.

Software Heritage’s archival infrastructure and identifiers ultimately empower a more transparent and secure open-source supply chain. By integrating SWH into their workflows, agent-based systems gain a dependable memory: they can detect unrecorded code, validate integrity through immutable hashes, and link to historical versions for context or legal compliance.


## AboutCode

The [AboutCode](https://aboutcode.org/) initiative encompasses several open datasets, tools, and APIs designed for in-depth analysis of software components concerning licenses, security posture, and provenance. Their key offerings include:

* [ScanCode.io](https://github.com/aboutcode-org/scancode.io/), a self-hosted service with a RESTful API for scanning codebases
* [VulnerableCode](https://public.vulnerablecode.io/), a public API that maps open source packages to known vulnerabilities
* [ScanCode LicenseDB](https://github.com/aboutcode-org/scancode-licensedb), a comprehensive license database hosted on GitHub

These tools provide valuable data, including:

* Detailed scan results from ScanCode.io in `JSON` or `SPDX` format, including detected licenses, package metadata, and file-level insights (Note: this service must be self-hosted.)
* Vulnerability mappings from VulnerableCode, linking package identifiers (e.g., `PURLs`) to `CVEs`, along with references and fixed version details
* An extensive collection of license texts, `SPDX` identifiers, and detection rules from ScanCode LicenseDB

Agents can leverage these tools to automate large-scale code scanning within CI/CD pipelines, query vulnerability data by package, and enrich license validation workflows or AI training sets with high-quality licensing metadata.

## ClearlyDefined

[ClearlyDefined](https://clearlydefined.io/) is a community-driven effort focused on aggregating and curating licensing and security metadata for open source packages. This information is available via a public [API](https://api.clearlydefined.io/definitions/), offering structured data across many popular ecosystems (`npm`, `Maven`, `PyPI`, `NuGet`, etc.).

The data includes:

* Declared and discovered software licenses (`SPDX` identifiers)
* URLs pointing to source code repositories
* Copyright holder information
* Full texts of detected licenses
* Confidence scores indicating metadata quality
* Per-file license information (accessible through specific query options)

The ClearlyDefined project is key in enriching SBOMs and compliance workflows by filling in missing or incomplete attribution data. To retrieve structured license insights, agents can query the `API` using standard package coordinates (`type/provider/namespace/name/version`).

## deps.dev (Open Source Insights by Google)

[deps.dev](https://deps.dev/) (Open Source Insights by Google) provides structured metadata and dependency information for software packages across significant ecosystems, including `npm`, `Maven`, `PyPI`, `Go`, and `Rust`. Its public [API](https://docs.deps.dev/api/v3/) returns comprehensive metadata such as:

* Complete dependency graphs, including direct and transitive relationships
* Known vulnerabilities from the OSV database, mapped to affected version ranges
* License data for each package version
* Version history with release dates
* Cryptographic hashes and links to source repositories
* Integrated security signals, such as OpenSSF Scorecard results

This service helps agents analyze how dependencies are interconnected, track the entry points of vulnerabilities, verify licensing, monitor version freshness, and assess overall risk and health using standardized metrics.

## OpenSSF Security Scorecards

[OpenSSF Security Scorecards](https://openssf.org/projects/scorecard/) is a project by the Open Source Security Foundation that automatically evaluates open source repositories against security best practices. It assigns numeric scores to a set of defined checks, and makes results available through a public [API](https://api.securityscorecards.dev/).

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
