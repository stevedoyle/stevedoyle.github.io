---
title: "Commands, Skills and Agents"
date: 2025-10-27
tags: [AI]
toc: false
---

Anthropic's Claude workflows are built around three concepts: **commands**, **agents**, and **skills**. Here, I take a look at what I think they mean and how I see them fitting together.

---

### Commands — Claude’s “just do this right now” muscle

A **command** is Claude’s lowest-latency utility. It’s a *single-shot, declarative instruction* — like telling it to “fix the tone of this email” or “summarize this document for executives.” Commands don’t involve long memory, deliberation, or self-evolving objective — they’re transactional.

You can think of commands as the equivalent of shell commands but in natural language, stateless, deterministic *within human reason*, and extremely reliable when the scope is narrow.

**Examples:**

- Fix grammar in this paragraph
- Generate test cases for this function
- Translate this to Spanish

### Agents — goal-pursuers that care about time

Agents are not smarter than commands — they’re *persistent*.
They hold a goal over time. They loop, try, adjust. They decide *what to do next*. They can even call out to tools autonomously.

The key philosophical shift is this: commands do tasks → agents try to achieve outcomes.

That distinction sounds minor, but it’s the difference between a surgeon and a personal trainer. One succeeds instantly or fails. The other keeps showing up and evaluating progress.

Agents are powerful but risky — because autonomy introduces failure modes, ambiguity, and expectation gravity.
Anyone who thinks agents replace human supervision is LARPing. The real promise is elastic delegation — you control when to graduate from “do this” to “keep working until you solve this”.

### Skills — the intelligence economy inside the agent

Skills are the building blocks the agent can call on. In Anthropic's world, a skill is something Claude knows how to do — reusable, name-addressable, composable. They're *not autonomous*, but they're *opinionated micro-competencies* that higher abstractions, like agents, can orchestrate.

The mistake people make is thinking skills are mini-agents. They aren't. A skill doesn't *decide* — it *executes*. Think of it as a public method on Claude's internal class.

Skills also serve a critical architectural purpose: they help manage context and can reduce token usage. By encapsulating specific capabilities as discrete, callable units, agents can invoke exactly what they need without loading irrelevant context. [Anthropic's research on context management][1] shows that strategic context editing can reduce token consumption by up to 84% while maintaining workflow completion rates.

In the long view, "skill marketplaces" is the most important idea here.
Skills form the economic layer — where organizations will eventually differentiate and trade intelligence like software libraries today.

### The hierarchy

1. Commands are one-off tasks.
2. Skills are reusable capabilities.
3. Agents are goal-driven orchestrators with memory and autonomy.

You graduate upward only when needed.

### What's still missing

- **Skill discoverability** - If skills become the API surface, we need registries, versioning, and dependency management.
- **Agent reliability** - Current agents can drift, hallucinate scope, or get stuck. Production use requires better circuit breakers.
- **Pricing models** - How do you price an agent that might use 100 tokens or 100,000? This will shape adoption.

### So where is this going?

The big inflection won't come from agents going superhuman — it'll come from **skills becoming the new API surface for organizational intelligence.**
Once that clicks, commands become the UI, agents become the orchestration tier, and skills become the economy.

[1]: https://www.anthropic.com/news/context-management
