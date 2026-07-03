---
title: "Blog 2"
date: 2024-01-01
weight: 2
chapter: false
pre: " <b> 3.2. </b> "
---
# Supercharge your cloud operations with the Kiro power for AWS DevOps Agent

When an alarm fires at 2 AM, the first thing most engineers do is grep logs, check recent deployments, and trace code paths. However, the context they need — metrics, traces, topology, configurations — lives in separate browser tabs and applications. The **Kiro power for AWS DevOps Agent** removes that context switching by connecting your IDE directly to the AWS DevOps Agent, allowing you to investigate incidents, identify root causes, and generate fixes, all from the same place you write code.

---

## Challenges in Cloud Operations Today

Operating modern cloud applications means navigating a maze of interconnected services. Operators face persistent challenges:
- **Context switching**: Investigating an incident requires jumping between the IDE, the AWS Management Console, log viewers, trace explorers, and documentation.
- **Siloed knowledge**: Understanding which metrics matter and what "normal" looks like often lives in outdated runbooks or in the heads of senior engineers.
- **Remediation gap**: Translating findings into a working fix requires switching contexts again and manually applying changes.

These challenges compound when teams operate across multiple AWS accounts and environments. Kiro powers address these by bringing operational intelligence directly into the IDE.

---

## Challenges in Modern Software Delivery

AI coding agents have accelerated code writing, but code review and pipeline processes haven't kept up:
- **Review capacity**: AI-assisted development produces changes faster than human reviewers can evaluate them, risking standard deviations.
- **Invisible dependencies**: Applications span multiple repositories. A parameter rename in one repository silently breaks downstream consumers.

---

## What are Kiro powers?

A Kiro power is a curated package that gives Kiro specialized capabilities in a specific domain. Each power typically includes:
- **MCP server configuration**: Connects Kiro to external tools and data through the Model Context Protocol.
- **Steering files**: Domain-specific instructions that teach Kiro how to route intents.
- **Contextual knowledge**: Domain-specific guidance captured in markdown files and lifecycle hooks.

---

## The Kiro power for AWS DevOps Agent

![Kiro Power Architecture](/images/2-Proposal/platform_architecture.jpeg)

The Kiro power for AWS DevOps Agent packages the full capabilities of AWS DevOps Agent into a single install. Features include:
- **Investigate incidents**: Describe symptoms in natural language, and Kiro orchestrates a deep investigation.
- **Optimize costs**: Ask for cost savings and receive specific, data-backed recommendations.
- **Review architecture**: Request a topology map or security audit with actionable improvement suggestions.
- **Chat across agent spaces**: Operate across multiple AWS DevOps Agent spaces from a single Kiro session.
- **Generate remediation code**: Kiro can generate the fix directly in your workspace.
- **Run a release readiness review**: Have the DevOps Agent review changes for dependency risks and standard deviations before you push code.
- **Perform exploratory release testing**: Kiro can run exploratory tests on deployed applications.

---

## Walkthrough: Investigating a production incident

1. **Describe the problem**: Tell Kiro about the issue (e.g., "My ECS service is throwing 503 errors"). Kiro automatically includes relevant workspace context.
2. **Kiro starts the investigation**: Kiro queries CloudWatch, X-Ray, and ECS events, analyzing metrics against task counts.
3. **Review findings**: The DevOps Agent returns a detailed root cause analysis and recommends mitigations.
4. **Generate and apply the fix**: Kiro implements the recommendation directly into your configuration files (e.g., updating database connection pool sizes) ready for review and commit.

---

*Original article:* [Supercharge your cloud operations with the Kiro power for AWS DevOps Agent](https://aws.amazon.com/blogs/devops/supercharge-your-cloud-operations-with-the-kiro-power-for-aws-devops-agent/)
