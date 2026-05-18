---
title: "Health Dashboard"
description: "An 8-part build log: how a personal Apple Health receiver grew into a methodologically honest dashboard with explicit ineligibility, multi-model design review, and a server that refuses to fabricate numbers."
cover: "cover-light.png"
---

A self-hosted health-tracking system built around one rule: refuse to fabricate numbers when the inputs are missing.

The series follows the order in which the system stopped being a pretty chart and started being something I trust to read first thing in the morning. Each part is one layer of trust earned: the data, the columns, the source behind each column, the target, the formula's silence, the verdict, the boundary between server and client, and finally the question the system is allowed to ask the user.

Read in order; cross-references between parts assume you have.

## What this is not

I am **not** a professional software engineer. I am **not** a physiologist or a medical researcher. This series is the build log of a personal pet project, not a peer-reviewed methodology paper.

Most of what is described here came together through reading on the open internet, comparing notes against published wearable-validation research where I could find it, and iterating with several LLMs (Claude, Gemini, ChatGPT, Manus, Grok, GLM) as sounding boards for design and code review. Every formula here has been argued with at least three of them. None of that makes it correct; it makes it honestly assembled.

If you spot a methodological mistake, a sensor misinterpretation, or a more grounded reference in the wearable-validation literature than what I have cited, I would genuinely like to hear about it. The whole project exists in two open-source repositories below and reads its own data; corrections that change the math are welcome.

## Source

- **Server** (Go, Postgres, Telegram/Gemini integrations): [Dzarlax-AI/health_dashboard](https://github.com/Dzarlax-AI/health_dashboard)
- **iOS client** (Swift 6, SwiftUI, HealthKit): [Dzarlax-AI/health-sync](https://github.com/Dzarlax-AI/health-sync)

Both repos are MIT-licensed. The server is self-hostable; the iOS client builds in Xcode 26+ and pairs with any instance of the server that the device can reach over HTTPS.
