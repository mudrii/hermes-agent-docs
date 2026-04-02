# Learning Path

Hermes Agent can do a lot -- CLI assistant, Telegram/Discord bot, task automation, RL training, and more. This page helps you figure out where to start and what to read based on your experience level and what you're trying to accomplish.

Start here: if you haven't installed Hermes Agent yet, begin with the Installation guide and then run through the Quickstart. Everything below assumes you have a working installation.

## How to Use This Page

- **Know your level?** Jump to the experience-level table and follow the reading order for your tier.
- **Have a specific goal?** Skip to By Use Case and find the scenario that matches.
- **Just browsing?** Check the Key Features table for a quick overview.

## By Experience Level

| Level | Goal | Recommended Reading | Time Estimate |
|-------|------|---------------------|---------------|
| Beginner | Get up and running, basic conversations, use built-in tools | Installation -> Quickstart -> CLI Usage -> Configuration | ~1 hour |
| Intermediate | Set up messaging bots, use memory, cron jobs, and skills | Sessions -> Messaging -> Tools -> Skills -> Memory -> Cron | ~2-3 hours |
| Advanced | Build custom tools, create skills, train models with RL, contribute | Architecture -> Adding Tools -> Creating Skills -> RL Training -> Contributing | ~4-6 hours |

## By Use Case

### "I want a CLI coding assistant"

Use Hermes Agent as an interactive terminal assistant for writing, reviewing, and running code.

1. Installation
2. Quickstart
3. CLI Usage
4. Code Execution
5. Context Files
6. Tips & Tricks

Pass files directly into your conversation with context files. Hermes Agent can read, edit, and run code in your projects.

### "I want a Telegram/Discord bot"

Deploy Hermes Agent as a bot on your favorite messaging platform.

1. Installation
2. Configuration
3. Messaging Overview
4. Telegram Setup / Discord Setup
5. Voice Mode
6. Security

### "I want to automate tasks"

Schedule recurring tasks, run batch jobs, or chain agent actions together.

1. Quickstart
2. Cron Scheduling
3. Batch Processing
4. Delegation
5. Hooks

Cron jobs let Hermes Agent run tasks on a schedule -- daily summaries, periodic checks, automated reports -- without you being present.

### "I want to build custom tools/skills"

Extend Hermes Agent with your own tools and reusable skill packages.

1. Tools Overview
2. Skills Overview
3. MCP (Model Context Protocol)
4. Architecture
5. Adding Tools
6. Creating Skills

Tools are individual functions the agent can call. Skills are bundles of tools, prompts, and configuration packaged together. Start with tools, graduate to skills.

### "I want to train models"

Use reinforcement learning to fine-tune model behavior with Hermes Agent's built-in RL training pipeline.

1. Quickstart
2. Configuration
3. RL Training
4. Provider Routing
5. Architecture

RL training works best when you already understand the basics of how Hermes Agent handles conversations and tool calls.

### "I want to use it as a Python library"

Integrate Hermes Agent into your own Python applications programmatically.

1. Installation
2. Quickstart
3. Python Library Guide
4. Architecture
5. Tools
6. Sessions

## Key Features at a Glance

| Feature | What It Does |
|---------|-------------|
| Tools | Built-in tools the agent can call (file I/O, search, shell, etc.) |
| Skills | Installable plugin packages that add new capabilities |
| Memory | Persistent memory across sessions |
| Context Files | Feed files and directories into conversations |
| MCP | Connect to external tool servers via Model Context Protocol |
| Cron | Schedule recurring agent tasks |
| Delegation | Spawn sub-agents for parallel work |
| Code Execution | Run code in sandboxed environments |
| Browser | Web browsing and scraping |
| Hooks | Event-driven callbacks and middleware |
| Batch Processing | Process multiple inputs in bulk |
| RL Training | Fine-tune models with reinforcement learning |
| Provider Routing | Route requests across multiple LLM providers |

## What to Read Next

- **Just finished installing?** Head to the Quickstart.
- **Completed the Quickstart?** Read CLI Usage and Configuration.
- **Comfortable with the basics?** Explore Tools, Skills, and Memory.
- **Setting up for a team?** Read Security and Sessions.
- **Ready to build?** Jump into the Developer Guide.
- **Want practical examples?** Check out the Guides section.

You don't need to read everything. Pick the path that matches your goal, follow the links in order, and you'll be productive quickly.
