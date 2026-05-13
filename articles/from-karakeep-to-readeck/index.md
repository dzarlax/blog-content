---
title: "Trading a 1.3 GB Bookmark Manager for 17 MB"
description: "Memory pressure on my 7.6 GB VPS forced me off Karakeep. The migration to Readeck snowballed into a small Go project (a multi-tenant Telegram bot and an MCP server) that I now plug into Claude."
date: 2026-05-13
lastmod: 2026-05-13
draft: false
tags: ["build-logs", "systems-automation", "personal-stack", "pet-projects"]
cover: ""
toc: true
---

I had been ignoring it for a while: every time I looked at `docker stats` on my Hetzner VPS, Karakeep was the heaviest container by a wide margin. A 7.6 GB box, 25 services, and one Node app taking **1.32 GB** of resident memory for what is, in my use, a glorified link list with screenshots.

Available RAM had drifted down to **1.4 GB**, swap was pinned at 2.0 / 2.0 GB, and Postgres (capped at 768 MB) was hitting 61% utilisation. Nothing was on fire, but the margin had stopped feeling like a margin.

This post is the story of how I unwound that. It became three things, in order: an audit, a migration, and then, because the off-the-shelf parts didn't quite fit, a small Go project.

## The audit

First step was to figure out what was actually heavy. `docker stats --no-stream` plus `for f in /proc/*/status; do awk '/^VmSwap:/{if($2>0)print …}' $f; done` ranked everything by memory and by swap residency. The top six:

| Container | RAM |
|---|---:|
| `karakeep-web` | 1.32 GB |
| `memory-embeddings` (TEI for RAG) | 964 MB |
| `authentik-server` | 728 MB |
| `city_dashboard` | 622 MB |
| `infra-postgres` | 469 MB |
| `authentik-worker` | 281 MB |

The 2 GB swap was almost entirely cold. `vmstat` showed `si=2, so=3 kB/s`: the swap was full but nothing was actually swapping. The kernel had paged out idle stuff (TEI runs once every 30 min, Evening News workers are also periodic) and kept hot pages in RAM. That pressure was textbook-healthy, not the real problem.

The real fix was straightforward: pulled a fresh Karakeep image (their `:release` tag had drifted three months), added a second 4 GB swapfile, and dropped `vm.swappiness` from the Ubuntu default of 60 down to 20. After:

| Metric | Before | After |
|---|---:|---:|
| RAM used | 5.4 GB | 4.4 GB |
| Available | 1.4 GB | 2.7 GB |
| Swap | 2.0 / 2 GB (100%) | 1.9 / 6 GB (32%) |

A simple version bump on Karakeep dropped its RAM from 1.32 GB to 818 MB, about 500 MB just from a fresh process and a clean Node heap. The box was breathing again.

But once you've shaved 500 MB off a 800 MB target, you start wondering whether you need that 800 MB at all.

## The choice

Karakeep has real strengths: AI auto-tagging, a chrome-driven snapshot/screenshot pipeline, and native mobile apps with share extensions. For me though, the bulk of saves are URLs forwarded from Telegram, and I read them later on iOS. AI tags are nice-to-have. Screenshot archives are nice-to-have. A working mobile share path is not.

My first instinct was [linkding](https://github.com/sissbruecker/linkding). Python/Django, around 100 MB resident, and an unusually rich third-party ecosystem: three iOS apps (LinkBuddy, Linkdy, LinkThing), four Android clients (LinkBuddy, Linkdy, Linklater, Pinkt), several Telegram bots, browser extensions, even a "linkding-injector" that surfaces your saved links inside Google or DuckDuckGo results. It would have worked.

What stopped me was the UI. linkding is a competent internal tool, dense and functional. The aesthetic is "ticket tracker 2019", and I'd be looking at this thing daily, on my own time. That isn't a technical objection, but it's a real one.

Next on the list was [Readeck](https://readeck.org/). Same Go family as the rest of my stack, around 60 MB resident, OPDS support for shipping bookmarks to an e-reader as EPUB. The previous time I'd looked at it I'd dismissed it because there were no native mobile apps. This time I checked the release notes and found that [v0.22 in March 2026](https://readeck.org/en/blog/202602-readeck-22/) shipped an iOS app with OAuth and offline reading, plus an active Android client on [F-Droid](https://f-droid.org/packages/de.readeckapp/). The blocker was gone.

[Shiori](https://github.com/go-shiori/shiori) was on the list for completeness. Go binary, around 30 MB, deliberately minimal: no mobile apps, no AI, beta-stage browser extension. Right tool for someone, wrong tool for me.

The remaining honest objection to Readeck was Karakeep's AI auto-tagging. I sat with that question for a while. The boring answer is that in practice I almost never actually queried bookmarks by the tags Karakeep generated; they were nice to read after the fact, not a retrieval pivot. The more interesting answer is that with an MCP server in front of Readeck, AI tagging becomes BYO-LLM: Claude can call `readeck_get_article`, summarise it, and call `readeck_add_labels`, on demand and in my own taste of tags. That isn't a feature I'm losing. It's a feature I'm deferring to a place I control.

Readeck won. Cleaner reading UX than linkding, mobile story now sufficient, Go binary, OPDS for the e-reader, and the AI-tagging objection turned out to be addressable rather than fatal. The trade-off that does stick is the lack of full Chrome snapshots. Readeck extracts article text and saves inline resources but doesn't store a visual page archive. For the way I use a bookmark manager, that's a fair trade.

## Wiring it up

Two pieces of existing infra carried the deploy: shared Postgres (`infra-postgres`) and authentik SSO.

For Postgres I just created a `readeck_user` role and a `readeck` database in the same instance Authentik and a handful of other services already use. The Readeck container connects with `READECK_DATABASE_SOURCE=postgres://readeck_user:…@infra-postgres-1:5432/readeck?sslmode=disable`. One backup pipeline covers everything.

SSO was harder than I expected.

Readeck supports forwarded authentication out of the box: it reads `Remote-User`, `Remote-Email`, `Remote-Groups` headers from a trusted reverse proxy. Authentik's proxy outpost emits `X-authentik-username`, `X-authentik-email`, `X-authentik-groups`. Traefik's built-in middlewares can't rename headers. They can set static custom headers, but they can't template from incoming ones.

My first attempt was a Caddy sidecar in front of Readeck:

```caddyfile
:8080 {
  reverse_proxy readeck:8000 {
    header_up Remote-User   {http.request.header.X-Authentik-Username}
    header_up Remote-Email  {http.request.header.X-Authentik-Email}
    header_up Remote-Groups admin
  }
}
```

It worked: about 10 MB of RAM for the shim, two containers per service, but functionally correct.

Then I went looking for a cleaner path and found one: Authentik scope mappings can return a special key, `ak_proxy.additionalHeaders`, that the outpost emits as arbitrary response headers. The scope mapping for Readeck became:

```python
return {
    "ak_proxy": {
        "user_attributes": request.user.group_attributes(request),
        "is_superuser": request.user.is_superuser,
        "additionalHeaders": {
            "Remote-User":   request.user.username,
            "Remote-Email":  request.user.email,
            "Remote-Groups": "admin" if request.user.is_superuser else "user",
        },
    }
}
```

One readeck-specific `authResponseHeaders=…,Remote-User,Remote-Email,Remote-Groups` on a forwardAuth middleware in Traefik labels, no sidecar. The Caddy idea died a quiet death.

Two more constraints showed up during testing:

1. **Mobile apps use API tokens, not OAuth cookies.** Authenticating every `/api/*` request through Authentik would break the iOS app and any Bearer-token client. The fix: a second Traefik router for `Host(read.dzarlax.dev) && PathPrefix(/api)` that bypasses the SSO middleware and forwards straight to Readeck. Readeck handles bearer auth itself for those requests.
2. **Let's Encrypt rate-limited me.** I had pushed the deploy before the DNS record propagated. Five quick ACME failures triggered a "too many failed authorisations for `read.dzarlax.dev`, retry after …" lockout. Waited the window, restarted the container, certificate issued on the first retry. Lesson: don't deploy a TLS-terminated route until the DNS is actually live.

## The toolkit

This is where it stopped being a migration and started being a project.

The old Telegram→Karakeep pipeline used [karakeepbot](https://github.com/madh93/karakeepbot), a third-party Go bot wrapping Karakeep's API. Same Telegram bot identity in BotFather (I reused the existing token), but the new backend speaks a different API. So a new program had to talk to it.

I also wanted Readeck inside Claude. My existing personal-memory and calendar MCP servers were starting to feel useful enough that adding "search my bookmarks" and "read this article" felt obviously valuable. Easier to do that with the same codebase that runs the bot, since they share an API client.

The repo is [Dzarlax-AI/readeck_toolkit](https://github.com/Dzarlax-AI/readeck_toolkit): one Go module, two binaries (`cmd/bot`, `cmd/mcp`), one Docker image. The shape that landed after a few iterations:

- **Bot** is multi-tenant via `config.toml`. Each `[[tenants]]` block maps a Telegram user id to a Readeck API token. Incoming message → look up sender id → use that user's token to call Readeck → bookmark lands in the right account. Unknown senders are silently dropped. Onboarding a new person (e.g. for shared household use) is: they create a Readeck API token in Settings → API tokens, DM `/whoami` to the bot to get their numeric id, you append a tenant block and `docker compose restart bot`.
- **MCP** is the opposite: stateless and credential-less. The server only knows the Readeck base URL. Each connecting MCP client supplies its own Readeck token via the `X-API-Key` header on connect; the tool handler pulls the token from request context and constructs a per-call Readeck client. One MCP instance is safely shareable: Readeck enforces per-token scoping. No tenant config on the server, no secrets on disk.

A couple of dead ends here too, worth naming:

- **First MCP design used SSE transport.** mcp-go's `NewSSEServer` was the easy default. It worked locally. Behind Traefik's `stripprefix` middleware, the `endpoint` SSE event advertised `/message?sessionId=…` without the `/readeck` prefix, so the client followed up to the wrong route. Fixing it meant dropping `stripprefix` and telling the server the full public path. Then I rechecked the spec and found that **SSE is now legacy**. Current MCP servers should use Streamable HTTP (single POST endpoint, optional chunked streaming response). `server.NewStreamableHTTPServer(s, server.WithStateLess(true), server.WithEndpointPath("/readeck/mcp"))` ended up being both cleaner and easier to reason about for a multi-tenant setup.
- **First `config.toml` had a separate `[mcp]` section with a `tenant` field** so the MCP could pick which user it acted as. After the `X-API-Key`-per-request redesign, that whole section evaporated. The MCP just needs `[readeck].base_url`; user identity arrives in headers.

Final tool surface (nine functions):

```
readeck_save           save URL, optional title + labels
readeck_search         full-text search
readeck_list_recent    last N bookmarks
readeck_get_article    extracted article, HTML→Markdown server-side
readeck_mark_read      flip is_archived
readeck_add_labels     append labels, preserve existing
readeck_remove_labels  drop specified labels
readeck_delete         permanent delete
readeck_list_labels    every label with bookmark count
```

`readeck_get_article` is the one that earns its keep inside Claude. The natural prompt is something like *"find my saved article about Postgres `shared_buffers` and summarise its concrete recommendations"*. Claude calls `readeck_search` with the query, picks the most relevant id off the returned list, calls `readeck_get_article`, and produces the summary. From my side I never opened the web UI. Pair this with `readeck_add_labels` and you have the BYO-LLM tagger that closes the Karakeep gap from the previous section.

## Where it landed

The bot and MCP together use **7 MB** of RAM at idle. Karakeep, which I stopped but kept on disk for a fortnight in case I regretted the move, was 818 MB. Forty-seven times the per-service footprint, in trade for no visual page snapshots, a less polished iOS experience, and roughly three afternoons of work.

The Karakeep removal is scheduled in Todoist for two weeks out, long enough to discover what I miss. So far the answer is "the automatic AI tags, occasionally", which is exactly the gap the MCP is positioned to fill once I sit down and write the Claude prompt for it.

## What's next

The obvious v2 is a save-hook in the bot. Right now the bot fires `POST /api/bookmarks` and replies. If I extend it to also `POST {id, url, manual_labels}` to a `save_hook_url` configured per tenant, anything on the receiving end (a Claude routine, an n8n flow, a small script with a model API key) can fetch the article via the MCP, generate AI tags and a summary, and write them back through `readeck_add_labels` plus a description update. The building blocks are already in the toolkit: the bot fires cleanly on save, the MCP can read and write everything needed. Only the webhook glue is missing. Roughly an evening of work, and it closes the "automatic" half of the AI-tags gap without re-baking an LLM into the toolkit itself. The whole point of the move was BYO-LLM, so the hook stays a hook.

Repo: [github.com/Dzarlax-AI/readeck_toolkit](https://github.com/Dzarlax-AI/readeck_toolkit). It's MIT, multi-arch image at `ghcr.io/dzarlax-ai/readeck-toolkit`. Even for a single user the bot earns its place: the iOS share sheet sends a URL straight to it in one tap, the bot saves with optional `#hashtag` labels and replies with the Readeck link, and you never open the web UI just to file a save. The multi-tenant config is there for the case where you want to share an install with a partner or a small team. The MCP server is for anyone using Claude (or any MCP-capable client) with their own Readeck.
