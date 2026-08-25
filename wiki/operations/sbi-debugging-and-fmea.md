---
title: SBI & TCS debugging and FMEA
type: operations
owner: dap-tcs-sbi
created: '2026-08-17'
updated: '2026-08-17'
sources:
- '"SBI & TCS — Living Knowledge Wiki" HTML knowledge doc (author: vvainer@drivenets.com), sessions 8, 16'
evidence:
- drivenets/dap:dap-workspace/apps/sbi/cloud/message-bus
- drivenets/dap:dap-workspace/apps/sbi/edge/sbi-controller
---

# SBI & TCS debugging and FMEA

## Symptoms

Debug the path, not the symptom. Start at the caller and follow the correlation through routing, ownership, edge execution, device response, and return delivery. Each boundary has a different failure signature. The observable signals you will actually see include: an immediate "device unknown/unusable" response; a routed request that then fails to connect; authentication cascade failures; no response during a Pulsar partition; a duplicate telemetry event; a late subscriber that sees no initial values; a standby that also touches the device; and TCS API returning `UNIMPLEMENTED`.

## Triage Sequence

Follow the correlation from caller to return delivery. Never paste secret values at any step.

1. Establish identity. Capture `correlation_id`, request ID, device ID, source service, operation type, and time window.
2. Prove caller publication. Check the CMS or workflow activity outcome and whether the request reached its cloud request topic.
3. Prove routing. In the Message Bus, did device lookup return a controller? Was the controller topic selected? An unknown device produces a caller-facing SBI error.
4. Prove controller ownership. In the orchestrator, DB, and config topic, is the device assigned, live, and published under the expected controller? Check HA pipeline active or standby state.
5. Prove execution. On the controller, check deserialization, capability validation, credential resolution, the protocol handler, timeout and retry, and the device connection.
6. Separate controller error from device error. Bad IP, missing credentials, or handler failures originate before or on the controller. A CLI or device rejection can be faithfully proxied upward.
7. Prove return delivery. Confirm the response or event entered the persistent outbound path and reached `{dest_service}-responses` or the subscription event topic.

Symptom to boundary to smallest next check:

| Symptom | Likely boundary | Next check |
|---|---|---|
| Immediate "device unknown/unusable" response | Message Bus mapping | Inventory projection and NE/NMS controller lookup |
| Routed, then connection failure | Controller config / network | Management IP, port, protocol capability, DCU reachability |
| Authentication cascade failures | Secrets + device AAA/local auth | Secret path metadata, source mode, CLLI variants; do not expose values |
| No response during Pulsar partition | Controller outbound resilience | bbolt queue state, reconnect/flush loop, destination topic |
| Duplicate telemetry event | Expected at-least-once delivery | Consumer idempotency and fanout crash/redelivery evidence |
| Late subscriber sees no initial values | TCS path-value state | Active policy coverage, stored matching path values, initial-sync send |
| Standby also touches device | HA transition / split-brain window | Pulsar ownership events, watchdog threshold, drain completion |
| TCS API returns UNIMPLEMENTED | Feature registration | Snowflake/IEBus enablement; ELK is intentionally a stub |

Retryable versus non-retryable: transient Pulsar or network failures, temporary downstream unavailability, and some DB failures may be retried or redelivered. Invalid input, unsupported capability, bad identifiers, and deterministic authorization or credential failures normally require correction, not repetition. Inspect the component-specific classification rather than assuming every timeout is safe to retry.

## Failure Modes

Qualitative FMEA. Criticality is qualitative because occurrence and detection rates need production evidence; "automatic" means a recovery path exists in current code, not that every outage is invisible to callers. Do not compute an RPN from this table without real production numbers.

| Failure mode | Effect / blast radius | Detection or containment | Recovery / evidence | Criticality |
|---|---|---|---|---|
| Caller fails before publish | One operation never enters SBI | Caller activity error, no broker message | Retry only if the caller op is idempotent; prove publish by correlation ID | Medium |
| Malformed or unsupported Protobuf command | Rejected before device I/O | Deserialization/capability error | Correct caller contract; send caller-facing error, do not blind-retry | Medium |
| Pulsar unavailable to caller or Message Bus | New work cannot route; many devices/sites affected | Client errors, reconnect metrics, backlog gap | Automatic client reconnect/redelivery; validate broker and publisher health | Critical |
| Message Bus instance crashes | Temporary routing loss | K8s health, Pulsar consumer ownership | `Shared` subscription lets another instance consume; verify backlog drains | High |
| Missing or stale device→controller mapping | A device or class is unroutable/misrouted | Lookup failure or wrong-controller response | Reconcile inventory to orchestrator to DB/config topic; not solved by retry | High |
| Active controller pod crashes | Assigned devices lose executor and live sessions | Pulsar consumer loss, K8s restart, connection metrics | `Failover` promotes standby automatically; prove ownership and a successful command | Critical |
| Active controller loses Pulsar | Risk of serving while ownership is uncertain | HA watchdog broker-reachability threshold | Automatic step-down/drain; self pipeline may enter disconnected local serving | Critical |
| Whole DCU / site outage | All locally assigned paths and support services fail | Cluster/site observability, missing heartbeats | Peer DCU takes ownership only if reachability, allocation, and capacity allow | Critical |
| Unsafe overlap / split-brain window | Two writers may touch the same device | Conflicting ownership events, duplicate sessions | Failover ownership, priority, and drain reduce risk; investigate every overlap | Critical |
| One device unreachable | That device's commands/telemetry only | Protocol error, gNMI disconnect | Automatic gNMI reconnect; request paths error; inspect device and mgmt network | Medium |
| Shared management network outage | Many devices in a segment fail together | Correlated protocol disconnects | Network recovery; do not treat as independent device failures | Critical |
| Secrets provider unavailable | New authenticated sessions/config rendering fail | Resolver/provider error; fail closed | Restore provider/local source; never substitute or expose values | High |
| Wrong/rotated credential | Auth fails for a device/site set | Device AAA/authentication error | Correct/rotate secret; correlate by CLLI/category/version | High |
| Config topic unavailable at boot | Controller may lack allocation/policy | Pulsar config connection failure | Load PVC snapshot where available, reconcile after reconnect | High |
| Outbound publish fails | Device action may complete but caller sees no response | Publish error, local persistent queue | Persist in bbolt, flush automatically after reconnect; caller handles duplicates | High |
| TCS fanout fails mid-delivery | Telemetry delayed or duplicated | Consumer redelivery/backlog | At-least-once recovery; downstream must dedupe or be idempotent | High |
| Response arrives before workflow listener | Workflow could wait forever | Pending-response state | Match after registration; TTL cleans orphaned pending responses | Medium |
| FileStore daemon/storage/publisher failure | Image or support-bundle jobs stall; commands unaffected | Job status, HTTP/storage errors, disk metrics | Replica/job retry after recovery; verify integrity before device use | High |

Failover verdict: implemented and normally automatic. Pulsar selects the active failover consumer and controller callbacks manage promotion and demotion. Manual action is still needed when the cause is stale mapping, insufficient peer capacity, network reachability, provider failure, or an unsafe split-brain symptom.

## Prevention

- To turn this qualitative FMEA into a numeric one, add production occurrence, time-to-detect, time-to-recover, retry success, affected-device count, and operator escape rate, then score severity, occurrence, and detection with owners. Do not invent RPN values from architecture alone.
- Reconcile inventory to orchestrator to config topic on a schedule, since stale mapping is a high-criticality mode that request retries never fix.
- Make downstream telemetry consumers idempotent, since at-least-once delivery guarantees duplicates will occur.

## Source

- Primary: the "SBI & TCS — Living Knowledge Wiki" HTML knowledge doc (author: vvainer@drivenets.com), sessions 8 and 16, synthesizing the DAP ToI session of 20 May 2026.
- Verified against the repo: [`message-bus`](https://github.com/drivenets/dap/tree/develop/dap-workspace/apps/sbi/cloud/message-bus) and [`sbi-controller`](https://github.com/drivenets/dap/tree/develop/dap-workspace/apps/sbi/edge/sbi-controller) in [drivenets/dap](https://github.com/drivenets/dap), repository state of 17 Aug 2026.

## See also

- [[sbi-control-data-plane]]: the request path this triage walks
- [[sbi-ha-and-failover]]: the failover and split-brain mechanics behind the HA rows
- [[tcs-subscriptions-and-wait-listen]]: the at-least-once fanout behind the duplicate-event rows
- [[sbi-secrets-and-vault]]: the fail-closed behavior behind the secrets rows
