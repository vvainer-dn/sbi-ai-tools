---
title: SBI & TCS overview
type: architecture
owner: dap-tcs-sbi
created: '2026-08-17'
updated: '2026-08-17'
sources:
- '"SBI & TCS — Living Knowledge Wiki" HTML knowledge doc (author: vvainer@drivenets.com), synthesizing the "DAP ToI 10. SBIs (Southbound Interfaces)" recorded session of 20 May 2026'
- drivenets/dap:dap-workspace/apps/sbi/
- drivenets/dap:dap-workspace/apps/tcs/
evidence:
- drivenets/dap:dap-workspace/apps/sbi/cloud/message-bus
- drivenets/dap:dap-workspace/apps/sbi/edge/sbi-controller
- drivenets/dap:dap-workspace/libs/shared/proto-sources/sbi/sbi.proto
---

# SBI & TCS overview

## Overview

SBI (South Bound Interface) and TCS (Telemetry Control Services) are one team's two related domains inside DAP. SBI is the distributed device-access infrastructure that carries a command from a DAP cloud caller to a physical network device or NMS, and carries the result back. TCS is a telemetry-oriented service family split in two: one half depends on SBI, the other reaches external telemetry systems directly and never touches a device.

Terminology note: the source material for this page consistently uses TCS, not TCM. TCS means Telemetry Control Services, and no component named or resembling "TCM" exists in the SBI/TCS scope. Treat "TCM" as a likely shorthand or typo for TCS unless a separate project or repo turns up under that name.

## Components

SBI owns the southbound path:
- Cloud-side orchestration and routing.
- Edge-side protocol execution against devices and NMS systems (CLI, gNMI, HTTP, NETCONF, console, file transfer).
- Inventory-derived device-to-controller mapping.
- Credential resolution at the point of use.

TCS owns telemetry control and access, as two independent services:
- TCS-Subscriptions: subscription policy, gNMI event fanout, path-value persistence, listener lifecycle. Depends on SBI, since controllers own the actual gNMI connections.
- TCS API: gRPC access to external telemetry systems. Snowflake querying and feature-gated IEBus gNMI publishing are live; ELK is a reserved, unimplemented stub. Does not depend on SBI, since these targets leave DAP directly.

## Flow (mental model)

```
DAP clients (CMS, workflows, services)
        |
        v
Pulsar + Message Bus   route and decouple
        |
        v
SBI Controller         execute at the DCU edge
        |
        v
Device / NMS           CLI, gNMI, HTTP, NETCONF, ...
```

The Message Bus resolves which controller owns a given device and routes the command there. The controller is the only thing that actually talks to the device. See [[sbi-control-data-plane]] for the full request/response sequence.

## Open questions

_None recorded yet._

## Source

- Primary: the "SBI & TCS — Living Knowledge Wiki" HTML knowledge doc authored by vvainer@drivenets.com, which synthesizes the "DAP ToI 10. SBIs (Southbound Interfaces)" recorded session of 20 May 2026. Ask the author for the doc or recording location.
- Verified against the repo: [`dap-workspace/apps/sbi/`](https://github.com/drivenets/dap/tree/develop/dap-workspace/apps/sbi) and [`dap-workspace/apps/tcs/`](https://github.com/drivenets/dap/tree/develop/dap-workspace/apps/tcs) in [drivenets/dap](https://github.com/drivenets/dap), repository state of 17 Aug 2026.

## See also

- [[sbi-control-data-plane]]: control plane vs. data plane, Pulsar topics, request/response flow
- [[sbi-device-edge]]: protocols, credentials, file service, device onboarding
- [[tcs-service-split]]: TCS-Subscriptions vs. TCS API in detail
- [[tcs-subscriptions-and-wait-listen]]: subscription lifecycle and the Wait & Listen pattern
- [[sbi-ha-and-failover]]: HA state machine and connection ownership
- [[sbi-secrets-and-vault]]: credential resolution boundary
- [[sbi-debugging-and-fmea]]: triage sequence and failure-mode analysis
- [[sbi-device-ecosystem]]: DNOS, DNOR, and the managed device/NMS landscape
