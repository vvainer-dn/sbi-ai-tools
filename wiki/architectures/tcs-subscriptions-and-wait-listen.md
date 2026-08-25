---
title: TCS-Subscriptions and Wait & Listen
type: architecture
owner: dap-tcs-sbi
created: '2026-08-17'
updated: '2026-08-17'
sources:
- '"SBI & TCS — Living Knowledge Wiki" HTML knowledge doc (author: vvainer@drivenets.com), sessions 5, 6, 15'
evidence:
- drivenets/dap:dap-workspace/apps/sbi/cloud/tcs-subscriptions
- drivenets/dap:dap-workspace/apps/sbi/edge/sbi-controller
---

# TCS-Subscriptions and Wait & Listen

## Overview

TCS-Subscriptions turns a requested YANG path into a delivered event, and the Wait & Listen pattern lets a Temporal workflow sleep durably until a correlated event arrives. They solve different halves: TCS maintains device-facing telemetry demand; Wait & Listen handles the asynchronous wait on the consumer side. The key optimization is policy aggregation, so many logical subscribers can share one device-level gNMI policy and connection.

## Components

Subscribe path:
1. Validate identifiers, source, response destination, and paths.
2. Serialize changes per device through a database transaction lock.
3. Make duplicate subscription and correlation pairs idempotent.
4. If an active policy already covers the paths, activate immediately and send initial stored values.
5. Otherwise build and apply a new union policy.

Policy versus subscription: a subscription records one consumer's interest. A policy is the device-facing aggregate the controller applies. If CMS and two workflows request overlapping paths on one device, three logical subscriptions still need only one gNMI connection. TCS computes a per-device path union and matches incoming events back to each logical subscription.

State you should recognize:

| Entity | Representative states | Operational meaning |
|---|---|---|
| Subscription | `pending`, `active`, `error` | Consumer intent and whether a usable policy serves it |
| Policy | `wait`, `pending`, `syncing`, `active`, `error`, `superseded`, `deleted` | Device-facing aggregate lifecycle |
| Listener | `waiting` | A Temporal workflow is registered for a subscription topic; the row is removed after dispatch |
| Path value | last-known value per device/path | Change detection and initial sync for late subscribers |

## Flow

Three-phase event fanout:

```
1. Resolve   read state, detect changes, match subscribers
      |
      v
2. Fanout    publish before state mutation
      |
      v
3. Persist   store/delete last path values
```

The order matters. If the service stops after publishing but before persisting, redelivery can produce a duplicate, which is acceptable under at-least-once delivery. Persisting first could suppress the redelivered change and silently lose the subscriber event. Describe this contract as at-least-once, not exactly-once: duplicates are possible; silent loss is the failure the ordering avoids.

Wait & Listen lifecycle, the consumer-side wait:

```
1. Workflow subscribes         TCS returns a subscription ID; may activate immediately if a policy exists
2. Workflow registers listener TCS binds the per-subscription Pulsar topic to the workflow; dispatches now if events pend
3. Workflow sleeps             Temporal keeps durable state without holding an activity worker
4. Device event is fanned out  the subscription topic buffers delivery; listener signals the workflow via the executions API
5. Workflow validates          if the condition is wrong, register again and wait; buffered later events are not lost
6. Workflow unsubscribes       TCS cleans up listener/topic state and recomputes the device policy
```

If a response arrives before listener registration completes, TCS can retain it as a pending response and match it later, with TTL cleanup. That closes the classic asynchronous race.

## Decisions

- Aggregate subscriptions into one per-device policy rather than opening one gNMI stream per logical subscriber. Ten consumers asking for the same path would otherwise create ten identical streams; the union plus per-subscriber matching removes that duplication.
- Publish before persist, accepting at-least-once. The system chooses possible duplicates over possible silent loss, because a duplicate is recoverable by an idempotent consumer and a lost event is not.
- Wait on Temporal state, not a held worker. A blocked activity worker per wait would exhaust capacity under many long-lived waits; the listener plus Temporal signal lets the workflow sleep durably while Pulsar retains events until a listener is ready.

## Open questions

_None recorded yet._

## Source

- Primary: the "SBI & TCS — Living Knowledge Wiki" HTML knowledge doc (author: vvainer@drivenets.com), sessions 5, 6, and 15, synthesizing the DAP ToI session of 20 May 2026.
- Verified against the repo: [`tcs-subscriptions`](https://github.com/drivenets/dap/tree/develop/dap-workspace/apps/sbi/cloud/tcs-subscriptions) and [`sbi-controller`](https://github.com/drivenets/dap/tree/develop/dap-workspace/apps/sbi/edge/sbi-controller) in [drivenets/dap](https://github.com/drivenets/dap), repository state of 17 Aug 2026.

## See also

- [[tcs-service-split]]: where TCS-Subscriptions sits relative to TCS API
- [[sbi-control-data-plane]]: the gNMI policy configuration topic this service writes to
- [[sbi-debugging-and-fmea]]: duplicate-event and lost-initial-value triage
