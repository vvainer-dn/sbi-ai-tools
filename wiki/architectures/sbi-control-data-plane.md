---
title: SBI control plane and data plane
type: architecture
owner: dap-tcs-sbi
created: '2026-08-17'
updated: '2026-08-17'
sources:
- '"SBI & TCS — Living Knowledge Wiki" HTML knowledge doc (author: vvainer@drivenets.com), session 2'
evidence:
- drivenets/dap:dap-workspace/apps/sbi/cloud/message-bus
- drivenets/dap:dap-workspace/apps/sbi/cloud/sbi-orchestrator
- drivenets/dap:dap-workspace/apps/sbi/edge/sbi-controller
---

# SBI control plane and data plane

## Overview

An SBI request splits across two planes, both carried over Pulsar. The configuration plane distributes current controller state through compacted topics. The data plane carries the operational request, response, and event traffic. The orchestrator controls who owns what, the Message Bus routes each command to the owning controller, and the controller executes it against the device.

## Components

- Configuration plane: compacted Pulsar topics whose latest value per key reconstructs current controller state. Three families ride here (see the table below).
- Data plane: request, response, and event topics carrying live operational traffic.
- Message Bus (`apps/sbi/cloud/message-bus`): rejects an empty or unknown device, resolves NE/NMS controller ownership from the SBI inventory projection, and publishes to the controller-specific topic. Callers stay independent of controller placement.
- SBI Controller (`apps/sbi/edge/sbi-controller`): the only component that talks to the device.
- Inventory: upstream device truth. Inventory change events plus seed and reconciliation logic populate SBI's narrower device/controller model, so the SBI DB is an operational projection, not the source of record.

## Flow

Representative request, CMS or workflow to device and back:

```
1. Caller       -> Pulsar        publish SBIRequest (device, source, correlation)
2. Pulsar       -> Message Bus   deliver cloud request
3. Message Bus                   resolve device -> controller
4. Message Bus  -> Pulsar        publish controller request topic
5. Pulsar       -> Controller    active failover consumer receives
6. Controller   -> Device        execute protocol operation
7. Device       -> Controller    result or protocol error
8. Controller   -> Pulsar        publish destination response topic (persist locally if disconnected)
9. Pulsar       -> Caller        deliver and correlate response
```

The response takes the direct return path. The controller already knows the destination service, so it publishes straight to that service's response topic. The Message Bus is not on the return path. If Pulsar is unavailable, edge resilience can persist outbound messages for later delivery.

Configuration topic families, as current-state inputs to a controller:

| Configuration family | Owner | Meaning at controller | Removal behavior |
|---|---|---|---|
| Device configuration | SBI Orchestrator | Connection details, device type, protocol capabilities | A tombstone removes the device from the controller view |
| gNMI policy | TCS-Subscriptions | Aggregated telemetry policy for one device | A tombstone removes the active policy |
| HA configuration | SBI HA control path | Failover peers and priority | Updates change pipeline ownership readiness |

Retrieval cue for agents: "who decides the controller?" is the SBI Orchestrator and allocation state. "Who routes a command?" is the Message Bus. "Who talks to the device?" is the SBI Controller.

## Decisions

- The return path bypasses the Message Bus on purpose. Routing is only needed to find the owning controller; the controller already holds the destination, so a second routing hop would add latency and coupling for nothing.
- Failure location follows the split: a missing or unknown device fails in the Message Bus and returns a caller-facing SBI error. A present device with bad connection attributes usually fails later, at the controller.

## Open questions

_None recorded yet._

## Source

- Primary: the "SBI & TCS — Living Knowledge Wiki" HTML knowledge doc (author: vvainer@drivenets.com), session 2, synthesizing the DAP ToI session of 20 May 2026.
- Verified against the repo: [`message-bus`](https://github.com/drivenets/dap/tree/develop/dap-workspace/apps/sbi/cloud/message-bus), [`sbi-orchestrator`](https://github.com/drivenets/dap/tree/develop/dap-workspace/apps/sbi/cloud/sbi-orchestrator), and [`sbi-controller`](https://github.com/drivenets/dap/tree/develop/dap-workspace/apps/sbi/edge/sbi-controller) in [drivenets/dap](https://github.com/drivenets/dap), repository state of 17 Aug 2026.

## See also

- [[overview]]: SBI & TCS overview
- [[sbi-ha-and-failover]]: how the active failover consumer in step 5 is elected
- [[sbi-debugging-and-fmea]]: isolating which boundary in this flow failed
