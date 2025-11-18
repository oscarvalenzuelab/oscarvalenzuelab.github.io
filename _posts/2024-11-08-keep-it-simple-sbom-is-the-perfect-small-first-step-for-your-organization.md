---
layout: post
permalink: /2024/11/keep-it-simple-sbom-is-the-perfect-small-first-step-for-your-organization.md.html
title:  "The 'keep it simple SBoM' is the perfect small first step for your organization."
author: oscar
categories: [ oss, securty, compliance, supplychain, openchain ]
tags: [sbom, compliance, oss, supplychain, openchain]
image: images/posts/small_feet.png
description: "The 'keep it simple SBoM' is the perfect small first step for your organization."
featured: false
comments: false
---

SPDX and CycloneDX are excellent standards for handling Software Bill of Material (SBoM), but full adoption requires time, tooling, and correct intake processes. If your organization is not yet ready for this, consider using a simplified format like KissBOM (commonly known as Open Source Package Inventory or OSPI). It's a practical choice that can ease your transition into SBOM management.

For early adopters, KissBOM offers a simplified SBOM format that is not only more accessible to manage but also covers the essentials: package URLs (PURLs), license information, and optional copyright statements. Its design for ease of use makes it an excellent option for organizations needing to quickly track their software components without the overhead of more complex formats. It even allows for the manual creation of an SBOM.

While KissBOM doesn't support the complexity of managing relationships and metadata, it can help achieve two primary goals: documenting the package inventory with license information and empowering organizations with the knowledge of using these artifacts to track and share package inventory information. Once best practices for generating, parsing, and tracking components are established, organizations can transition to fully standardized formats, implementing the necessary tooling and robust processes to create and receive SBOMs in a fully standardized format.

Starting with a simplified format will alleviate Open Source Compliance activities and allow risk screening without involving heavy tooling. For example, using the package data from KissBOMs and data sources like VulnerableCode can help simplify security risk analysis. I've been using KissBOMs for testing tools that automatically produce Open Source Compliance artifacts like Legal Notices and Source Compliance to alleviate the work of generating Attribution Documents. While all of these projects are still "Works in Progress," the key is the adoption of a simplified SBoM.

Adopting a full standard will be ideal, but if we want to start experimenting and aligning organizations, concepts like KissBOM are the perfect small step to consider today.