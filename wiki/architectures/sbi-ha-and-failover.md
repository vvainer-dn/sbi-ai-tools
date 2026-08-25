---
title: SBI HA and failover
type: architecture
owner: dap-tcs-sbi
created: '2026-08-17'
updated: '2026-08-17'
sources:
- '"SBI & TCS — Living Knowledge Wiki" HTML knowledge doc (author: vvainer@drivenets.com), sessions 7, 10'
evidence:
- drivenets/dap:dap-workspace/apps/sbi/edge/sbi-controller
- drivenets/dap:dap-workspace/apps/sbi/edge/lweopm
---

# SBI HA and failover

## Overview

Controller high availability is a pipeline state machine, not two active replicas. Kubernetes handles process restart, Pulsar `Failover` elects one active request consumer per controller topic, controller drain protects in-flight work, and PVC-backed resilience covers more of the disconnect case than the original recording described. The unit that matters is the controller-topic subscription: Pulsar picks the active consumer, and controller HA callbacks activate or drain its device pipeline.

## Components

- Active pipeline: owns the request partition and the live device connections.
- Standby pipeline: configured with device state for failover, but not serving devices.
- Pulsar `Failover` subscription: one eligible consumer is active, another can take over, with explicit priority.
- Watchdog: steps an active pipeline down when broker reachability is lost.
- Disconnected self pipeline: a special local-serving path when Pulsar is unavailable. It is not a second cross-DCU leader.
- Direct local execution API plus LWEOPM (`apps/sbi/edge/lweopm`): a separate emergency-access surface at the DCU.

Connection maintenance by protocol:
- gNMI: long-lived controller-initiated streaming subscriptions with reconnect handling.
- CLI/SSH: reusable session management with idle expiry; requests drive use.
- NETCONF: a fresh SSH-backed session per request.
- HTTP, console, and file transfer: request or job driven.

Visibility: `GetDevices` exposes controller and HA state plus protocol status. The current snapshot reliably populates gNMI connectivity; CLI-related fields exist but are not equally populated, so a blank CLI status is not proof of no reachability.

## Flow

Active to draining to standby:

```
Active                       owns request partition and device connections
   |
   v
Partition loss / watchdog    stop accepting new work
   |
   v
Draining                     finish or bound in-flight work
   |
   v
Standby                      SSH wiped, gNMI closed
```

Degraded boot when Pulsar is unavailable:

```
Pulsar unavailable   -> Load PVC snapshot (device configuration cache)
  -> Serve self pipeline (local/direct path where enabled)
  -> Reconnect (refresh snapshot + live updates)
  -> Flush outbound queue (responses and events)
```

## Decisions

- Elect one active consumer per controller topic with `Failover`, rather than running two active writers. Two active pipelines could both touch the same device, so the design keeps a single active leader and preconfigured standbys.
- Drain before standby. Promotion and demotion callbacks drive Active to Draining to Standby, and the watchdog steps an active pipeline down on lost broker reachability, to shrink the unsafe overlap window during a transition.
- Load a PVC snapshot at boot when the configuration topic is unreachable, then reconcile live state after reconnect, so a broker outage does not leave a controller with no allocation.

State of the standalone story: the May 2026 recording presented full local operation as future work. The maintained system now has real building blocks, including PVC configuration snapshots, locally persisted and flushed outbound responses and events, a disconnected self-pipeline serving path, and a direct local execution API with LWEOPM as a separate emergency surface. This does not prove that every product-level standalone requirement is complete, especially secret availability, operational exposure, and deployment policy.

## Open questions

_None recorded yet._

## Source

- Primary: the "SBI & TCS — Living Knowledge Wiki" HTML knowledge doc (author: vvainer@drivenets.com), sessions 7 and 10, synthesizing the DAP ToI session of 20 May 2026.
- Verified against the repo: [`sbi-controller`](https://github.com/drivenets/dap/tree/develop/dap-workspace/apps/sbi/edge/sbi-controller) and [`lweopm`](https://github.com/drivenets/dap/tree/develop/dap-workspace/apps/sbi/edge/lweopm) in [drivenets/dap](https://github.com/drivenets/dap), repository state of 17 Aug 2026.

## See also

- [[sbi-control-data-plane]]: the HA configuration topic and the active failover consumer
- [[sbi-device-edge]]: the long-lived gNMI connections a pipeline owns
- [[sbi-debugging-and-fmea]]: split-brain and failover-transition triage
