---
sidebar_position: 3
title: 'Learning Path'
description: 'Choose your learning path through the Hermes Agent documentation based on your experience level and goals.'
---

# Learning Path

Hermes Agent can do a lot — CLI assistant, Telegram/Discord bot, task automation, RL training, and more. This page helps you figure out where to start and what to read based on your experience level and what you're trying to accomplish.

:::tip Start Here
If you haven't installed Hermes Agent yet, begin with the [Installation guide](/docs/getting-started/installation) and then run through the [Quickstart](/docs/getting-started/quickstart). Everything below assumes you have a working installation.
:::

## How to Use This Page

- **Know your level?** Jump to the [experience-level table](#by-experience-level) and follow the reading order for your tier.
- **Have a specific goal?** Skip to [By Use Case](#by-use-case) and find the scenario that matches.
- **Just browsing?** Check the [Key Features](#key-features-at-a-glance) table for a quick overview of everything Hermes Agent can do.

## By Experience Level

| Level | Goal | Recommended Reading | Time Estimate |
|---|---|---|---|
| **Beginner** | Get up and running, have basic conversations, use built-in tools | [Installation](/docs/getting-started/installation) → [Quickstart](/docs/getting-started/quickstart) → [CLI Reference](/docs/cli-reference) → [Configuration](/docs/configuration) | ~1 hour |
| **Intermediate** | Set up messaging bots, use advanced features like memory, cron jobs, and skills | [Sessions](/docs/sessions) → [Messaging](/docs/messaging) → [Tools](/docs/tools) → [Skills](/docs/skills) → [Memory](/docs/memory) → [Cron](/docs/cron) | ~2–3 hours |
| **Advanced** | Build custom tools, create skills, train models with RL, contribute, run nested orchestrator delegation | [Architecture](/docs/architecture) → [Adding Tools](/docs/developer-guide/adding-tools) → [Creating Skills](/docs/developer-guide/creating-skills) → [Delegation](/docs/delegation) → [RL Training](/docs/rl-training) → [Contributing](/docs/contributing) | ~4–6 hours |

:::tip Advanced track — orchestrator delegation
Once you're comfortable with single-agent flows, the v0.11.0 `delegate_task` tool supports an `orchestrator` role that can spawn its own workers, with `delegation.max_spawn_depth` (default `1`, range `1`–`3`). See [Delegation Patterns](/docs/guides/delegation-patterns) for the YAML and recommended depths.
:::

## By Use Case

Pick the scenario that matches what you want to do. Each one links you to the relevant docs in the order you should read them.

### "I want a CLI coding assistant"

Use Hermes Agent as an interactive terminal assistant for writing, reviewing, and running code.

1. [Installation](/docs/getting-started/installation)
2. [Quickstart](/docs/getting-started/quickstart)
3. [CLI Reference](/docs/cli-reference)
4. [Code Execution](/docs/code-execution)
5. [Context References](/docs/context-references)
6. [Tips & Tricks](/docs/guides/tips)

:::tip
Pass files directly into your conversation with context references. Hermes Agent can read, edit, and run code in your projects.
:::

### "I want a Telegram/Discord bot"

Deploy Hermes Agent as a bot on your favorite messaging platform.

1. [Installation](/docs/getting-started/installation)
2. [Configuration](/docs/configuration)
3. [Messaging Overview](/docs/messaging)
4. [Telegram Setup](/docs/messaging/telegram)
5. [Discord Setup](/docs/messaging/discord)
6. [Voice Mode](/docs/voice-mode)
7. [Use Voice Mode with Hermes](/docs/guides/use-voice-mode-with-hermes)
8. [Security](/docs/security)

### "I want to automate tasks"

Schedule recurring tasks, run batch jobs, or chain agent actions together.

1. [Quickstart](/docs/getting-started/quickstart)
2. [Cron Scheduling](/docs/cron)
3. [Batch Processing](/docs/batch-processing)
4. [Delegation](/docs/delegation)
5. [Hooks](/docs/hooks)

:::tip
Cron jobs let Hermes Agent run tasks on a schedule — daily summaries, periodic checks, automated reports — without you being present.
:::

### "I want to build custom tools/skills"

Extend Hermes Agent with your own tools and reusable skill packages.

1. [Tools Overview](/docs/tools)
2. [Skills Overview](/docs/skills)
3. [MCP (Model Context Protocol)](/docs/mcp)
4. [Architecture](/docs/architecture)
5. [Adding Tools](/docs/developer-guide/adding-tools)
6. [Creating Skills](/docs/developer-guide/creating-skills)

:::tip
Tools are individual functions the agent can call. Skills are bundles of tools, prompts, and configuration packaged together. Start with tools, graduate to skills.
:::

### "I want to train models"

Use reinforcement learning to fine-tune model behavior with Hermes Agent's built-in RL training pipeline.

1. [Quickstart](/docs/getting-started/quickstart)
2. [Configuration](/docs/configuration)
3. [RL Training](/docs/rl-training)
4. [Provider Routing](/docs/provider-routing)
5. [Architecture](/docs/architecture)

:::tip
RL training works best when you already understand the basics of how Hermes Agent handles conversations and tool calls. Run through the Beginner path first if you're new.
:::

### "I want to use it as a Python library"

Integrate Hermes Agent into your own Python applications programmatically.

1. [Installation](/docs/getting-started/installation)
2. [Quickstart](/docs/getting-started/quickstart)
3. [Python Library Guide](/docs/guides/python-library)
4. [Architecture](/docs/architecture)
5. [Tools](/docs/tools)
6. [Sessions](/docs/sessions)

## Key Features at a Glance

Not sure what's available? Here's a quick directory of major features:

| Feature | What It Does | Link |
|---|---|---|
| **Tools** | Built-in tools the agent can call (file I/O, search, shell, etc.) | [Tools](/docs/tools) |
| **Skills** | Installable plugin packages that add new capabilities | [Skills](/docs/skills) |
| **Memory** | Persistent memory across sessions | [Memory](/docs/memory) |
| **Context References** | Feed files and directories into conversations | [Context References](/docs/context-references) |
| **MCP** | Connect to external tool servers via Model Context Protocol | [MCP](/docs/mcp) |
| **Cron** | Schedule recurring agent tasks | [Cron](/docs/cron) |
| **Delegation** | Spawn sub-agents for parallel work (with optional orchestrator role) | [Delegation](/docs/delegation) |
| **Code Execution** | Run Python scripts that call Hermes tools programmatically | [Code Execution](/docs/code-execution) |
| **Browser** | Web browsing and scraping | [Browser](/docs/browser) |
| **Hooks** | Event-driven callbacks and middleware | [Hooks](/docs/hooks) |
| **Batch Processing** | Process multiple inputs in bulk | [Batch Processing](/docs/batch-processing) |
| **RL Training** | Fine-tune models with reinforcement learning | [RL Training](/docs/rl-training) |
| **Provider Routing** | Route requests across multiple LLM providers | [Provider Routing](/docs/provider-routing) |

## What to Read Next

Based on where you are right now:

- **Just finished installing?** → Head to the [Quickstart](/docs/getting-started/quickstart) to run your first conversation.
- **Completed the Quickstart?** → Read [CLI Reference](/docs/cli-reference) and [Configuration](/docs/configuration) to customize your setup.
- **Comfortable with the basics?** → Explore [Tools](/docs/tools), [Skills](/docs/skills), and [Memory](/docs/memory) to unlock the full power of the agent.
- **Setting up for a team?** → Read [Security](/docs/security) and [Sessions](/docs/sessions) to understand access control and conversation management.
- **Ready to build?** → Jump into the [Architecture](/docs/architecture) overview to understand the internals and start contributing.
- **Want practical examples?** → Check out the Guides section for real-world projects and tips.

:::tip
You don't need to read everything. Pick the path that matches your goal, follow the links in order, and you'll be productive quickly. You can always come back to this page to find your next step.
:::
