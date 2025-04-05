---
layout: post
permalink: /2025/02/why-most-sboms-fail-and-what-to-do-about-it.html
title:  "Why Most SBOMs Fail and What to Do About It"
author: oscar
categories: [ oss, security, compliance, supplychain, openchain ]
tags: [sbom, compliance, oss, supplychain, openchain]
image: images/posts/sbom.png
description: "Why Most SBOMs Fail and What to Do About It"
featured: true
comments: false
---

SBOM adoption is accelerating. Regulatory pressure, threats to software supply chains, and transparency demands drive widespread use. But while SBOMs are becoming standard, their quality often falls short.

Open Source License Compliance (OSLC) teams have tracked software components and licenses for years before SBOMs became mainstream, often using spreadsheets. Standardized formats like SPDX and CycloneDX promised automation and clarity, but SBOM-driven processes usually fail to deliver in practice.


### The Reality: SBOMs Are Often Incomplete, Inaccurate, and Hard to Use


Despite years of standardization and significant progress around standardized structure formats, most SBOMs lack the consistency and depth needed for real-world use. They pass schema checks but fail basic usability and quality tests.

** Common SBOM issues fall into several categories:**

- Many files are incomplete, missing critical fields such as licenses, supplier names, or checksums.
- Others contain duplicate components or rely heavily on “no-assertion” entries, offering little usable information.
- License data is frequently inaccurate, a problem made worse by the overconfidence placed in automated SCA tools—often promoted by security departments that, by sheer coincidence, also happen to control the majority of the tooling budget.
- Format incompatibility is also a frequent challenge. Although SPDX and CycloneDX aim for similar outcomes, their structural differences create friction during integration or conversion.
- Compounding this, updates to SBOM standards introduce new fields and capabilities, but tools often lag in adopting them.
- Many SBOMs are generated automatically by software composition tools and assumed to be accurate without further validation, leading to widespread trust in documents that may not meet compliance or quality requirements.

### What Makes an SBOM High Quality?

The OpenChain Telco SBOM Guide v1.1 offers a practical definition of SBOM quality. It emphasizes standardization, completeness, and transparency to support software supply chain management. It outlines recommendations and requirements for including key metadata, license data, and transitive dependencies. The core goal is simple: every SBOM should be clear, consistent, and complete at the point of delivery.

### Testing SBOM Quality: How Can You Measure It?

Here are practical methods to assess the quality of SBOMs for those using or generating them. These are general recommendations, and industry-specific practices may further enhance this framework. The following list represents basic validation checks that have proven effective in various scenarios.

- **Schema Validation**: Use tools, such as JSON or XML validation tools, to verify that your SBOM adheres to the SPDX or CycloneDX schemas.
- **NTIA/CRA Compliance**: Verify that your SBOM includes the necessary fields for regulatory compliance, such as license information, supplier details, and versioning.
- **License Verification**: Employ license validation tools to compare the licenses listed in your SBOM against established license datasets and identify any inconsistencies. In experimental scenarios, I've observed that even advanced SCA tools may struggle to achieve 100% accuracy in license identification. When a license cannot be definitively determined, the SBOM often contains multiple notations, "no-assertion" entries, or empty fields.
- **Evaluating No-Assertion Rates**: I have encountered SBOMs where every field for each component is marked as "no-assertion." It is advisable to verify this information directly in the raw SBOM file, as many commercial tools attempt to infer missing data during the SBOM import, potentially contaminating the SBOM with unverifiable assumptions as the tool lacks access to the source code for confirmation.
- **Legal Risk Detection**: Scan SBOMs against curated datasets of problematic packages to identify high-risk dependencies associated with known vulnerabilities. Some open-source communities maintain projects with datasets of problematic components (hidden binaries, incorrect license assertions, undeclared deep dependencies, etc.), which can aid in risk identification. Examples include VulnerableCode from NexB, ClearlyDefined from OSI, and OSSA (Open Source Software Advisory) from Xpertians. If your project uses any cryptography library, it’s a good time to check against the Crypto Algorithm dataset offered by SCANOSS.
- **Cross-Format Compatibility**: When supporting multiple SBOM formats, conduct tests to preserve data integrity during conversions between SPDX and CycloneDX.
- **Hashing**: Verify that all components include the same set of hashes and that the length and format of these hash strings correspond to the expected values.
- **Metadata Integrity**: Vendors sometimes provide heavily manipulated SBOMs. In such cases, the tool description may be absent, or the information may exhibit inconsistencies throughout the file.

### The Industry Problem: SBOMs Evolve Faster Than Tools

The most significant barrier to SBOM reliability is the gap between evolving standards and stagnant tooling. While SPDX and CycloneDX continue to add new metadata and security features, most supporting tools—scanners, automation pipelines, and policy engines—struggle to keep up. This misalignment creates gaps in automation, format conversion, policy enforcement, and risk detection. The result is a fragmented ecosystem with inconsistent adoption and limited interoperability across the software supply chain.

### The Path Forward

SBOMs are essential, but generating a file isn’t enough. Usability and reliability require collaboration across standards bodies, toolmakers, and users. Schema validation is the floor, not the ceiling. We need clear quality benchmarks, better cross-format compatibility, and automation that flags low-quality SBOMs. Until a widely accepted quality standard emerges, teams must validate and refine the SBOMs they produce, consume, and share feedback to strengthen the ecosystem.

---

**What challenges have you faced with SBOMs? What does “quality” mean in your context? I’d love to hear how you validate and improve the SBOMs in your environment.**
