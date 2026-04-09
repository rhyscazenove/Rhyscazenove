---
layout: post
title: "Agents in the Pipeline: from a skill that works locally to a service you can trust"
date: 2026-04-09
categories: [ai, engineering]
tags: [claude-code, agents, gitlab-ci, azure, governance, security]
description: "A write-up of my talk at CCCL #5 in London — how we built a production governance harness for agentic AI workflows at the Natural History Museum."
---

Last night I gave a talk at [CCCL #5](https://cccl.dev) in London. This is a write-up of what I covered. You can also [view the slides](/presentations/cccl-april-2026.html) directly.

The short version: Claude Code skills are easy to get running locally. Making them trustworthy enough to run unattended in production is a different problem entirely — and one worth solving deliberately.

---

## The starting point

If you've been building with Claude Code, you've probably reached this point. You've written a skill that does something genuinely useful — a documentation generator, a log investigator, a code reviewer. It works well. You've iterated on it. You trust it.

Now you want to run it as a service.

That's where things get interesting. Running locally and running as a service look superficially similar but are fundamentally different in the ways that matter:

| Running locally | Running as a service |
|---|---|
| You control it. | Runs unattended. |
| Agent can explore. | No exploration. |
| Human always in the loop. | No human oversight. |
| Failures are visible. | Drift is silent. |

That last one is the important one. When an agent runs on a schedule and something goes subtly wrong, nobody sees it in the moment. By the time you notice, you might have a week of bad output, or worse.

---

## Three pillars

Getting a Claude Code agent safely into production requires three things to be in place simultaneously.

**Guardrails.** Constrain what the agent can do at runtime. Block dangerous commands, confine it to its problem space, prevent unconventional tooling. The agent should only be able to do the things you've explicitly decided it should be able to do.

**Confinement.** An isolated, reproducible environment. No side effects, no state leakage, no dependency drift between runs. Every execution should start from a known state.

**Observability.** Visibility into everything the agent does — including what it's *allowed* to do, not just what gets blocked. This is how you verify it hasn't drifted, and how you keep the guardrails current.

Running an agent in production without this is like letting a golden retriever loose at a buffet — enthusiastic, well-intentioned, and absolutely going to cause a scene.

---

## Simon Willison's Lethal Trifecta

Before going further, it's worth naming the risk model we're designing against. Simon Willison has described what he calls the Lethal Trifecta: three things that are each fine in isolation but dangerous together.

1. Access to private or sensitive data
2. Exposure to untrusted content
3. A mechanism to exfiltrate data

Any one alone is manageable. All three together is the problem — untrusted content can inject instructions that use the exfiltration mechanism to leak private data. The framework we've built at NHM protects against many vectors of this, but it is not a complete guarantee. The design principle is to break the flow across multiple agents so no single agent ever holds all three.

---

## The framework: four parts

Our implementation uses GitLab CI, a Docker container, and Claude Code hooks. It's four separable parts, each with clear ownership.

### Container Image

The foundation. You decide exactly which libraries, frameworks, and tools are available to the agent — nothing else. Built from a Dockerfile, cached in GitLab's container registry for reuse across every pipeline run. No dependency drift, no surprises. The container *is* the confinement.

### Security Hooks

Claude Code's hook system fires before and after every tool call. We use this for two things: security (PreToolUse validation that can block calls before they execute) and observability (PostToolUse metrics capture on every completed call).

A useful pattern we've found: configure your hooks to log every *rejected* call. Review these regularly to understand what the agent tried to do. Then add or remove tools from the container accordingly. The rejected log is one of the most informative signals in the whole system.

### Pipeline Config

GitLab CI orchestration with manual triggers, timeouts, artifact retention and audit compliance. The pipeline is the runtime harness — it sets the boundaries within which the container and agent operate.

### Ownership Model

This is the one that often gets skipped. Security shouldn't own the skills; domain experts shouldn't own the hooks. Each team owns what they understand: Software Engineering builds the skills, Infrastructure manages the container, Security writes the hooks, Domain Experts handle acceptance testing. No single team is a bottleneck, and no single team has to understand the whole system.

---

## 7 independent protection layers

Within this framework, we've implemented seven layers of defence, each providing independent protection:

```
         AGENT
    ─── action hooks ───
    L1 — Command blocklist
   L2 — Path traversal guard
  L3 — Network egress control
 L4 — Credential pattern block
    ── input guard ──
  L5 — Prompt injection guard
      ─ container ─
   L6 — Container isolation
        ─ audit ─
     L7 — Audit logging
```

The architecture mirrors defence in depth from traditional security: an attacker (or misbehaving agent) has to defeat each layer independently. L1–L4 run as action hooks; L5 as an input guard; L6 is the container itself; L7 is continuous logging of everything.

The layers are designed to be independent so that a failure in one doesn't cascade. L6 would contain something that escaped L1–L5. L7 would capture evidence of anything that reached L6.

---

## What we've built with it

Three use cases in production at NHM:

**Documentation Generator.** Scans git history, groups changes by theme, generates Architecture Decision Records and Mermaid architecture diagrams. Runs incrementally on merge.

**Onboarding Generator.** Creates full developer onboarding documentation from a codebase — app overview, architecture guide, getting-started guide, troubleshooting. Triggered on demand.

**Incident Analysis.** Webhook-triggered when error rates exceed a threshold. Uses Azure MCP to analyse the issue and prepare evidence — deep links to KQL queries and charts in Azure Monitor — so the engineer assigned to investigate has a running start before deciding the best course of action.

All three use the same harness. The hooks, container, and pipeline config don't change between them — only the skill itself changes.

---

## What we learned

**Governance first, features second.** Get the harness right before you scale use cases. It's much easier to add skills to a working harness than to retrofit governance onto skills that are already running.

**Modular ownership is essential.** Security shouldn't own the skills; domain experts shouldn't own the hooks. When ownership is clear, each team can iterate on their layer without stepping on others.

**Hook architecture gives you observability for free.** The same pattern that blocks also emits metrics. You don't need a separate observability pipeline — the hooks are already there.

**Start boring, get valuable.** Documentation and auditing are perfect low-risk pilots. The agent has read-only access to git history, there's no sensitive data in play, and the output is easy for humans to verify. It's a good place to build confidence in the framework before expanding scope.

**Human oversight stays in the loop.** AI analyses, humans approve and refine. The goal isn't to remove humans from the process — it's to give humans better information faster so they can make better decisions.

---

## The practical architecture

For those who want the technical specifics: we're running on Azure with GitLab CI as the orchestration layer. The container registry is GitLab's built-in registry. Hooks are bash scripts. Azure MCP is used for the Incident Analysis use case to query Application Insights.

The whole thing is platform-agnostic by design — the hooks are bash or PowerShell, the container runs anywhere, and the pipeline config is YAML. We've tested it against Azure DevOps and GitHub Actions as well as GitLab CI.

---

## The event

CCCL #5 was hosted by [Vikram Pawar](https://www.linkedin.com/in/vikrammpawar/), Claude Code Community Leader, and [Rob Hart](https://www.linkedin.com/in/roberthartuk/), founder of GitNation. The [full agenda is available here](https://cccl-ai.github.io/meetups-live/cccl-5b/agenda).

Other talks by Jan Peer, Ruslan Zavacky, Daniel Buchele, Valera Latsho, Aris Mandor and Talha Sheikh were all really fascinating. I love learning about where people are at in their AI journey. Some very impressive and pioneering demos.

I was impressed with the audience, and enjoyed meeting some of the community. They are very engaged and had some very good follow-up questions for me. It was their enthusiasm which prompted me to launch this blog!

---

## Slides

The slides from the talk are available [here](/presentations/cccl-april-2026.html).

If you're building something similar or thinking through the governance model for your own agentic workflows, I'm happy to talk through it — find me on [LinkedIn](https://linkedin.com/in/rhyscazenove).

---

*Rhys Cazenove is AI Lead at the Natural History Museum, South Kensington, working within Technology Solutions on the AI Proof of Value programme.*
