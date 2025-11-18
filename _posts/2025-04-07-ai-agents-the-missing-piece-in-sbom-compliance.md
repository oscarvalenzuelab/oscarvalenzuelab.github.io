---
layout: post
permalink: /2025/04/ai-agents-the-missing-piece-in-sbom-compliance.html
title:  "AI Agents: The Missing Piece in SBOM Compliance"
author: oscar
categories: [ oss, security, compliance, supplychain, ai, n8n, sbom ]
tags: [sbom, compliance, oss, supplychain, openchain, ai, n8n]
image: images/posts/agenticai_sboms.png
description: "Taming the SBOM Chaos: Using AI Agents to Audit SBOMs for OSS Compliance"
featured: true
comments: false
---

Today, I gave a talk called “Taming the SBOM Chaos: Using AI Agents to Audit SBOMs for OSS Compliance.” The slides and materials are on my [GitHub account](https://github.com/oscarvalenzuelab/sbom_analysis_using_agentic).

As Software Bill of Materials (SBOM) adoption grows, so does the complexity of managing, validating, and ensuring compliance with evolving regulatory frameworks. Many companies struggle with these challenges due to a lack of specialized in-house Subject Matter Experts (SMEs). My talk explored how AI-powered workflows, leveraging specialized models and OpenData APIs, can streamline compliance and audits. Attendees gained insights into how AI agents can assist in automating SBOM analysis, reducing human workload, and enhancing compliance strategies in an increasingly regulated software landscape.

Here’s a quick write-up of the ideas behind the talk:

## The problem
Open source license compliance is complicated. A license might seem simple, but how you use the software changes everything. Using a library internally might involve minimal obligations, but shipping it within a product can trigger significant compliance requirements under the same license.

So the same code, under the same license, can mean different obligations depending on its use. That’s why compliance isn’t just about reading the license or running a scanner to generate a report. You need to understand the whole picture—how the software is built, how it runs, and who it’s for. That makes scaling compliance across large projects very difficult. Every case is different, and making the right call takes time, experience, and judgment.

## Tools don’t solve the problem.
We have tools that try to help but don’t go far enough. Most tools assume the input is perfect. They expect a clean SBOM with all the correct and complete data. They presume every package has a valid license string, every file has the correct hash, and every rule is black and white.

But in real life, SBOMs are often broken or missing key information. Many tools struggle with this reality. Some crash, others give you results that look right but aren’t. Either way, they often don't explain their reasoning. Even if you identify an error in the source data, correcting it within the tool isn't always possible.

Even commercial tools that promise complete automation (Automated SCA audits) fall short. They might work for simple cases (like security vulnerability scans), but they struggle with edge cases or complex builds. And they often lock you into their thinking, acting as black boxes that you can’t easily question or improve their results.

## SBOMs should help. But they often don’t.
A good SBOM should tell you what’s inside your software: package names, versions, licenses, and hashes. But most SBOMs aren’t that helpful.

Some use different formats. Others are missing key fields like license data or component hashes. Sometimes, the fields are there, but they’re wrong or manually edited. There’s no standard for what “good” looks like, and the quality varies significantly depending on how the SBOM was created.

Many SBOMs create more work instead of providing answers, forcing teams to spend valuable time correcting inaccuracies rather than using the data for analysis.

## How Agentic AI helps
This is where agentic AI comes in. It’s not just a script or a one-time tool. It’s a system that can think, plan, and take action to solve a goal.

If you provide an SBOM to an AI agent, it will read the data, even if the SBOM is incomplete or incorrectly formatted. It can figure out what’s missing and try to fill in the blanks. It can check other sources, like public license databases or internal records. It can pull together what it finds and generate a clear summary for assessment.

In short, it doesn’t stop at parsing the file. It tries to answer the bigger question: “What are the risks, and what should we do about them?”

For example, imagine you get an SBOM with no license info and bad hashes. A regular tool would either fail or give you incomplete results. An AI agent would keep going. It could search for license data using the package identifiers. It might cross-check with external databases. It could flag what it couldn’t verify and give you a report showing what it found and what’s still unknown.

That’s a big step forward.

## The Key Ingredient: Expert-Curated Data
Training an AI agent without data is like hiring an intern and asking them to handle compliance on day one—with no examples, training, or documentation. They won't know where to start. Forced to provide answers without adequate guidance, you can expect anything from half-baked responses to outright hallucinations.

Like any Open Source Compliance Engineer, AI agents need foundational knowledge and context to perform effectively. If we expect agents to make sense of licenses, assess risks, and produce useful reports, we must give them expert-curated information. That includes examples of license metadata, known issues with specific packages, rules for when obligations apply, and context about how software is used.

These aren’t just “nice to have.” They’re the foundation. Without them, the agent can’t reason. It can’t tell if a component is risky. It can’t spot patterns. It can’t improve to help you better.

So instead of building tools overloaded with underdeveloped features (suffering from 'Swiss Army knife' syndrome), we need to focus on building and maintaining strong data feeds of knowledge. These should come from people who know open source compliance, like compliance engineers, lawyers, and auditors, who have done the work and been in the trenches.

The better the data, the better the agent. That’s what makes the difference. Pioneering organizations like [SCANOSS](https://www.scanoss.com/), [NexB](https://nexb.com/), and [Software Heritage](https://www.softwareheritage.org/) understand this. They lead the way by offering the curated knowledge and data feeds crucial for effective compliance. This foundational work will fuel the next wave of automation driven by AI.

Other organizations like OpenSSF, ClearlyDefined, and Deps.dev offer data sources to address security and compliance information.

## Final thought
AI won’t replace compliance engineers but will change how we work. Instead of spending hours looking for missing package metadata, checking GitHub for licenses, reading reports, or fixing bad SBOMs, we’ll design systems that do most of the heavy lifting for us. We’ll focus on the decisions that matter, and we’ll train agents to handle the rest.

We’re already seeing that shift, and it's worth building on. AI agents represent the next wave in compliance management. Now is the time to start riding it.
