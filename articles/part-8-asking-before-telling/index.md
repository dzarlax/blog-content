---
title: "Health Dashboard, Part 8: Asking Before Telling"
description: "Part 8 of my Health Dashboard build log: the newest piece of the system, a one-tap morning check-in over Telegram. Not yet proven useful, still under observation, with explicit non-uses to protect against the failure modes Part 6 warned about."
date: 2026-05-18
lastmod: 2026-05-18
draft: false
tags: ["build-logs", "pet-projects", "systems-automation"]
series: "health-dashboard"
series_part: 8
cover: ""
toc: true
---

Part 6 said: subjective input cannot validate the stress formula. That rule stands. A formula that auto-tunes itself against a user's self-report becomes a closed loop with no external referent. The user rationalises post-hoc, the formula updates toward the rationalisation, and the next round of self-report is biased by the formula's previous output. The cure for that is to keep the rubric external: next-morning HRV, RHR shift, sleep architecture. Numbers the user is not consciously writing down.

So why, several weeks later, did I ship a one-tap morning check-in that asks the user how they feel?

Because the rule applied to calibration. It did not apply to everything subjective input could be doing, at least in theory. This part is about what those other roles might be, and an honest note up front: **this is the most recent feature in the whole series and the one that has not yet proven its keep.** A few days of production use is not a verdict. The check-in is being observed, not declared a success. Everything that follows is the design as it stands today and the hypotheses behind it, not a retrospective on a settled question.

## What the check-in is, mechanically

Every morning, on the same scheduler tick that would have sent the morning report, the server fires a Telegram message:

```text
How do you feel today?
[ Great ] [ OK ]
[ Meh   ] [ Sick ]
```

Four inline-keyboard[^inline-keyboard] buttons. Tap one, the message updates to "Saved", the morning report fires a few seconds later. If no tap arrives before the morning cap (`MorningCapHour`, normally 11:00 local), the row is marked `expired`, the report sends anyway with a one-line soft note ("No morning check-in today"), and a tap arriving later that day is recorded as `late_answered` for analytics only.

The wire shape is a Telegram callback. The button's `callback_data`[^callback-data] is `checkin:<answer>:<YYYY-MM-DD>`. 26 bytes worst case, under Telegram's 64-byte limit. When the user taps, Telegram POSTs the callback to `/api/telegram/webhook/<secret>`[^webhook] on the server. The server validates the path secret, validates the `X-Telegram-Bot-Api-Secret-Token` header, looks up the tenant by `chat_id` (so a valid Telegram-signed callback aimed at the wrong tenant is rejected), parses the payload, and writes the row.

The whole feature is a few hundred lines of Go, one table, two scheduler hooks, and one webhook handler.

## Why these four buttons

The original idea was a 1–5 slider, the way Whoop and Oura collect subjective input. I rejected it. The reasons are the same ones Part 6 laid out for stress validation, and they apply equally to any subjective surface, no matter what the data is used for downstream:

- A continuous scale invites confirmation bias. "I had a deadline today, so I must have been stressed, log 4." Discrete categories with no implicit ordering between "low" and "medium-low" cut the temptation to find the answer that confirms the score.
- 1–5 sliders default to "3" on busy mornings, and most mornings are busy. Three of four discrete buttons fit on one Telegram mobile row, and none of them is obviously the "I am not thinking" option.
- The data downstream wants categories anyway. "Sick" is a separate flag, not a 4 on a slider. The action diverges (illness signature in Part 6, AI recommendation pivots to rest), and you do not want to lose that to a 1-pixel slider movement.

The four categories I landed on map roughly to: "above my normal" (Great), "at my normal" (OK), "below my normal" (Meh), "I am unwell" (Sick). The asymmetry of three buttons for a normal-ish day and one for illness is intentional. Illness is the case where the system most needs an explicit signal it cannot derive from physiology alone (wrist temperature data only goes back to October 2025, RHR is lagged, HRV is sparse), and where downstream behaviour changes the most. Giving it its own button is cheaper than asking the user to choose between "Meh" and "Sick" on a 5-point scale.

The labels are short English words by deliberate choice. The Telegram client renders inline-keyboard buttons in a narrow horizontal layout, and longer translations would wrap awkwardly. The localisation happens elsewhere (the prompt text above the buttons is fully localised), but the buttons themselves stay terse.

## What the answer is not used for

Three explicit non-uses, each one a temptation I had to refuse:

**The answer does not tune β in the stress formula.** Part 6's rubric requires four objective channels (next-morning HRV, next-morning RHR, sleep architecture, test-retest stability) and the check-in does not become a fifth. Adding a self-reported channel would re-open exactly the feedback loop the rubric exists to close.

**The answer does not retrain the readiness sub-scores.** Part 4 closed Phase 1 on naive baselines because the linear and tree models did not clear the predeclared floor. Adding a "user said meh today" feature to the models is the kind of thing that *would* lift AUC, and the lift would be artefact. The user's tap is highly correlated with the inputs the model already sees (a "Meh" tap correlates with last night's sleep, which the model already reads). It would be predicting itself.

**The answer does not modify the dashboard's computed numbers.** Readiness stays whatever the server computed it to be. The check-in shows up as a separate one-line confirmation on the dashboard ("Your morning answer: Meh"), positioned alongside the numbers, never blended into them.

Each of those non-uses is a concrete piece of code that does *not* exist. The check-in row in `subjective_checkins` is read by exactly two places in the codebase: the morning-report scheduler (to gate the send) and the dashboard renderer (to print the one-line confirmation). Nothing else.

## What the answer is supposed to be for

Three intended roles, none of which look like "calibration". Each is a hypothesis the next few weeks of running will either support or quietly invalidate. I am writing them down here as the *design intent*, not as outcomes:

**Gating the morning report.** This is the primary hypothesised purpose. The morning report does not fire until the user has either tapped a button or the cap has passed. The bet is that this single rule does most of the work of building a habit: the user opens Telegram in the morning, sees the question first, taps, then sees their numbers. The act of answering becomes the entry point. The numbers arrive *after* a small subjective commitment, not before. That inverts the usual relationship of "the device tells you how you feel." Whether that inversion actually anchors a habit on me, or whether I just learn to autopilot-tap "OK" every morning, is the open question the production run is meant to answer.

**Coverage signal on the dashboard.** The hero block on `/` shows a one-line "Your morning answer: …" when an answer exists. When the day expires without one, the dashboard shows nothing. Not a "Tap to log!" CTA, just absence. Absence is the signal: a row of days with `expired` status next to days with `answered` is a coverage map for the habit itself, separate from the coverage maps for sleep and HRV.

**Future narrative review.** Part 4 flagged the 2022 strict-event-rate spike (5.7% vs 1–2% in other years) as "possibly an illness cluster worth manual narrative review before feeding into trained models." That phrase was a placeholder for "I have nothing to cross-reference against." With a check-in log accumulating, in two years I will be able to look back at any anomaly cluster and ask: were these days tagged "sick"? Was there a sustained "meh" run before they started? That is not training data. No model trains on it, no formula calibrates against it. It is the kind of evidence that lets me decide whether an anomaly cluster was physiology or instrumentation, with at least one independent channel that did not see the formula's output.

The third bullet is why `late_answered` exists as a separate state. A tap arriving an hour after the cap is not part of the primary validation pool. Too much of the day has unfolded, the answer is no longer "before the morning report"; it is "in retrospect, after I saw the numbers". Mixing it into `answered` would let answer-latency drift quietly contaminate any cohort analysis later. So both buckets exist, the dashboard can render either ("Your morning answer: …" / "Logged late: …"), and the analytics queries can pick which pool they want.

## The state machine

Four states, with explicit transitions encoded in a pure Go helper that exists primarily so the policy is auditable in one place:

```go
nextCheckinStatus(current, action, expiresAt, now)

// actions:
//   "tap"    : user pressed a button
//   "expire" : scheduler decided the cap has passed
```

- `prompted` → `answered` when the user taps before `expires_at`
- `prompted` → `expired` when the scheduler runs `ExpireCheckin` after `expires_at`
- `prompted` → `late_answered` when the tap arrives after `expires_at`
- `expired` → `late_answered` when a tap eventually arrives
- `answered` → `answered` on re-tap (idempotent[^idempotent], the closed set protects against double-counting)

{{< details summary="Show `nextCheckinStatus` in full (Go, 22 lines)" >}}

```go
// nextCheckinStatus computes the target status for a row given its
// current state, the action applied, the row's expires_at, and the
// current wall clock. Pure function, exercised by tests without a
// DB so the policy stays explicit and trivially auditable.
func nextCheckinStatus(current, action string, expiresAt, now time.Time) (string, error) {
    switch action {
    case "tap":
        if current == CheckinStatusAnswered {
            return CheckinStatusAnswered, nil // idempotent re-tap
        }
        if now.Before(expiresAt) && current == CheckinStatusPrompted {
            return CheckinStatusAnswered, nil
        }
        // expired / prompted-past-cap / late_answered all collapse here
        return CheckinStatusLateAnswered, nil
    case "expire":
        if current != CheckinStatusPrompted {
            return "", errors.New("can only expire a prompted row")
        }
        if now.Before(expiresAt) {
            return "", errors.New("expires_at not reached yet")
        }
        return CheckinStatusExpired, nil
    }
    return "", fmt.Errorf("unknown action %q", action)
}
```

Twenty-two lines. The entire state machine of the check-in feature. Every branch is one test case in the table-driven test.

{{< /details >}}

`nextCheckinStatus` is the canonical policy document. It is a pure function (no DB, no clock, no Telegram) and the table-driven test that exercises every transition runs in milliseconds. The DB-side `ExpireCheckin` and `SaveCheckinAnswer` re-encode the same rules in SQL with `FOR UPDATE`[^for-update] row locks and conditional UPDATEs, because the policy needs to be enforced atomically when a scheduler-expire and a user-tap race in the same millisecond.

That race is the kind of thing you do not catch in the first PR. PR #121 went through roughly six rounds of review-and-fix before settling. In order: a feature-flag gap (the check-in gate was triggering in the morning scheduler even when no webhook secret was configured, so users without check-in setup would see the morning report wait indefinitely for a tap that could never arrive); a race in the first version of `ExpireCheckin` between a two-statement read-then-update and an incoming tap (collapsed into a single conditional UPDATE in `7a50aa1`); a forgotten render of the expired-day note in the morning report (the i18n key was added but never plumbed into the cap-path, `983125f`); a stale-callback case where a tap on yesterday's prompt could mutate yesterday's answer row (rejected before save in `4e45212`); a lazy-init secrets mismatch where the scheduler gate read a different source than the registration path (`60e4a34`); and finally a partial-POST case in the admin settings endpoint that could accidentally `deleteWebhook` for env-backed installs.

Six rounds, on a feature that looks small in the schema. Each round was a different *kind* of bug. The race was about concurrency, the feature-flag was about boot-time integration, the expired-note was a product-copy contract that was easy to miss because it lived in the cap-path which is not the happy-path, the stale-callback was about idempotency semantics, the secrets mismatch was about lazy-init order, the partial-POST was about API surface. None of them were caught by the unit tests of the components in isolation. They surfaced in code review on the integration between components.

## The webhook security side

The check-in is the first feature in this project that accepts inbound HTTP from a third-party service (Telegram). Everything else is outbound: the server pulls from HealthKit-via-iOS-app, pushes Telegram messages, queries Gemini. Inbound webhooks have a different threat model.

Four layers of validation on the callback:

1. **Path secret.** The webhook URL is `/api/telegram/webhook/<secret>`, where `<secret>` is a per-tenant value rotated independently of the bot token. A leaked Telegram bot token does not give an attacker the webhook path.
2. **Header secret.** `X-Telegram-Bot-Api-Secret-Token` is set when registering the webhook and validated on every incoming request. This is Telegram's own anti-spoofing mechanism.
3. **`chat_id` lookup.** Even a valid Telegram-signed callback for the wrong chat is rejected. The handler walks the multi-tenant pool map looking for a tenant whose configured `TELEGRAM_CHAT_ID` matches the callback's chat. No match → reject.
4. **HTTPS-only at startup.** The server refuses to start if the webhook URL is not HTTPS. Telegram requires HTTPS for webhook delivery anyway, but a misconfigured deployment that fell back to HTTP would silently lose every callback. A startup guard surfaces the problem before the first morning prompt fires.

The combination of (1) and (3) is the load-bearing one. The path secret stops a generic attacker who scraped the webhook URL pattern. The `chat_id` lookup stops an attacker who somehow knows the path but does not know which tenant's `chat_id` corresponds to which path, and prevents an attacker who controls *any* Telegram account from impersonating an arbitrary tenant on a valid path.

Auto-registration on token changes (PR #122) handles the operational case where the bot token rotates. The server detects the change, calls Telegram's `setWebhook` with the new secret, and updates the registry. A failed registration surfaces in the admin UI as a coloured badge ("Webhook · failed · …") so a silent rotation does not silently break the check-in.

## Telegram-first, by design with room for dashboard

Before any code, the brainstorm doc compared three entry points:

- **A. Telegram-first.** One tap in the place where the morning report already arrives. The habit forms in the existing notification surface.
- **B. Dashboard-first.** Native widget on the dashboard hero. Requires daily dashboard opens, which I do not reliably do.
- **C. Hybrid MVP.** Both, with a shared backend and a single row per (tenant, date). Best long-term shape, biggest first PR.

The verdict from the brainstorm was "A for the first slice, designed so C is easy next." The schema reflects that. The primary key on `subjective_checkins` is `(date, source)`, not just `date`. The `source` enum currently has one value (`telegram`); a future `dashboard` source can co-exist on the same date without overwriting the Telegram answer. The pure transition helper is source-agnostic. The webhook handler does not assume Telegram is the only producer.

The reason the hybrid is not in the first PR is the principle from Part 7: a thing is allowed to exist as soon as it can exist usefully alone. The Telegram path is useful on its own (habit, gate, log, narrative). The dashboard path would be additive, not required. Shipping both in PR #1 would mean writing the dashboard surface before the storage layer had any production use to argue against its assumptions.

## The expired-note path

When the cap passes without an answer, the morning report sends anyway. The user is still entitled to their numbers. But the report carries a one-line note, localised per the tenant's `report_lang`:

> No morning check-in today. Tap "How do you feel?" in yesterday's prompt to log retroactively.

The note exists for two reasons. First, silence should not be invisible. The dashboard's coverage signal needs to reflect that today is uncalibrated for the subjective channel. Second, silence should not be punishment: the report still arrives, the numbers are still readable, the user's day is not held hostage to a tap.

This decision is the same shape as the data-quality gates in Parts 5 and 6. When sensor data is missing, the bank is not rendered; when stress coverage is insufficient, drain falls back to v2.0; when the check-in is expired, the report sends with a note. Each case has its own degraded-mode UI that admits the gap honestly instead of fabricating a placeholder.

`render checkin_expired_note in morning report cap-path` is the commit that landed this (`983125f`). It is also the commit that made me pause and confirm with myself that the note text was not editorial. Not "you missed it!", not "remember to check in tomorrow", just the bare fact that today is uncalibrated and a link back to the prompt. The point is the coverage signal, not the nudge.

## What is allowed to ask, what is not

A simple frame that came out of building this: the system is allowed to **ask** the user how they feel. The system is not allowed to **show that asking changed any of its other numbers.** Those are different actions with different methodological consequences.

Asking is a UX surface. It anchors a morning habit, it produces a coverage signal, it generates an audit trail for future narrative review. Asking is also a form of acknowledging that the user has perceptions the sensors cannot fully capture, that on the days the formulas say "readiness 73, you are fine" the user can still legitimately tap "Sick".

Showing-that-asking-changed-anything is calibration. Calibration against self-report is the closed loop Part 6 spent 2000 words explaining how to avoid. The system can ask without changing. The check-in row sits next to the day's numbers in the database, never inside their computation.

The temptation to bridge those ("the user said 'Sick', let's downgrade today's readiness from 73 to 50") is the largest hidden lever in a personal health system. I am not going to pull it. The numbers I show have to be the numbers the formulas produced. The user's tap is its own independent record. If they disagree, that disagreement is the interesting signal, and I want to keep it visible, not paper over it.

## What has not yet proven itself

Everything in this part is design and code. None of it is a verdict.

The check-in shipped recently enough that I do not have an honest production read on whether it earns its place. Five things are open, and each is a reason this part of the build log is not closed:

1. **Does the gating actually anchor the habit, or do I autopilot through it?** A morning where I tap "OK" without reading the question is no different from a morning with no check-in at all, except now there is a row in `subjective_checkins` claiming there was one. The signal degrades silently. I have no instrumented way to distinguish reflexive taps from considered ones yet, and probably will not. The user is the only sensor for that distinction.
2. **Do I tap honestly on the days that matter?** The Sick button is the one with the most downstream consequence (illness signature, AI recommendation pivots). It is also the button I am least likely to press on a marginal day where I am not sure. If I underuse it, the coverage signal is worse than nothing. It implies "not sick" when the truth was "uncertain".
3. **Does the four-category design hold up?** It is plausible that I end up using only two of the four buttons in practice, in which case the discrete scale is mostly decorative. Worse, it is plausible that "OK" eats "Meh" because tapping "Meh" feels self-pitying on a normal-low day. The asymmetry I designed for might collapse the wrong way.
4. **Is the late-answered pool actually useful?** I added the `late_answered` state because mixing post-hoc taps into the validation pool was clearly wrong. Whether the late-answered pool has *its own* analytical use, or whether it is just a slightly more polite version of "lost", will take months of accumulation to tell.
5. **Is the narrative-review claim real?** "In two years I will be able to look back at anomaly clusters and ask whether they correlated with 'sick' taps" assumes the taps will be a clean record. If the answers to (1)-(3) are bad, the record is corrupted and the future-archaeology argument falls apart.

The honest disposition is to keep observing for a quarter, then ask the same questions with data instead of guesses. The schema and the four states are deliberately set up so the answers can be queried without further code changes. `status`, `answer`, `prompted_at` and `answered_at` carry enough to compute response latency, distribution per button, and coverage over time.

If the data says "the check-in is mostly noise", the right move is to remove it, not to renegotiate the methodology until it looks useful. Part 6's refusal to tune β against weak signal applies here too: a feature has to clear its own floor before it earns the right to stay in the system. I am keeping a private todo to revisit this in the autumn.

## The actual lesson

This was the first feature in the build log where the architectural question was not "what should the formula compute" but "what should the user be able to say". Both questions are real and both have careful answers. They are different questions. And unlike the others in this series, the answer to this one is still being observed. The design exists, the code runs, the production data is just starting to accumulate.

The first article was about trusting the data. The second was about trusting the columns. The third was about trusting the source of each column. The fourth was about trusting the target. The fifth was about trusting the formula's silence. The sixth was about trusting the verdict. The seventh was about trusting the boundary between server and client. This one is about trusting the *question*. Being explicit about what the system is allowed to ask the user, what the answer is and is not for, and being honest that the answer to "is the question itself worth asking" is not yet in.

The build log catches up to the present here. The five sub-scores, the bank, the flags, the validation rubric, the thin iOS client, and the morning check-in are not finished. They are a system that will produce more data, surface more anomalies, and force more redesigns. The check-in in particular is at the bleeding edge of that list: the freshest piece, the least proven, and the most likely to be on probation when I revisit it in a few months. The point of writing the series was not to declare a destination. It was to lay out the order in which the system stopped being a pretty chart and started being something I trust to read first thing in the morning, and to be explicit about which pieces I trust because they have earned it and which pieces are still on trial.

[^inline-keyboard]: A Telegram message attachment: a grid of tap-able buttons rendered directly below the message body. When the user taps one, Telegram POSTs a callback to the bot's webhook with a payload identifying which button was tapped and on which message. Distinct from a "reply keyboard" (which replaces the user's keyboard); inline keyboards stay attached to a specific message.
[^callback-data]: The opaque string a Telegram bot attaches to each inline-keyboard button. When the user taps the button, this string is what Telegram sends back to the bot's webhook. Limited to 64 bytes per button. Typically encodes "what action and what context"; here `checkin:<answer>:<date>` is enough to identify which button was tapped and which day's prompt it belongs to.
[^webhook]: A pattern where a server registers an HTTPS URL with an external service and that service POSTs to the URL whenever an event occurs. Inverse of polling: the server is told when something happens instead of asking repeatedly. For Telegram bots, the alternative is "long polling" (the bot's client asks Telegram for updates on a loop). Webhooks are cheaper and lower-latency but require the server to have a public HTTPS endpoint.
[^idempotent]: A property of an operation: applying it once produces the same result as applying it any number of times. `UPDATE foo SET status='X' WHERE status='X'` is idempotent (no-op the second time). `UPDATE foo SET counter=counter+1` is not. Critical for safe retry logic in distributed systems, because the network can deliver the same request twice.
[^for-update]: A Postgres clause on `SELECT` that takes a row-level write lock on each returned row for the duration of the transaction. Other transactions that try to update the same rows block until the first transaction commits or rolls back. The standard tool for "read this row and then update it without anyone else racing in between". Without it, two concurrent transactions could each read the same `prompted` status and both transition it, producing inconsistent state.
