---
title: Event Loop as Correctness Signal
tags: [async, typescript, node, observability, perf-hooks]
summary: For high-consequence Node services, treat event-loop delay and event-loop utilization as first-class correctness signals — not just performance metrics. A saturated loop means deadlines, timers, and cancellations stop being honored.
keywords: [event loop, perf_hooks, monitorEventLoopDelay, eventLoopUtilization, elu, latency, observability, correctness]
---

*Node's event-loop health metrics are documented as performance APIs. For high-consequence services, treat them as correctness signals — because when the loop is saturated, deadlines you wrote down stop being deadlines.*

# Event Loop as Correctness Signal

> **Rule:** every high-consequence Node service exports event-loop delay and utilization metrics from [`perf_hooks`](https://nodejs.org/api/perf_hooks.html), alerts on regressions, and treats sustained saturation as a correctness incident — not just a performance one. Node frames these APIs as observability primitives. The framing as a correctness signal is the standard's own characterization, justified because a fully blocked loop means timers, deadlines, heartbeats, and cancellations all stop firing.

Node provides two stable APIs that, taken together, tell you whether the event loop is keeping up:

1. [`perf_hooks.monitorEventLoopDelay()`](https://nodejs.org/api/perf_hooks.html#perf_hooksmonitoreventloopdelayoptions) — returns an `IntervalHistogram` of nanosecond-precision loop-delay samples.
2. [`performance.eventLoopUtilization()`](https://nodejs.org/api/perf_hooks.html#performanceeventlooputilizationutilization1-utilization2) — returns cumulative active/idle durations and a 0-to-1 ELU ratio.

Both are stability 2 — Stable. Both are first-party Node APIs. The framing of these as *correctness* signals (as opposed to merely *performance* signals) is an engineering interpretation, not Node's own language — but the interpretation rests directly on Node's own documentation of how the APIs work.

## Why these metrics matter for correctness

The Node docs are explicit about *why* event-loop delay is a meaningful signal:

> Using a timer to detect approximate event loop delay works because the execution of timers is tied specifically to the lifecycle of the libuv event loop. That is, a delay in the loop will cause a delay in the execution of the timer, and those delays are specifically what this API is intended to detect.

Read that carefully: **a delay in the loop causes a delay in the execution of every timer.** Which means:

- A `setTimeout(callback, 5000)` does not fire after 5 seconds. It fires after at least 5 seconds, plus the loop delay.
- An `AbortSignal.timeout(5000)` does not abort after 5 seconds. It aborts after at least 5 seconds, plus the loop delay.
- A `keep-alive` ping that says "send a heartbeat every 30 seconds" sends it every 30+δ seconds.
- A circuit breaker that says "give up after 1 second" gives up after 1+δ seconds.

If δ (loop delay) is large enough, the deadlines you wrote into the code are no longer the deadlines the runtime is honoring. The downstream system may have already given up. The user may have already retried. The rate limiter may already be open. The system is producing wrong-state outcomes, not slow ones.

## The ELU=1 example

The Node docs ship a worked example for `eventLoopUtilization`:

```ts
import { performance } from "node:perf_hooks";
import { spawnSync } from "node:child_process";

performance.eventLoopUtilization();           // sample 1
spawnSync("sleep", ["5"]);                    // synchronously blocks for 5s
const elu = performance.eventLoopUtilization();
console.log(elu.utilization);                 // 1
```

Node's docstring on the example:

> Although the CPU is mostly idle while running this script, the value of `utilization` is `1`. This is because the call to `child_process.spawnSync()` blocks the event loop from proceeding.

That single example justifies the entire correctness framing. ELU is not measuring CPU usage. It is measuring whether the event loop is making progress. A value of 1 means the loop has been completely starved, regardless of whether anything was actually computing.

## What to instrument

For a critical-path Node service, the minimum signal set:

```ts
import { monitorEventLoopDelay, performance } from "node:perf_hooks";

const histogram = monitorEventLoopDelay({ resolution: 20 }); // 20ms resolution
histogram.enable();

setInterval(() => {
  // p50, p99 in nanoseconds — emit as metrics
  emitMetric("event_loop_delay_p50_ns", histogram.percentile(50));
  emitMetric("event_loop_delay_p99_ns", histogram.percentile(99));
  emitMetric("event_loop_delay_max_ns", histogram.max);
  histogram.reset();

  // ELU between samples
  const elu = performance.eventLoopUtilization();
  emitMetric("event_loop_utilization", elu.utilization);
}, 10_000);
```

The metrics belong on the same dashboard as the SLO — they are *predictors* of SLO breaches. A p99 delay climbing into the tens of milliseconds is a future SLO breach you can see before it happens.

## Alerting thresholds

Static thresholds are project-specific (a 100ms delay matters less for a 5-minute batch job than for a 10ms-SLA real-time API). The discipline is independent of the threshold:

- **The SLO defines the deadline budget.** (See [[Engineering Philosophy/Principles/Error Budgets and Reliability Policy]].)
- **The event-loop delay budget is a fraction of that.** A handler that has 50ms to respond and runs through 3 awaited calls cannot tolerate a 30ms loop delay per await without missing the 50ms total.
- **Alert on sustained breach, not isolated spikes.** Single high samples are common (GC, JIT). Sustained high samples are an incident.

When the alert fires, the response is the same as any SLO breach — see [[Engineering Philosophy/Principles/Error Budgets and Reliability Policy]] and [[Engineering Philosophy/Principles/Staged Canary Deployment]].

## What causes loop saturation in practice

Common culprits, in rough order of frequency:

1. **Synchronous filesystem, DNS, crypto, compression in the request path.** Node's `*Sync` APIs all block the loop. So does heavy `JSON.parse` of large payloads. Move to streaming or async equivalents; offload heavy CPU work to a worker thread.
2. **Long-running computations on the main thread.** Image transforms, data joins, regex backtracking. Worker threads or out-of-process workers.
3. **Tight loops with no yield point.** A `for` loop processing 100k items synchronously starves everything else.
4. **Exhausted thread pool from `crypto`, `dns.lookup`, or `fs`.** The libuv thread pool is finite (default 4); saturating it makes async APIs feel synchronous.
5. **Garbage collection pauses under high allocation pressure.** Less common, but visible on the delay histogram as periodic spikes correlating with GC.

The instrumentation lets you distinguish these. Sustained high p99 with normal p50 looks like episodic heavy work; both p50 and p99 climbing together looks like sustained pressure.

## Composing with other practices

- [[Languages/TypeScript/Practices/Foundational/Runtime Suitability Boundary]] — the latency-suitability decision is the upstream decision; this metric is what continuously validates it.
- [[Languages/TypeScript/Practices/Async and Concurrency/Deadline Propagation with AbortSignal]] — every outbound IO has a deadline, but the deadline is only as honest as the loop delay allows. ELU tells you whether your deadlines are still honest.
- [[Engineering Philosophy/Principles/Error Budgets and Reliability Policy]] — event-loop delay is a leading indicator of error-budget burn.
- [[Engineering Philosophy/Principles/Staged Canary Deployment]] — canary promotion can be gated on event-loop-delay regression vs the control population.
- [[Engineering Philosophy/Principles/Layered Risk Categories]] — this metric belongs to the runtime layer.

## Related

- [[Languages/TypeScript/Practices/Foundational/Runtime Suitability Boundary]]
- [[Languages/TypeScript/Practices/Async and Concurrency/Deadline Propagation with AbortSignal]]
- [[Languages/TypeScript/Practices/Async and Concurrency/Async Context Propagation]]
- [[Engineering Philosophy/Principles/Error Budgets and Reliability Policy]]
- [[Engineering Philosophy/Principles/Staged Canary Deployment]]
