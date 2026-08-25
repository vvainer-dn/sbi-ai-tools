---
title: The device-facing edge
type: architecture
owner: dap-tcs-sbi
created: '2026-08-17'
updated: '2026-08-17'
sources:
- '"SBI & TCS — Living Knowledge Wiki" HTML knowledge doc (author: vvainer@drivenets.com), session 3'
evidence:
- drivenets/dap:dap-workspace/apps/sbi/edge/sbi-controller
- drivenets/dap:dap-workspace/apps/sbi/edge/fs/fs-daemon
- drivenets/dap:dap-workspace/apps/sbi/edge/fs/fs-publisher
---

# The device-facing edge

## Overview

At the edge, SBI stays protocol-aware but tries to stay business-logic-light. The caller supplies the commands; the controller handles transport, authentication, connection lifecycle, and the edge coordination that cannot be avoided. Some operations are single request/response; others hold a connection open and so participate in controller lifecycle and HA.

## Components

- Unary-style operations: CLI, HTTP, NETCONF, console, and many file commands are execute, return, release.
- Long-lived operations: gNMI subscriptions keep open connections, so they take part in controller lifecycle, HA activation, drain, and reconnect.
- Pass-through philosophy: the calling workflow normally supplies the CLI. Controller-side question-and-answer patterns and secret placeholders handle interaction and secure material.
- File service: FS Daemon (`apps/sbi/edge/fs/fs-daemon`) schedules Artifactory transfer jobs; FS Publisher (`apps/sbi/edge/fs/fs-publisher`) exposes the shared cached file to devices over HTTP, FTP, SFTP, and TFTP.

Credential cascade, tried in order until one authenticates:

1. AAA, 11-character CLLI
2. AAA, 8-character CLLI
3. Local, 11-character CLLI
4. Local, 8-character CLLI
5. Device-type default

The recording framed a dedicated Secret Manager as future work. The maintained system supports Vault and Secrets Manager v2 modes, with legacy secret-token resolution still using Vault in the latter mode. See [[sbi-secrets-and-vault]].

Why the file service exists: large binaries and support bundles should not occupy Pulsar payload channels. Two directions:
- NOS image: Artifactory to DCU cache to device.
- Support bundle: device to DCU cache to Artifactory.

## Flow

NOS image delivery:

```
1. Workflow asks SBI to stage an image   -> routed to the owning controller
2. FS Daemon downloads asynchronously    -> Artifactory to shared DCU cache/PVC; job polled to terminal state
3. Controller resolves publisher address -> substitutes it into device-facing commands
4. Device pulls and applies              -> downloads from the edge-local publisher; workflow verifies and applies
```

Large files do not travel inside normal Pulsar commands. Pulsar carries control instructions; Artifactory and the edge file service carry the file bytes.

Device onboarding places SBI in two security-sensitive steps without making it the owner of the onboarding workflow:

```
Factory-default device -> external/upstream prep -> CMS configuration (template + secret placeholders)
  -> SBI applies config (resolve secrets, execute) -> SBI rotates gNMI trust (non-default certificate)
```

## Decisions

- Keep file bytes off Pulsar. Routing a NOS image or a support bundle through broker payloads would strain the message path, so the file service moves bytes over device-compatible protocols and Pulsar only carries the instruction.
- The credential cascade is ordered, not parallel. AAA is preferred over local, and the longer CLLI form is tried before the shorter one, so the most specific authorized identity wins first.

## Open questions

_None recorded yet._

## Source

- Primary: the "SBI & TCS — Living Knowledge Wiki" HTML knowledge doc (author: vvainer@drivenets.com), session 3, synthesizing the DAP ToI session of 20 May 2026.
- Verified against the repo: [`sbi-controller`](https://github.com/drivenets/dap/tree/develop/dap-workspace/apps/sbi/edge/sbi-controller), [`fs-daemon`](https://github.com/drivenets/dap/tree/develop/dap-workspace/apps/sbi/edge/fs/fs-daemon), and [`fs-publisher`](https://github.com/drivenets/dap/tree/develop/dap-workspace/apps/sbi/edge/fs/fs-publisher) in [drivenets/dap](https://github.com/drivenets/dap), repository state of 17 Aug 2026.

## See also

- [[sbi-secrets-and-vault]]: the credential resolution the cascade above depends on
- [[sbi-ha-and-failover]]: why long-lived gNMI operations take part in controller HA
- [[sbi-device-ecosystem]]: the devices and NMS systems these protocols target
