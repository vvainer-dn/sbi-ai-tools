---
title: SBI secrets and credential resolution
type: architecture
owner: dap-tcs-sbi
created: '2026-08-17'
updated: '2026-08-17'
sources:
- '"SBI & TCS — Living Knowledge Wiki" HTML knowledge doc (author: vvainer@drivenets.com), session 13'
evidence:
- drivenets/dap:dap-workspace/apps/sbi/edge/sbi-controller
- drivenets/dap:dap-workspace/apps/secrets-manager
---

# SBI secrets and credential resolution

## Overview

An SBI request carries secret references, not cleartext credentials. The caller puts template placeholders and path parameters in the request; the controller resolves the real value at the point of use, from an edge Secrets Manager or from Vault, and authenticates or renders locally. No secret value crosses Pulsar.

## Components

- What crosses Pulsar: identifiers and lookup context, such as secret category, device or site identity, secret-path parameters, and version metadata. These are closer to a structured lookup key than a public URL.
- Secrets Manager (`apps/secrets-manager`): the default edge-local resolver, so secrets can be obtained near execution.
- Vault: a supported source mode. Legacy `{{SECRET}}` tokens are a Vault-specific carve-out. Provider selection is configured, not inferred from the request topic.
- The credential cascade that consumes these values lives in [[sbi-device-edge]].

## Flow

```
CMS / workflow            template placeholders + path parameters
   |
   v
SBIRequest                category, CLLI, version, no secret value
   |
   v
Active SBI Controller     resolve at point of use
   |
   v
Secrets Manager or Vault  authorized lookup
   |
   v
Device                    authenticate / render locally
```

Failure posture: provider failures fail closed. The controller must not guess, log, or reuse an unverified value. Availability therefore depends on the configured provider or an explicitly supported local capability.

## Decisions

- Pass references, resolve at the edge. Keeping cleartext out of caller databases, templates, and broker payloads lets Vault or Secrets Manager rotate and gate values without copies spreading through the system.
- Fail closed on a provider error. A wrong or stale credential is a controlled failure; a guessed or reused value would be a security defect, so the controller stops instead.
- TLS still matters even though the message carries references rather than values. Identifiers, commands, device topology, and response data are sensitive, and the controller-to-provider call must also be authenticated and authorized.

## Open questions

_None recorded yet._

## Source

- Primary: the "SBI & TCS — Living Knowledge Wiki" HTML knowledge doc (author: vvainer@drivenets.com), session 13, synthesizing the DAP ToI session of 20 May 2026.
- Verified against the repo: [`sbi-controller`](https://github.com/drivenets/dap/tree/develop/dap-workspace/apps/sbi/edge/sbi-controller) and [`secrets-manager`](https://github.com/drivenets/dap/tree/develop/dap-workspace/apps/secrets-manager) in [drivenets/dap](https://github.com/drivenets/dap), repository state of 17 Aug 2026.

## See also

- [[sbi-device-edge]]: the credential cascade that uses the resolved value
- [[sbi-ha-and-failover]]: why secret availability during isolation is still an open standalone concern
