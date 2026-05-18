---
title: "Health Dashboard, Part 6: Measuring Stress Without Lying to Myself"
description: "Part 6 of my Health Dashboard build log: why a single 'stress score' is structurally dishonest, what the available signals can and cannot say, how the canonical formula replaced the obvious one after a six-LLM review, and the rubric that stops the system from auto-tuning itself into noise."
date: 2026-05-18
lastmod: 2026-05-18
draft: false
tags: ["build-logs", "pet-projects", "systems-automation"]
series: "health-dashboard"
series_part: 6
cover: ""
toc: true
---

In Part 5 the EnergyBank redesign left a placeholder: `drain = α · active_kcal + β · sustained_hr_load`, with `β = 0` until a validation rubric per-user returns `validated`. That term, the autonomic-load drain, is the v2.2 piece. It is also the one I most wanted to be careful with, because "stress" is the single most over-claimed metric in consumer health products.

This part is about what stress actually means physiologically, what my data can and cannot say about it, why the obvious formula was wrong, and what the rubric is for.

## What "stress" is not

Most consumer wearables ship a "stress score". Garmin Body Battery, Whoop strain, Fitbit Stress Management. They are convenient and, in the words of Marco Altini (PhD physiology, HRV4Training), [made-up scores](https://the5krunner.com/2025/08/19/garmin-body-battery-slammed-indirectly-by-altini-made-up-scores/): closed algorithms, no peer review, outputs that drift between physiologically similar days. The wearable-validation literature is unanimous about this:

> Raw HRV[^hrv] and RHR showed significant associations with validated stress measures. Proprietary composite algorithms lack transparency and independent validation. Composite scores add questionable value over raw signals.

The methodological problem is not the company secrecy. It is that "stress" is not one state. It is a spectrum of biological signatures with different timescales and different markers:

| Type | Marker pattern | Timescale | In my data? |
|---|---|---|---|
| Acute sympathetic spike (startle, caffeine, pre-deploy nerves) | HR↑, HRV↓, EDA↑ | seconds–minutes | HR yes; HRV partial; EDA no |
| Sustained autonomic load (worried about the dog all day) | HR shifted +5–10 bpm, depressed HRV, faster respiratory rate | hours | the v2.2 target |
| Allostatic load / chronic stress (burnout, sustained sleep debt) | baseline RHR drift up, HRV drift down | weeks–months | yes via 30–90d trends |
| Illness response (flu, COVID prodrome) | RHR↑5–10 bpm, wrist temp↑0.3°C, HRV↓↓ | days | partial (wrist_temperature only since 2025-10) |
| Recovery debt (yesterday's hard session) | overnight HRV↓, slight RHR↑ | next morning | yes |

Collapsing these into one number obscures which process is loaded. A 67/100 "stress score" tells me nothing about whether I am anxious, fighting a virus, or sitting on three nights of bad sleep. The actions that follow are different in each case.

So the v2.2 design starts from a constraint: **multiple signals, multiple flags, no composite score.** Each flag routes to a different downstream behaviour.

## What the data has, and what it does not

Sample counts in the database as of 2026-05-12:

```text
heart_rate              1 916 248 samples   2015-01-11 → present   high-density (every few min)
respiratory_rate           35 954 samples   2021-09-24 → present   ~daily aggregates
blood_oxygen_saturation    32 830 samples   2020-12-31 → present
heart_rate_variability     13 373 samples   2018-04-29 → present   sparse (~5/day, event-triggered)
wrist_temperature             191 samples   2025-10-08 → present   nightly only, Apple Watch S8+
```

Three signals that are not there:

- **EDA / GSR**[^eda-gsr]. Apple Watch does not measure it. Fitbit Sense and Whoop 4.0+ do.
- **Cortisol.** No non-invasive consumer wearable path exists in 2026.
- **Continuous HRV.** Apple measures HRV during specific events (Breathe sessions, after exercise, occasional nocturnal). It does not expose a continuous overnight aggregate even though the sensor has the data. Whoop and Oura do.

The practical implication is that **heart rate is the only dense continuous signal.** The methodology has to lean on HR primarily, with HRV / respiratory rate / temperature as confirmatory channels when they happen to be available. Any design that hinges on HRV being plentiful is a design that does not run on my dataset.

## Personal baselines, with a variance floor

The same personal-baseline convention from Part 5 (rolling 30-day, never persisted, `cold` / `warmup` / `steady` state machine) carries over here. The non-obvious addition is on variance:

```text
sd_hr_awake[d] = max(MAD-based SD[^mad-sd] of last 30 days, SD_FLOOR_HR)
```

`SD_FLOOR_HR = 3.0 bpm`. Without a floor, users with very stable daytime HR can have an MAD-derived SD as low as 1.5–2 bpm. Then a routinely-busier-than-usual day at +5 bpm reads as z ≈ 2.5, a false "anomaly". The 3-bpm floor was chosen empirically on my own 90-day data; N=1 is a documented limitation that the plan revisits once a cohort of more than three users runs the same methodology.

## Three blockers before β can go non-zero

The plan flags three pre-conditions that have to land before β multiplies anything:

**1. `daily_scores.rhr_avg` is not RHR.** It is a per-day average of all heart rate samples, not a resting baseline. v2.2 must not consume this column as `RHR_baseline`. Either the column gets fixed to compute from overnight low-HR windows, or a separate `baseline_hr_overnight` lands as median HR over the last 3 hours of the main sleep segment (with `03:00–06:00` as a fallback only when sleep data is imputed/stale). Pick one path, do not run both in parallel.

**2. Awake window has to derive from `sleep_analysis`, not from a fixed clock.** The first version of v2.2 used `07:00–22:00` as a stopgap. That breaks on siestas, night-shift, jet-lag, and "asleep at 02:00" late nights. The fix is an algorithm that picks the longest continuous asleep segment whose midpoint falls within ±6 hours of local midnight for date `d`, sets wake-time to its end, and sleep-onset to the start of the next qualifying segment. The fallback to the fixed window only fires when no main sleep is found, and emits an `imputed_awake_window` flag.

The existing `internal/storage/typical_wake.go::GetTypicalWakeTime(days)` helper returns the *typical* (averaged) wake time. Wrong shape for this use case; that helper legitimately wants the average for `MorningCapTime`. A new `WakeTimeForDate(date)` is needed.

**3. Coverage gate.** A day with fewer than 8 hours of HR-covered awake time (watch off charger, sync gap) must not compute `sustained_hr_load`. Without this, days when the watch is off between 10:00 and 16:00 systematically under-drain. The gate emits a `stale_stress` flag and falls back to v2.0 kcal-only drain for that day.

Each of these is a detail that does not change the headline formula. They do, however, determine whether the formula is computing what it claims to compute.

## The formula that almost shipped

The first v2.2 draft proposed something simple:

```text
drain_hr = β · ∫ max(0, HR(t) − RHR) dt   # bpm·hours
```

Integrate the area where HR runs above RHR. Multiply by β. Add to drain. β ≈ 0.12 against bpm·hours[^bpm-hours], target ~10–15 drain points on a "stressful sedentary day". It looked clean. It is the variant most likely to be implemented by someone moving fast.

On 2026-05-12 I did something I had not done before on this project. I pasted the v2.2 draft, the canonical `STRESS_MEASUREMENT.md`, and the surrounding context into **six different LLMs** in succession (Gemini Flash, ChatGPT, Manus, Grok, Gemini Pro, and GLM 5.1) and asked each to review independently. Then I pasted each review back into my normal Claude Code session and asked it to synthesize: where do the six agree, where do they conflict, what is each suggesting that the others are not.

The synthesis was structured. Six points where all reviewers agreed, regardless of model family: that `daily_scores.rhr_avg` was misnamed and unsafe to consume; that the awake-window default of 07:00–22:00 broke on shift work and jet-lag; that illness should not inflate drain; that β as a constant had a unit-mismatch between the two documents; that the draft and the canonical document were in conflict; that the asymptotic restore mechanism and the bootstrap convergence were strong and should not be touched. Then a three-way fork on the drain formula itself, with different reviewers preferring different variants: raw `HR − RHR` in bpm·hours (Gemini Flash), daily-averaged z-shift (an earlier `STRESS_MEASUREMENT` draft), or hourly sustained z-load (Manus).

The synthesis converged on three concrete problems with the v2.2 draft shape:

**It leans on `daily_scores.rhr_avg`, which is not RHR.** Blocker (1) above. The formula was about to consume a column whose meaning had silently drifted from what the name suggested.

**It mixes physical and postural components with stress signal into a single raw bpm delta.** Standing HR runs ~10 bpm above lying HR. A day with more standing produces apparent overshoot that is not autonomic load. The integral cannot tell those apart.

**At daily-average granularity it loses the temporal structure of the day.** A 4-hour sustained shift and a 1-hour sharp spike followed by 14 normal hours produce the same daily-average overshoot. They are physiologically different events.

The draft is preserved in the repo as `ENERGY_BANK_V2_2_DRAFT.md` with a banner that says SUPERSEDED. I keep these around for the same reason I keep the Phase 1 feasibility verdicts: a formula I rejected today will look reasonable again in three months when I have forgotten the specific reason it was wrong.

A note on the multi-model review itself, because it is unusual enough to call out. After the fifth review the suggestions started repeating; the sixth (GLM 5.1) added four genuinely new actionable points and after that I stopped, because each next pass yielded one nitpick and a lot of validating text. The pattern is real enough that I have it written down as a heuristic: 3–4 model reviews is a useful sweep, 5–6 is diminishing returns, more than that is bikeshedding. The right next step after diminishing returns is implementation or domain-expert outreach, not another model.

## The formula that replaced it

The canonical v2.2 formula is **hourly sustained z-load over a personal awake baseline**:

```text
sustained_hr_load[d] = Σ_h max(0, hr_z_hour[h] − Z_THRESHOLD)
                       for h ∈ awake_window[d]
                       only if hr_coverage_hours ≥ MIN_COVERAGE

hr_z_hour[h] = (hour_hr[h] − baseline_hr_awake[d])
             / sd_hr_awake[d]    # MAD-based, floored

hour_hr[h] = median of 5-min minima within hour h
```

Each piece of this formula fixes one of the three review findings:

**Median of 5-minute minima per hour, not means.** Standing-up minutes are captured by the maximum, so the median of minima keeps the autonomic floor of each hour. Postural noise stops looking like stress.

**Z-shift over a personal baseline, not raw bpm.** A bpm number means different things at different fitness levels. The z-shift normalises out the absolute baseline, and the MAD floor (3 bpm) keeps the z-score from exploding on users with naturally narrow HR distributions.

**Hourly integral, not daily average.** A day-average would assign the same drain to four hours of sustained +10 bpm and to one short spike plus fourteen normal hours. The hourly integral preserves temporal structure: only the hours that actually deviated count. The starting `Z_THRESHOLD = 0.5` is a setting (`settings.energy.z_threshold`), not a hard-coded constant. For cognitively-demanding desk work, HR can sit at z = 0.5–0.8 systematically without being autonomic stress, and tuning that knob is a calibration step, not a code change.

Two values land in `components` JSONB for transparency. Drain consumes only `sustained_hr_load_z`, but the audit trail also stores `hr_overshoot_bpm_hours = Σ max(0, hour_hr[h] − baseline_hr_awake)` in raw units. The second number is the one that can be quoted in a user-facing sentence: "HR ran ~8 bpm above your normal for ~4 hours." The two are correlated but not redundant.

## Stratified flags, not a composite

Five flags fall out of the z-scored channels, each pointing at a different physiological process:

```text
acute_stress[d]       = exists window <2h where hr_z_hour > +2
sustained_load[d]     = hr_z_hour > +1 sustained ≥4h consecutive
illness_signature[d]  = (temp_shift > +1) AND (resp_shift > +1) AND (hrv_drop > +1)
recovery_debt[d]      = overnight (hrv_drop > +1) AND (rhr_shift > +0.5)
parasympathetic_rebound[d] = (hr_shift > +1) AND (hrv_drop < −1)
```

Each routes to a different layer of the system:

- **`acute_stress`** → no action. Transient, no point telling the user.
- **`sustained_load`** → feeds EnergyBank drain through the formula above.
- **`illness_signature`** → does **not** feed drain. The HR rise the flag detected is already in `sustained_hr_load`; adding an illness multiplier would double-count. The flag routes to the verdict layer: overrides the verdict text ("body fighting infection, rest aligns with this") and suppresses any AI "push harder" recommendation.
- **`recovery_debt`** → factors into next-day readiness baseline (Part 4 territory).
- **`parasympathetic_rebound`** → interpretation flag only. HR up and HRV up is real physiology: vagal rebound[^vagal-rebound] after sustained stress or heavy training. The HR rise was real autonomic expenditure and stays in drain; the flag adds context for the verdict narrative ("autonomic state mixed, HR high, HRV high, likely recovery phase").

The split between drain and verdict matters. Drain is the physiological cost of the day. Verdict is what the user should do tomorrow given the cost and the trend. Mixing them inflates drain artificially on sick days (the HR rise the illness flag detected is already in the drain term).

## Why no self-report in validation

The rule that ended up in this document ("subjective input cannot validate the stress formula") did not arrive as a decree. It arrived in roughly five minutes of back-and-forth.

The original v2.2 validation plan included a narrow self-report. The reasoning, when I first sketched it with my assistant, was that a free-form 1–5 daily score would be noisy but a *targeted* anchor would not be: one button in Telegram for "today was genuinely hard," another for "today was unusually calm," used maybe a few times a month. The pitch was that 5–10 high-confidence anchor points should weigh more in Pearson r than 28 lukewarm threes from a daily slider. It looked like a clean compromise between zero self-report and a noisy daily diary.

I read this draft and told the assistant, in roughly these words: *"I don't really like asking the user. They lie."*

The assistant's first response was to enumerate other anchors that did not rely on the user typing anything: calendar events ("flight," "interview," "doctor"), wrist-temperature spikes (confirmed illness), HealthKit-recorded workouts (confirmed training load), next-morning HRV (autonomic residual the user does not consciously control), sleep-onset latency (physiological response not under conscious editing). The argument: if you cannot trust people to rate themselves honestly, use the channels the data already produces.

I told the assistant the calendar idea did not work. Apple Calendar is not exported by HAE, and adding a Google Calendar integration just for stress calibration was the wrong dependency for a health-data project. That removed two of the five anchors. The remaining three are the ones in `STRESS_MEASUREMENT.md` today: next-morning HRV residual as the primary channel, RHR shift as secondary cross-check, sleep architecture as tertiary.

So the no-self-report rule was not an a-priori position. It was the outcome of a short conversation that started from "let's add a narrow self-report button" and ended at "let's use only the signals the user is not consciously generating." The mechanism that survived is now four objective channels and a decision rubric.

The reasons it survived are worth keeping explicit:

Subjective daily stress logs are methodologically poisoned *as a calibration signal*. People forget. They default to "3". They rationalise post-hoc: "there was a deadline today, must have been stressed, log 4." That is textbook confirmation bias and it inflates the Pearson r[^pearson-r] in the exact direction the formula already points. A formula that auto-tunes itself against a self-report that auto-tunes itself against the formula is a closed loop with no external referent.

That rule applies to **validation of physiological formulas**. It does not mean subjective input has no place in the system at all. It means subjective input does not get to vote on what β should be. Part 8 picks up the question of where subjective input does belong: gating the morning report, anchoring the habit, marking days for narrative review later. None of those uses calibrate the formula. The check-in row and the stress-validation rubric live in the same database but never look at each other.

The validation harness uses only objective signals already in the database. No user input path. Four channels, ranked by signal quality:

1. **Next-morning HRV residual (primary).** Pearson(`sustained_hr_load[d]`, `overnight_HRV[d+1]`) over a 30-day rolling window. HRV the next morning is not consciously controllable. It can be influenced by behaviour (alcohol, late training) but the user is not writing a number down, so post-hoc rationalisation does not bend it. Expected: `|r| ≥ 0.3` with **negative** sign (high stress load → depressed next-morning HRV) for the formula to be validated for this user.
2. **Next-morning RHR shift (secondary).** Cross-check for channel 1. Caveat: RHR can sign-flip physiologically. Vagal rebound after heavy load means *lower* next-morning RHR. Used as agreement check, not standalone pass criterion.
3. **Sleep architecture degradation (tertiary).** Daytime autonomic load predicts longer sleep_onset_latency, more sleep_awake, reduced deep_pct in the first third of the night. Three sub-correlations, vote on sign agreement.
4. **Test-retest stability (sanity).** Run the same date through the formula on day d+1 vs d+7. `sustained_hr_load` should differ by ≤1 z-load unit; bank by ≤2 points. Drift larger than that means inputs are non-settled or the formula is overfitting recent context. Pinned in a Go test (`TestBankConvergence`) so the convergence claim does not silently regress on tuning.

Channel 3 has an Apple-Watch caveat baked in: Apple Watch is known to over-detect deep sleep (it conflates immobility with slow-wave sleep). For nights where the picked source is Apple Watch, `deep_pct_first_third` is downweighted and only `sleep_onset_latency` and `sleep_awake` count toward the channel-3 vote. For nights with Oura or another research-grade device picked, all three sub-signals vote. This prevents a known sensor bias from rejecting valid calibrations.

## The rubric, and the refusal to tune

The validation rubric is run by `/api/admin/stress-validation`. No request body, no CSV upload. Pure read over existing tables. Idempotent.

| Channel 1 (HRV) | Channels 2+3 agreement | Action |
|---|---|---|
| `r ≤ −0.3` | At least one of {RHR, sleep} agrees in sign | **Validated**, β may be tuned per user |
| `−0.3 < r < −0.1` | At least two channels agree in sign | **Weak signal**, β stays at placeholder, flag `calibration_weak`, recheck weekly |
| `\|r\| < 0.1` on channel 1 | n/a | **Inconclusive**, likely HRV sparsity; β stays at 0, `data_quality_warn` |
| `r > 0` on channel 1 | n/a | **Wrong-direction**, formula is not capturing this user's physiology; β suppressed, escalate to manual review |

Three guard rails matter inside this table:

**HRV sparsity preflight.** Channel 1 requires a minimum sample density. Apple Watch HRV is event-triggered. A user without a daily breathing habit may produce 0–2 overnight samples per week. If `count(overnight HRV samples)` in the 30-day window is below 15, channel 1 is marked `cold` and the rubric falls back to channels 2+3+4 with a 2-of-3 agreement requirement. Without this guard, computing Pearson r on three or four points produces statistical noise that the rubric would misinterpret as signal.

**Disagreement override.** Channel 1 alone is not sufficient. If channel 1 says `r ≤ −0.3` but both channel 2 (RHR) and channel 3 (sleep architecture) disagree in sign, the verdict is downgraded to `weak` regardless of r magnitude. A single-channel result with two contradicting cross-checks is more likely overfitting or artefact than real physiology.

**Wrong-direction suppression.** `r > 0` is treated as "the formula is measuring the wrong thing for this user", not as "tune harder until r goes negative". The wrong-direction case escalates to manual review.

The point of the rubric is that the system **refuses to calibrate silently** when the signal is weak or self-contradictory. Default behaviour is `β_effective = 0`, with `sustained_hr_load_z` still computed into `components` for audit. Flipping `settings.energy.stress_drain_enabled = true` only happens after the rubric returns `validated`.

## What the components JSONB lets me prove later

The v2.2 design ships two values in `components` even though drain reads only one of them: `sustained_hr_load_z` (the canonical input) and `hr_overshoot_bpm_hours` (the discarded-draft input, kept for human-readable audit and for cohort comparison). Both stay until cohort validation shows one suffices.

Six months from now, when a third or fourth tenant has been running long enough to produce a usable distribution, the question "should the canonical signal have been z-shift or bpm·hours?" will be answerable from data already on disk. The current architecture does not commit to that answer; it commits to keeping both numbers honest.

That is what "no opaque score" means in practice. Not the absence of a number on the dashboard. There is a number, and it goes up and down. The point is the presence, behind every number, of an audit trail that another reader could rederive.

## What is missing from this article

Two pieces of unfinished business worth surfacing rather than glossing over.

**Observability of the formula in production.** The documents describe how to compute `sustained_hr_load`, validate it, and gate it, but say almost nothing about what to monitor once β goes non-zero. Without counters for coverage-gate skips, per-channel calibration states, the last rubric-run Pearson r, and the flag-fired distribution, one cannot distinguish "the formula is correctly quiet because this user has no stress days" from "the formula is silently dropping most days at the coverage gate." The spec is in the backlog as its own task and is not yet in production, because β is not on for anyone.

**Domain-expert outreach.** This article cites Marco Altini ([HRV4Training](https://hrv4training.com)) as the physiologist whose critique anchors the "no opaque scores" position. The natural next step after a six-LLM review is to send the canonical formula to an actual physiologist with a sharp question: does an hourly z-load against a MAD-based personal awake baseline make sense as a drain proxy, given the available signals? I have a Todoist task to do this and have not done it. The reason is partly cowardice and partly that the question gets sharper once β is provisionally calibrated on real data.

## The actual lesson

Stress is the metric most likely to be wrong on a consumer wearable and the metric I most want to be right on this one. The redesign exists to refuse three failure modes that are common in the space: collapsing a physiological spectrum into a single score, tuning against self-report, and silently producing a number when the inputs are not there.

The previous article was about trusting the formula's silence. This one is about trusting the *verdict*. Knowing what each flag means, what it routes to, and what evidence the system has the right to claim.

So far the series has been about the server learning to have opinions: about ingestion, about columns, about targets, about formulas, and about the verdicts those formulas are allowed to produce. The next article picks up the question that follows from all of that. Once the server has earned the right to its opinions, what is the iOS client allowed to do? The short answer is: less than the temptation would suggest.

[^hrv]: Heart rate variability. The millisecond-level fluctuation in the intervals between consecutive heartbeats. Counter-intuitively, *more* variability is better: a steady metronome heart suggests sympathetic dominance (stress, illness, fatigue), while a heart that varies its tempo freely indicates parasympathetic recovery. HRV is the canonical autonomic-state marker in modern wearable research.
[^mad-sd]: Standard deviation estimated via median absolute deviation, rather than the classical formula (square root of mean squared deviation from the mean). The classical SD is heavily distorted by outliers; one extreme value can double it. MAD-based SD takes the median of absolute deviations from the median, then multiplies by 1.4826 to estimate SD under a normal distribution. Resistant to the kind of outlier days a wearable produces routinely (sensor errors, exceptional events).
[^vagal-rebound]: Post-stress autonomic state in which heart rate is still elevated *and* HRV is also elevated above baseline. The vagus nerve overshoots its compensatory braking after a period of sympathetic activation. Physiologically real, common after intense exercise or sustained mental stress, and easy to misclassify as either "still stressed" (HR high) or "fully recovered" (HRV high) if you only look at one signal.
[^pearson-r]: Pearson correlation coefficient. The standard measure of linear correlation between two numeric variables, ranging from −1 (perfect inverse correlation) through 0 (no linear relationship) to +1 (perfect positive correlation). For physiological data, |r| ≥ 0.3 is conventionally treated as "real signal worth acting on" given sample noise; |r| < 0.1 is "likely noise."
[^eda-gsr]: Electrodermal activity (also called galvanic skin response). Sweat-gland conductance measured by a small voltage across two electrodes on the skin. A direct sympathetic-nervous-system marker: sweat glands are innervated only by the sympathetic branch, so EDA tracks arousal without the parasympathetic dampening that complicates HR-based stress reads. Apple Watch lacks the sensor; Fitbit Sense, Whoop 4.0+ and research-grade chest straps have it.
[^bpm-hours]: A unit of integrated heart-rate overshoot: beats per minute multiplied by hours. If your HR sat at 80 bpm while your RHR baseline was 60 bpm, you accumulated 20 bpm·hours per hour of overshoot. Intuitively interpretable but methodologically problematic because it mixes physical and postural HR rises with autonomic stress, which is why the canonical formula moved to z-scored hourly load (see `[^mad-sd]`).
