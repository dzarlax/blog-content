---
title: "How a Sleeping Terminal Cost Me $1,000 Overnight"
description: "A Claude Code post-mortem on background daemons, prompt caching, and infinite agent loops — and how a strict spending limit failed to save my bill."
date: 2026-05-08
lastmod: 2026-05-08
draft: false
tags: ["product-ai", "systems-automation", "claude-code"]
cover: ""
toc: true
canonical: "https://dzarlax.substack.com/p/how-a-sleeping-terminal-cost-me-1000"
---

Picture this: you're working on a project, using Claude Code to refactor a massive monorepo. Evening comes, you close your laptop lid, and in the morning you wake up to a **$1,025** API bill — even though you had strict spending limits set up in your billing dashboard.

That's exactly what happened to me. After a deep dive into the logs, we reconstructed the timeline and found the root cause. Spoiler: the culprits were background daemons, infinite agent loops, and — ironically — the prompt caching mechanism itself.

## The setup

I was working on a project with a massive codebase. Every request to Claude pulled in roughly **250,000 tokens** of context.

As a wrapper for Claude Code, I was using **AgentDeck** — a handy tool that acts as a bridge and creates its own PTY terminal managed by a background daemon. That daemon is where this story begins.

## The prompt-caching trap

Many people think that cached token reads are basically free. They aren't.

The cache itself was working as designed. Session logs showed exactly what you'd expect:

- `input_tokens`: 1
- `cache_read_input_tokens`: 242,304

But reading from the cache still carries a per-token cost. For Claude 3 Opus, reading 1M cached tokens costs $1.50. That means every single response in this session cost roughly **$0.37** — before any output tokens.

A 90% discount on 250,000 tokens still adds up fast when something is hitting the model on a loop.

## The "ghost" session

Normally, when you close a terminal window, you kill the process. But with the AgentDeck daemon in the middle, the Claude process was kept alive in the background instead.

When the laptop went to sleep, the session paused. macOS, however, periodically wakes the machine for background tasks (Power Nap). The daemon dutifully revived the session each time — in the logs, this showed up as `System wake detected`.

At some point, the agent got stuck in a loop. It was trying to complete a task and constantly re-checking the status.

A surviving log fragment told the rest of the story:

- `messageCount`: 330
- `durationMs`: 929,873 (~15.5 minutes)

The agent was making **one request every ~3 seconds**. Every request cost ~$0.37 in cache reads.

**Total burn rate: ~$1.50 per second, or $120–$140 per hour** — for a terminal I thought was closed.

## Anthropic's limit didn't fire

The most frustrating part: I'm on a **Claude for Work (Team) subscription** with a strict Hard Limit configured in the org's web console. It didn't trigger.

Support later confirmed there was a bug where API billing limits were being ignored for Claude Code requests.

On top of that, inside the `claude` CLI itself, an `extraUsage` flag was set to `true`, effectively giving the agent a blank check against my card.

## Takeaways

If you're using autonomous coding agents, here's the checklist I run now:

1. **Don't trust daemons.** If you use wrappers like AgentDeck or Warp Agent, verify that they actually kill sessions when the window closes. I uninstalled AgentDeck.
2. **End sessions explicitly.** Don't leave the terminal hanging. Type `/exit` or `Ctrl+D`.
3. **Set hard limits where they're actually enforced.** Right now that's the web console, not the Claude Code config. If you find the limit doesn't fire — assume it doesn't and act accordingly.
4. **Audit subagents.** The `/subagents` command shows whether you've left a silent agent running in the background.
5. **Be careful with `--auto`.** Infinite autonomy on a large monorepo can burn through a month's budget in an hour if the agent gets stuck in an error-correction loop.

The only thing that finally stopped this $1,000 spending spree was logging out of the CLI. That's a thin line of defence for a tool that can spend money this fast.
