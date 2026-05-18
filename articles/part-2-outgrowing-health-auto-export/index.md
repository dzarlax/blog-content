---
title: "Health Dashboard, Part 2: Outgrowing Health Auto Export"
description: "Part 2 of my Health Dashboard build log: why a generic Apple Health exporter stopped being sufficient on its own, what adding a native iOS client cost in regressions, and how `sleep_core` turned out to be lying for years. Health Auto Export still runs alongside the native client. This is about outgrowing its limits, not retiring it."
date: 2026-05-18
lastmod: 2026-05-18
draft: false
tags: ["build-logs", "pet-projects", "systems-automation"]
series: "health-dashboard"
series_part: 2
cover: ""
toc: true
---

In Part 1 the server learned to accept observations without losing them and to rebuild every derived table from raw data. That was enough to make a dashboard useful. It was not enough to make the data honest.

This part is about the moment a third-party exporter stopped being a sufficient ingestion path on its own, and about the two regressions that followed when I added my own iOS client alongside it. Health Auto Export still runs against the server today. The `/health` endpoint accepts both producers, and the code path that parses HAE payloads is maintained. The "outgrowing" in the title is about its limits, not its retirement.

## Why Health Auto Export was the right starting point

Health Auto Export gave me a working ingestion surface in an evening. JSON payloads over HTTP, configurable schedule, a list of metric names that already lined up with what the server wanted to store. No Xcode, no entitlements, no provisioning profiles. The payload shape it emitted is the same one the server speaks today:

```json
{"data": {"metrics": [{"name": "...", "units": "...", "data": [...]}]}}
```

For the first phase of the project ("show me a number in the morning") that was the right tradeoff. I wanted to learn what the server needed to become, not write an iOS app for a system that did not yet have a point.

## Where it stopped fitting

The exporter is generic by design. Each thing the server learned to care about turned that generality into a problem.

- **Source identity.** The server needed to know whether a sleep row came from Apple Watch (staged) or RingConn (coarse summary). Health Auto Export shipped the source string, but it shipped both rows when the same night was covered twice, and it had no way to suppress one in favour of the other.
- **Sync timing.** The morning report only fires when last night's sleep data is "settled": last segment ended ≥45 minutes ago, no fragments in the last 20 minutes. With a generic exporter on its own schedule, the server could not tell whether silence meant "no data yet" or "no Watch connected".
- **Partial days.** A failed sync mid-day left the server reasoning about a date that was half-real and half-missing. With no `HKQueryAnchor`[^hkqueryanchor] on the client side, recovery meant manually re-exporting a date range from the app.
- **Workouts.** Originally not supported by either path. The server had no workouts endpoint, and HAE's default schema does not push them. This one resolved itself differently from the others, as the section below describes.
- **Sleep stages.** The exporter passed through whatever HealthKit returned, including the coarse `.asleepUnspecified` layer that Apple Watch on iOS 26 emits alongside the fine-grained stages. The server had no way to know one was a duplicate of the other.

The last item is where the cost compounded later. More on that below.

## Adding a native client alongside HAE

The next step was a native iOS app, `health-sync`. Swift 6, iOS 26+, no third-party dependencies. It is a second producer feeding the same `POST /health` endpoint, not a replacement for Health Auto Export. HAE keeps doing what it is good at; the native client picks up the cases HAE cannot reach. Both producers share the source-of-truth path described in Part 1: raw payload into `health_records` first, derived tables after.

Why bother with the native client at all if HAE works for most metrics? Because the phone is the device with direct HealthKit[^healthkit] access, and a few decisions only make sense on that side of the wire. The diagram below uses Apple's own query types[^hk-queries]:

```
HKObserver (all metrics) → wakeup
    ↓
SUM metrics  → HKStatisticsCollectionQuery (hourly buckets)
AVG metrics  → HKSampleQuery (raw samples, minutely)
Workouts     → HKWorkoutQuery + HKWorkoutRoute + HR samples
    ↓
Merge into single payload
    ↓
POST /health  (one request)
```

Three design rules came out of the SPEC and stuck:

1. **One request per sync cycle.** No per-metric requests, no vitals/hourly split. The client batches every changed metric into one `POST /health`. The server already accepts mixed payloads.
2. **`HKQueryAnchor` per metric.** Each sync fetches only samples since the last successful delivery. Anchors live in SwiftData[^swiftdata], keyed by metric name.
3. **Source filtering at the client, before sending.** If Apple Watch data exists for a date, the client skips RingConn midnight summaries for that date. The server still does its own cross-validation, but the client removes the obvious noise upstream so the server's logs stay readable.

The server side stayed the source of truth. The native app has no scoring, no aggregation, no sleep dedup, no source priority. Those all live behind `/api/health-briefing`, `/api/metrics/data`, `/api/readiness-history`. The client only handles ingestion and chart rendering. When new metrics or copy ship, no app update is needed.

That split sounds clean on paper. It cost two visible regressions to get to a state where the data agreed with itself.

## Workouts came back through the other path

The "workouts not supported" bullet in the constraints list above resolved itself in a way I did not plan for. In May 2026 an external contributor, [makvitaly](https://github.com/makvitaly), opened [PR #17](https://github.com/Dzarlax-AI/health_dashboard/pull/17) on the server adding a `POST /health/workouts` endpoint and the storage helpers behind it. The endpoint takes the JSON shape Health Auto Export's "Workouts" automation emits (duration, distance, energy, heart-rate timeline, route) and writes it into a new `workouts` table alongside the per-metric `metric_points` stream. As of 2026-02-13 onward, that endpoint has been the source of every structured workout in the database. The 102 walking entries that Part 4's audit found there all came in through this HAE path, not through the native client.

That outcome is the two-producer architecture working as intended. The "wrong" thing was not "HAE cannot do workouts". It was "the server did not have an endpoint to receive them, so neither producer could push them". Once the endpoint existed, HAE became the path that actually fills it for me today. The native client could grow its own workout sync (and the SPEC has it as a planned feature), but there has been no reason to prioritise that while HAE already delivers.

This is also why the project lists HAE as a supported ingestion path in the README rather than as a legacy compatibility shim. Someone who wants workouts but does not want to build the iOS client gets them, through a code path that an outside contributor added without ever needing my help.

## Round 1: per-segment sleep records disappeared

On 2026-03-24, the day `health-sync` started feeding sleep data into `metric_points` alongside HAE, the per-segment records stopped landing for the nights the native client owned. The database still got a nightly `sleep_total`, but the deep / REM / core breakdown was gone for those nights.

Cause: the first version of `HealthKitManager.fetchSleep` collapsed every `HKCategorySample` in a session into a single `Accum` struct, then emitted one row per phase per night. From the server's point of view, every night arrived pre-aggregated. The per-segment time series that drives the stages-stacked chart simply did not exist anymore.

The fix landed as health-sync PR #3 (`feat(sleep, workouts): ship per-segment sleep + workouts`). Each `HKCategorySample` is now emitted as its own row with `metric_name ∈ {sleep_deep, sleep_rem, sleep_core, sleep_awake}` and the segment's actual start/end. The server's `extractPoints` already knew how to expand a `sleep_analysis` payload into five metrics, so no server change was needed.

The interesting part is what this regression made visible. With Health Auto Export, I had never inspected what a per-segment sleep row looked like; the exporter was emitting them and the server was storing them and the chart was rendering them, and the whole pipeline worked by accident of compatibility. The first time I owned the producer side, the contract broke immediately. That is the price of generality: it hides the protocol.

## Round 2: 17-hour nights from 8-hour sleep

PR #3 made stages reappear on the dashboard. It also made the stages-stacked chart visibly wrong. Nights that I knew were ~8 hours of sleep were rendering as 12–17 hours of stacked bars.

The cause took a while to find because both layers looked legitimate when read separately. On iOS 26, Apple Watch emits sleep samples in two layers for the same wall-clock interval:

1. A coarse `.asleepUnspecified` (or legacy `.asleep`) span covering the whole night.
2. Fine-grained `.asleepDeep` / `.asleepREM` / `.asleepCore` segments stacked on top. This is the same breakdown Health.app renders.

PR #3 emitted both. The server's `metric_name='sleep_core'` accumulator did the only thing it could do: it summed them. An 8-hour real night with `deep=1.5h + rem=2h + core=4.5h` became `core ≈ 4.5h + unspecified-routed-to-core ≈ 8h ≈ 12.5h` of "core" in `daily_scores`.

The simple sanity check from then on:

```sql
SELECT sleep_total, (sleep_deep + sleep_rem + sleep_core + sleep_awake) AS stages_sum
FROM daily_scores;
```

`stages_sum` should approximate `sleep_total` within a small awake/in-bed margin. A 2× divergence means the overlap bug is back.

The fix landed as health-sync PR #10 (`fix(sleep): skip coarse asleepUnspecified when stages exist`). Per session, the client checks whether any fine-grained marker is present. If yes, the coarse layer is dropped from both the aggregate accumulator and the per-segment emission. If no, meaning older Apple Watch, RingConn, or iPhone Sleep Schedule estimates, the coarse layer is kept as the only available signal. `asleepHours()` got the same guard so main/nap classification stays consistent with what gets emitted.

After deploy, existing "dirty" dates on the server were re-fetched by the app through the same path: bump the relevant `HKQueryAnchor`, let the next observer wakeup re-emit, server-side `UpsertRecentCache` rebuilds the affected dates from `metric_points`.

## What `sleep_core` was actually saying

PR #10 stopped the double-counting, but the underlying lie remained: the column called `sleep_core` was mixing two physiologically different things.

- For Apple Watch with stage tracking: real "core sleep" minutes, Apple's heuristic for light non-REM sleep.
- For RingConn, iPhone Sleep Schedule, older Apple Watch: a coarse "just asleep" span with no stage information at all.

The dashboard's stages chart rendered both as "Core". A reader could not tell from the column whether the night had actually been measured at stage resolution or whether the source had just said "asleep, no idea what kind".

The fix was to introduce a fifth phase metric, `sleep_unspecified`. The wire contract:

```json
{
  "name": "sleep_unspecified",
  "units": "hr",
  "data": [{"date": "2026-05-14 23:00:00 +0200", "qty": 7.5, "source": "RingConn"}]
}
```

iOS 2.3 emits `sleep_unspecified` only when the session has no `.asleepDeep` / `.asleepREM` / `.asleepCore` markers anywhere. Sources that report stages continue to emit `sleep_core` as the real measurement. After the rollout, RingConn-only and iPhone-only nights stopped landing in `sleep_core` and started landing in `sleep_unspecified` instead.

Server-side changes were small and ordered to stay backward-compatible at every step:

1. Add `sleep_unspecified` to the daily aggregator so `sleep_total = deep + rem + core + unspecified` becomes the new invariant.
2. Add the metric to `/api/metrics` with localized display name and description in en/ru/sr.
3. Dashboard adds a 5th band to the stages chart between `rem` and `awake`, with the tooltip "No per-stage breakdown from this source".
4. A historical migration (`cmd/migrate_sleep_unspecified`) moves pre-v2.3 `sleep_core` rows that have no sibling `sleep_deep` / `sleep_rem` within ±1 calendar day to `sleep_unspecified`, then targeted-rebuilds those dates through the same `UpsertRecentCache` path that a fresh ingest uses. The ±1 day window protects Apple Watch staged nights crossing midnight from being mis-classified.

The migration has a known false-negative case: if a source ever emitted any stages, `±1 day` around those nights stays in `sleep_core` regardless of whether those specific rows are genuinely coarse. Acceptable for my install, documented for anyone else who clones the repo.

## What owning the client actually changed

The two regressions are not the interesting result. They were predictable: any time you replace a generic producer with a custom one, the implicit contract surfaces as bugs. The interesting result is what became expressible after the switch.

- **Source filtering at the producer.** The client can decide "Apple Watch is here for this date, don't ship RingConn's midnight summary." With Health Auto Export, every source's interpretation of the night arrived at the server and the server had to argue with itself about which to trust.
- **Stages vs coarse as separate concepts.** `sleep_unspecified` only exists because the client can inspect a HealthKit session and decide which bucket the data belongs in. Telling them apart server-side would have required guessing from source name strings.
- **Workouts (the second producer).** The native client also grew its own workout sync. Per-workout HR timelines, GPS routes, time-overlap apportionment of steps and calories. Mostly between PRs #4 and #9 on the iOS side. It posts to the same `/health/workouts` endpoint PR #17 added on the server, so both producers are *capable* of delivering workouts and the table does not care which one wrote the row. In practice, since HAE already filled that table reliably from 2026-02-13 onward, the native client's workout path has stayed available but largely unused for live ingest. Useful insurance, not yet load-bearing.
- **Recovery from gaps.** When a sync fails, the next observer wakeup re-fetches from the per-metric `HKQueryAnchor`, the failed payload sits in `RetryQueue`, and the server's `UpsertRecentCache` rebuilds the affected dates as if it were a fresh ingest. No manual re-export step.

There is also a thing I am not doing yet, deliberately. The app does not cross-check what HealthKit currently knows on the device against what the server believes. That feature is in the SPEC marked out of scope for v1, because it changes the trust model: today the device is the producer and the server is the keeper; cross-check would make the device the auditor too. I want one role at a time.

## The actual lesson

Health Auto Export was not the wrong choice and it is not gone. It is still the right shape for most of what it does. Scheduled JSON over HTTP, no Xcode, no provisioning. The server still accepts it, and the parser that handles its payloads is maintained in the same code path that processes the native client's payloads.

What changed is that HAE stopped being *sufficient on its own* for me. The moment the server started having opinions about the data, about which source was authoritative, about whether one HealthKit layer was a duplicate of another, about what a column was actually measuring, those opinions had to be expressible somewhere. A generic exporter cannot have them on your behalf. It can only pass through what HealthKit gave it. The native client is where those opinions live; HAE keeps doing the rest.

Personally I run the native client now. It is what feeds my own dashboard every day. But the project is self-hostable, and the HAE path stays a first-class ingestion option for anyone who clones the repo and does not want to build an iOS app from source. The two-producer architecture is not just a legacy artefact; it is the supported way for someone else to stand the system up with a TestFlight-free path. Anyone who wants the source filtering and stage handling described above can build `health-sync` themselves. Anyone who does not gets a working dashboard from HAE alone, workouts included thanks to PR #17, and inherits the same trade-offs.

The previous article was about trusting the data. This one was about trusting the columns. The next is about trusting the sources behind those columns: Apple Watch, RingConn, and iPhone do not agree about what they measured, and the system has to decide which one to believe on each given night before the rest of the pipeline can do anything useful with the result.

[^hkqueryanchor]: HealthKit's incremental-sync cursor. An opaque token returned by `HKAnchoredObjectQuery` that lets the next query fetch only samples added since the previous successful call. Stored per-metric in SwiftData so the client never re-reads old data; if a sync fails, the anchor is not advanced and the next observer wakeup retries from the same point.
[^swiftdata]: Apple's persistence framework introduced in 2023 as a Swift-native successor to Core Data. Schema-from-source-of-truth (`@Model` classes), automatic migration for simple schema changes, and tight integration with SwiftUI.
[^healthkit]: Apple's framework for storing and querying health and fitness data on iOS. All sensor readings from Apple Watch, third-party sleep trackers like RingConn, and manual entries flow through HealthKit on the device before any export reaches a server.
[^hk-queries]: HealthKit exposes several query types tuned for different access patterns. `HKObserverQuery` wakes the app when new data of a given type arrives (even in the background, with `enableBackgroundDelivery`). `HKSampleQuery` returns raw individual readings within a date range. `HKStatisticsCollectionQuery` returns pre-aggregated bucketed values (hourly sums, daily averages, etc.) which is cheaper than fetching raw samples and aggregating client-side. `HKWorkoutQuery` and `HKWorkoutRoute` return structured workout sessions with their associated route and HR samples.
