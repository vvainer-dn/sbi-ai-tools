---
title: TCS service split (Subscriptions vs API)
type: architecture
owner: dap-tcs-sbi
created: '2026-08-17'
updated: '2026-08-17'
sources:
- '"SBI & TCS — Living Knowledge Wiki" HTML knowledge doc (author: vvainer@drivenets.com), session 4'
evidence:
- drivenets/dap:dap-workspace/apps/sbi/cloud/tcs-subscriptions
- drivenets/dap:dap-workspace/apps/tcs/tcs-api
---

# TCS service split (Subscriptions vs API)

## Overview

TCS is two services that share a name for organizational reasons and take very different paths. Do not infer that TCS API routes through SBI, or that TCS-Subscriptions stores the historical analytics queried from Snowflake. One controls live device telemetry through SBI; the other reaches external telemetry systems directly.

## Components

TCS-Subscriptions (`apps/sbi/cloud/tcs-subscriptions`): controls live gNMI subscriptions. A Pulsar-driven service that validates subscription requests, aggregates YANG paths by device, pushes policies to controllers, fans out device events, stores last path values, and bridges events to waiting Temporal workflows. It uses SBI, because controllers own the actual gNMI connections. Full lifecycle in [[tcs-subscriptions-and-wait-listen]].

TCS API (`apps/tcs/tcs-api`): accesses or triggers external telemetry systems. A gRPC service for Snowflake querying and feature-gated IEBus gNMI publishing; the ELK contract is reserved but not implemented. It does not use SBI, because these targets leave DAP directly.

Current TCS API target matrix:

| Target | Transport | Current behavior | Guard / reliability detail |
|---|---|---|---|
| Snowflake | gRPC to authenticated connector | Query with paginated results | Feature flag, validation, connection pool, read-only role |
| IEBus | gRPC to Kafka producer | Publishes gNMI subscription requests | Feature flag, input validation, idempotency identifier |
| ELK | gRPC contract | Reserved stub | Not implemented |

Retrieval cue for agents: a Snowflake or IEBus question belongs to TCS API. A live device gNMI policy or event-fanout question belongs to TCS-Subscriptions plus the SBI Controller.

## Flow

The two services do not chain. They are reached independently by their callers:

```
Live telemetry demand -> TCS-Subscriptions -> SBI Controller -> device gNMI stream
External telemetry     -> TCS API           -> Snowflake / IEBus (leaves DAP; no SBI)
```

## Decisions

- Keep the two concerns in separate services despite the shared TCS label. Live subscription control has a device-facing dependency on SBI and a very different lifecycle from a stateless gRPC query against an external warehouse, so folding them together would couple unrelated failure domains.

## Open questions

_None recorded yet._

## Source

- Primary: the "SBI & TCS — Living Knowledge Wiki" HTML knowledge doc (author: vvainer@drivenets.com), session 4, synthesizing the DAP ToI session of 20 May 2026.
- Verified against the repo: [`tcs-subscriptions`](https://github.com/drivenets/dap/tree/develop/dap-workspace/apps/sbi/cloud/tcs-subscriptions) and [`tcs-api`](https://github.com/drivenets/dap/tree/develop/dap-workspace/apps/tcs/tcs-api) in [drivenets/dap](https://github.com/drivenets/dap), repository state of 17 Aug 2026.

## See also

- [[overview]]: SBI & TCS overview
- [[tcs-subscriptions-and-wait-listen]]: the TCS-Subscriptions lifecycle in detail
- [[sbi-control-data-plane]]: the gNMI policy topic TCS-Subscriptions writes to
