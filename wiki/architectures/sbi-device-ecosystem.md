---
title: SBI device and NMS ecosystem
type: architecture
owner: dap-tcs-sbi
created: '2026-08-17'
updated: '2026-08-17'
sources:
- '"SBI & TCS — Living Knowledge Wiki" HTML knowledge doc (author: vvainer@drivenets.com), session 14'
evidence:
- drivenets/dap:dap-workspace/apps/sbi/edge/sbi-controller
---

# SBI device and NMS ecosystem

## Overview

SBI targets a mix of network devices and management systems. Different controllers and handlers exist because their APIs, session semantics, data models, and failure behavior differ. Two DriveNets names are easy to confuse: DNOS is the network operating system on the device, and DNOR is a separate orchestration system, not another spelling of DNOS.

## Components

| Name | Meaning in this context | Typical SBI interaction |
|---|---|---|
| DNOS | DriveNets Network Operating System on network infrastructure | Direct device CLI/SSH and model-driven gNMI |
| DNOR | DriveNets Network Orchestrator, a management/orchestration system | HTTP/REST and gNMI-facing integration as configured |
| Nokia BNG | Broadband Network Gateway; classic and model-driven CLI variants differ | CLI/SSH with variant-specific handling; telemetry where supported |
| Nokia OLT | Optical Line Terminal for access networks | Device-specific CLI/management integration |
| Nokia AMS | Access Management System | SOAP requests and JMS-related event integration |
| Nokia NSP | Network Services Platform | REST/SOAP management APIs |
| Ciena ESM | Ciena Element and Service Manager | REST-based management integration |
| CPNR | Cisco Prime Network Registrar | REST-based IP/DNS/DHCP management integration |

Why gNMI is called modern: it is model-driven, so YANG/OpenConfig paths describe structured state, gRPC and Protobuf provide a typed contract, and streaming subscriptions deliver changes continuously. CLI is human-oriented text with vendor syntax and parsing risk, but stays necessary when a device exposes no equivalent modeled API.

## Decisions

- One SBI release per DCU with a single binary that selects protocol and device handlers through the device-type map. Legacy deployments separated device families into distinct controller workloads; the current direction consolidates them, though environment migration can still leave a mixed topology.

## Open questions

_None recorded yet._

## Source

- Primary: the "SBI & TCS — Living Knowledge Wiki" HTML knowledge doc (author: vvainer@drivenets.com), session 14, synthesizing the DAP ToI session of 20 May 2026. Product names for DNOS and DNOR were cross-checked against the public DriveNets product pages.
- Verified against the repo: [`sbi-controller`](https://github.com/drivenets/dap/tree/develop/dap-workspace/apps/sbi/edge/sbi-controller) in [drivenets/dap](https://github.com/drivenets/dap), repository state of 17 Aug 2026.

## See also

- [[sbi-device-edge]]: the protocols and connection lifecycle used against these targets
