---
title: "Lambda Cold Starts: Choosing Between Provisioned Concurrency, SnapStart, and Code-Level Optimization"
date: 2026-11-09
draft: false
tags: ["aws", "lambda", "serverless", "performance", "cold-start", "provisioned-concurrency", "snapstart"]
---

## The Problem

Your Python Lambda API hums along at p99 of 800ms during steady traffic. Then a spike hits — a client drops a burst, Lambda scales a dozen new execution environments from zero, and your p99 suddenly reports **3.2 seconds**. The SLA says 1s. Someone is about to have a bad call with the customer.

You know the usual suspects for mitigation: Provisioned Concurrency, SnapStart, and "just write better init code." But they behave very differently depending on your runtime, your traffic, and your budget — and the official docs don't hand you a decision procedure. You can easily spend $45/month on capacity that buys nothing, or discover SnapStart doesn't even support your runtime after a full day of prototyping.

This post gives you the decision framework: given my **runtime, traffic pattern, latency target, and budget**, which approach should I use — and in what order?

## First, Understand What a Cold Start Actually Is

A Lambda cold start isn't one problem, it's a sequence:

1. **Runtime startup** — AWS spins up the sandbox and boots your runtime (Python interpreter, Node.js, JVM, etc.).
2. **Initialization** — Your `Handler` is built, module-level imports run, clients (DynamoDB, S3, HTTP pools) are constructed, and *your* initialization code runs.
3. **Handler invocation** — Your actual code executes.

The third part is roughly identical warm vs. cold. Cold start *extra* latency is phases 1 + 2 — and those are dominated by **your dependencies and init code**, not the handler.

That distinction drives everything below. Provisioned Concurrency eliminates phases 1 and 2. SnapStart eliminates *most* of phase 1 and *all* of phase 2, at the cost of a restore step. Code-level work shrinks phase 2 specifically. So the right tool depends on *where your cold-start time actually lives* — and that's why **monitoring comes first**.

## The Three Mitigations, Honest Trade-Offs

### Provisioned Concurrency — the brute-force, works-everywhere option

AWS keeps N execution environments warm 24/7, before requests arrive. Full stop, zero cold start for those N instances.

- **Works for every runtime** — Python included.
- **Cost: 10× the on-demand rate, whether or not it's used.** You're charged for provisioned capacity *held* continuously, so a 128MB function with 100 provisioned instances runs roughly **$45/month even at zero traffic**. It's the "keep the parking lot full for the car that arrives at 3am" model.
- **The catch:** it's the *only* first-class option for Python, and it's expensive when idle. Provisioning for a burst that happens once a day is pure waste.

### SnapStart — the free one, but picky about runtime and latency

AWS snapshots your function's initialization and restores it when a new environment spins up. Instead of re-running your imports and client construction, it deserializes a frozen image.

- **Supported: Java (Corretto), .NET, Node.js (18+), and Go (since Oct 2024).** **Python is NOT supported** — a hard limitation as of this writing, no matter how much you wish otherwise.
- **Zero idle cost** — you pay only the normal on-demand price, with no monthly fee whether or not it's used.
- **The catch: it adds a snapshot-restore step to the cold path.** That restore is ~200–500ms on the first invoke after a scale-from-zero event. If your SLA is 400ms p99 and a cold start would push past it, SnapStart's restore itself can blow the budget. And it does nothing for already-warm instances — it only helps the *cold* path.

### Code-level optimization — free, but has a hard ceiling

Shrink phase 2: lazy-load heavy imports inside the handler instead of at module level, drop unnecessary dependencies, use a smaller base image (Lambda container image with distroless/minimal), re-use connection clients across invocations.

- **Free.** No idle cost.
- **Diminishing returns are real.** You can't optimize away the runtime boot (phase 1). If your 800ms→3.2s spike is mostly Python interpreter + framework startup, heroic effort might shave 100–300ms, not 2.4 seconds. For a 1s SLA, code-level alone usually isn't enough — but it *should* always be the first step, because it shrinks whatever you'd otherwise have to pay to provision.

## Step 1: Measure Before You Spend (No Exceptions)

Actually, the very first step is monitoring — before any of the above. You'd be amazed how often teams provision capacity for a cold-start problem that's actually a dry connection pool or a slow downstream call.

- **Turn on AWS X-Ray** tracing on the function. X-Ray segments split the cold path so you can see `Init` vs. `Invoke` time directly.
- **CloudWatch Logs** — look for the `REPORT RequestId... Duration... Billed Duration... Init Duration: 1432.34 ms` line on cold invocations. That `Init Duration` number is your whole decision input: it tells you *exactly* how much cold-start time lives in phases 1+2.
- **Instrument cold starts** with a `ColdStart: true` attribute (log-based or a custom metric) so you can measure **cold-start frequency and its contribution to p99** separately from warm latency.

The two numbers you need before deciding anything:

1. **`Init Duration` distribution** — the actual cold-start cost per invocation.
2. **Cold-start frequency** — how often a new environment is actually spun up. A function at steady concurrency 3 with plenty of warm instances has a *different* cold-start problem than one that scales to zero every night.

If init is 200ms, you probably don't need Provisioned Concurrency at all — code-level + monitoring may get you under SLA for free. If init is 2.4s (your case), provisioning is unavoidable, and the question becomes *how much*.

## Step 2: The Runtime × Language Matrix

This decides which options are even on the table. It's the fastest filter:

| Runtime | SnapStart | Provisioned Concurrency | Notes |
|---|---|---|---|
| **Python** | ❌ Not supported | ✅ Only real option | Code-level is always the pre-step |
| **Java** | ✅ Great fit | ✅ | |
| **.NET** | ✅ | ✅ | |
| **Node.js** | ✅ (18+) | ✅ | |
| **Go** | ✅ (since 2024) | ✅ | |

If you're on Python (this team's case), the matrix *already* answers half the question: **SnapStart is off the table.** You're choosing between Provisioned Concurrency and code-level — no SnapStart to weigh. That's liberating, honestly.

## Step 3: Traffic Pattern Analysis

Now map your runtime to your traffic curve. Three canonical patterns:

### Steady baseline + occasional spikes — provision the baseline
You run at concurrency ~10 permanently and spike to 50 at peak. **Provision the baseline (say 12), let spikes burst to on-demand.** At 128MB, that's roughly 12 × $0.45 ≈ **$5–6/month** of idle cost — cheap insurance that swallows your normal load with zero cold starts, while the rare spike absorbs an on-demand cold start you accept (or shave with code-level).

### Spiky / bursty — SnapStart wins
Traffic arrives in unpredictable waves from near-zero. Provisioning the *peak* would be absurdly expensive (all that idle capacity for a 5-minute burst). SnapStart is the clear winner *for supported runtimes*: no idle cost, and it turns a 2.4s cold init into ~300ms restore. For Python here, your only answer in the spiky case is a *small* provisioned buffer + code-level, and accepting elevated p99 during extreme bursts.

### Always warm / high sustained traffic — Provisioned is overkill
If you're basically never cold (warm instances always available), adding Provisioned Concurrency just burns money on capacity you're not using. Measure first — you may not even have a cold-start problem worth solving.

## Step 4: Latency Target vs. Budget — Sizing Provisioned Concurrency

Here's the concrete math that turns "how much?" into a number. Your numbers: warm p99 800ms, cold p99 3.2s (≈2.4s cold-start penalty), SLA 1s.

1. **Goal:** get cold p99 ≤ 1s. You can tolerate ~200ms of cold-start overhead (800ms warm + 200ms = 1s). Your cold phase is 2.4s — way over — so *any* cold invocation likely violates the SLA. **You effectively need to eliminate cold starts entirely on the critical path.**
2. **Find your concurrency floor.** From CloudWatch, look at your **max concurrent executions at the 99th percentile over the last 30 days** (or from X-Ray concurrency). That's the capacity you must keep warm so that under normal peaks, requests always hit a warm instance.
3. **Provision that number** (plus a small % headroom), with provisioning auto-scaling so it grows and shrinks with demand. Cost ≈ provisioned instances × memory(GB) × provisioned rate (~10× on-demand) × 8,760h × 3,600s. For 128MB (0.125GB) × 25 instances that's roughly **~$40/month** — the price of SLA compliance, and less than one engineer-hour; usually worth it.

So the decision for this team: **provision enough to cover baseline + normal peaks, sized from real max-concurrency data, and accept that extreme tail bursts may still cold-start.** If even *that* is unacceptable and budget is tight, code-level work reduces the number of instances you need warm by shrinking init.

## Step 5: Hybrid & Hardening Patterns

Don't think of these as mutually exclusive — a good system usually layers them:

- **Provisioned for hot paths, code-level for everything else.** Your critical API route gets provisioned; a rarely-hit admin endpoint just takes the cold start.
- **Provisioned concurrency auto-scaling.** Configure the warm count to grow and shrink with demand rather than staying fixed, so you don't over-provision for off-peak.
- **Scheduled warmers (function warming) as a *cheap stopgap*.** A CloudWatch/EventBridge rule invoking the function every few minutes keeps environments warm for close to normal invocation cost — *far* cheaper than Provisioned Concurrency at low, variable concurrency. Use it while you determine your concurrency floor. Caveat: it doesn't guarantee warmth under true scale-from-zero bursts, and it's a hack — a stopgap, not an architecture.
- **SnapStart + Provisioned** (for non-Python): keep a tiny provisioned baseline for guaranteed-warm requests *and* SnapStart for the else-case, so even cold spikes are fast.

## The Decision Tree (Your Shortcut)

Given the team's actual situation — **Python, spiky-ish traffic, SLA 1s**:

> **If runtime is Python** → Provisioned Concurrency is the only lever that meaningfully removes cold starts. SnapStart isn't available. Size it from real max-concurrency data, use auto-scaling, and do code-level first to shrink what you have to provision.
>
> **If runtime is Java/Node/.NET/Go and traffic is spiky** → SnapStart. Zero idle cost, eliminates init, and restore latency (~200–500ms) is usually acceptable.
>
> **If runtime supports SnapStart but traffic is steady / latency target is ultra-tight** → Provisioned Concurrency (SnapStart's restore can exceed a 400ms SLA).
>
> **If you don't know your traffic pattern** → code-level optimization + X-Ray/CloudWatch monitoring for 1–2 weeks first. Measure `Init Duration` and concurrency, then make the call with data instead of guesses.

## What Would I Actually Do in Your Shoes?

For this Python API specifically:

1. **This week (free):** Turn on X-Ray, add a `ColdStart` custom metric, and capture `Init Duration` on a sample of cold starts. Apply the easy code-level wins — lazy-load heavy deps, reuse clients, slim the base image.
2. **Then:** Pull 30 days of max concurrent executions at the 99th percentile. That's your provisioning target.
3. **Provision that number** with provisioning auto-scaling turned on, sized to your baseline + normal peaks. Expect roughly **tens of dollars per month** at 128MB — and confirm with your own bill the week after.
4. **Keep monitoring.** If the tail is still bad, code-level harder. If it's still expensive, reconsider whether the extreme tail genuinely needs the SLA.

The goal isn't "pick the one right tool." It's "know your runtime ceiling, measure your real cold cost, size your capacity from data, and only pay for what the SLA actually demands." Do that and you'll spend roughly half of what most teams burn guessing.

Need a Terraform snippet or CloudWatch metric filter for the cold-start measurement? Drop a comment and I'll add it.
