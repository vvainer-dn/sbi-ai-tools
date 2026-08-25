# sbi-ai-tools

Knowledge base for the **SBI/TCS domain** in DAP — the data layer for an AI domain
agent that will help the team triage SBI/TCS issues and attempt automated resolution.

Modeled on [drivenets/dap-ai-tools](https://github.com/drivenets/dap-ai-tools)
(the DAP DevOps knowledge wiki): same page anatomy, slug conventions, and
category-per-page-type layout.

## Provenance

The seed content (9 pages) was copied from the knowledge-mesh wiki
[`dap-tcs-sbi`](https://knowledge-mesh.dev.ai.drivenets.net/browse#/dap-tcs-sbi),
which synthesizes:

- the **"DAP ToI 10. SBIs (Southbound Interfaces)"** recorded session of 20 May 2026
- the **"SBI & TCS — Living Knowledge Wiki"** HTML knowledge doc (author: vvainer@drivenets.com)
- verification against [drivenets/dap](https://github.com/drivenets/dap)
  (`dap-workspace/apps/sbi/`, `dap-workspace/apps/tcs/`), repository state of 17 Aug 2026

## Structure

| Path | Contents | Status |
|---|---|---|
| `index.md` | Root wiki index | generated |
| `wiki/architectures/` | Architectural facets of SBI/TCS (boundaries, flows, decisions) | **8 pages (seeded)** |
| `wiki/operations/` | Consolidated triage playbook + FMEA | **1 page (seeded)** |
| `wiki/patterns/` | Recurring symptom → root cause → fix recipes | placeholder |
| `wiki/incidents/` | One page per real incident | placeholder |
| `wiki/resolved_tickets/` | One page per resolved AR-XXXXX ticket | placeholder |
| `wiki/services/` | Per-service runbooks | placeholder |
| `kc-config/` | Page-type schema + profile (`sbi-tcs.schema.yaml`, `sbi-tcs.profile.toml`) | config |
| `scripts/SLUG-CONVENTIONS.md` | Naming contract that keeps wikilinks resolvable | doc |

Page anatomy: YAML frontmatter (`title`, `type`, `owner`, `created`, `updated`,
`sources`, `evidence`) → H1 repeating the title → `## ` sections in schema order →
mandatory `## Source` → `## See also` with `[[wikilinks]]`.

## SBI/TCS service map (code side)

The wiki documents *why/how*; the code and its `.ai/` docs in
[drivenets/dap](https://github.com/drivenets/dap) document *what/where*. Cross-reference,
don't duplicate.

| Zone | Service | Lang | Purpose |
|---|---|---|---|
| cloud | `message-bus` | Python | Routes `SBIRequest` messages to per-controller Pulsar topics |
| cloud | `sbi-orchestrator` | Python | Inventory sync; publishes per-device config to compacted topics |
| cloud | `sbi-seed` | Python | K8s Job: SBI DB schema + inventory seed |
| cloud | `sbi-clickhouse-migrations` / `tcs-clickhouse-migrations` | Go | ClickHouse schema migrations |
| cloud | `tcs-subscriptions` | Go | Telemetry subscription manager: policy aggregation, gNMI event fanout, Wait & Listen |
| edge | `sbi-controller` | Go | The only component that talks to devices (CLI, gNMI, HTTP, NETCONF, console); HA |
| edge | `fs-daemon` / `fs-publisher` | Python / Go | Edge file service: Artifactory transfer jobs + HTTP/FTP/SFTP/TFTP serving |
| edge | `jmscollector` | Java | Nokia AMS JMS/XML events → Protobuf → Pulsar |
| edge | `lweopm` | Go + React | Emergency on-prem device-access UI per DCU |
| tcs | `tcs-api` | Python | Read-only gRPC proxy: Snowflake (live), IEBus (feature-gated), ELK (stub) |

Best code-side docs: `dap-workspace/apps/sbi/.ai/` (domain-level),
`apps/sbi/cloud/tcs-subscriptions/.ai/`, `apps/sbi/edge/sbi-controller/.ai/`.

## Read / write model

- **Read**: agents (and humans) consult this wiki first for domain knowledge —
  start at `index.md`, follow `[[wikilinks]]`, grep for symptoms.
- **Write**: today, manual/seeded. Next phases add the triage-agent write path
  (`pattern` / `incident` / `resolved_ticket` / `service` pages) following
  `kc-config/sbi-tcs.schema.yaml` and `scripts/SLUG-CONVENTIONS.md`.

## Roadmap

1. ✅ Seed wiki from the `dap-tcs-sbi` knowledge-mesh wiki (this repo).
2. SBI triage agent: reads this wiki + `drivenets/dap` code, triages Jira/incident
   input, writes back `incident`/`pattern`/`resolved_ticket` pages.
3. Per-service `service` runbook pages.
4. MCP read access + refresh automation (mirroring dap-ai-tools' `wiki-mcp` /
   `kc-agent` model).
