# Engineering Practice: Observability and Audit

**Operationalises:** [platform/ADR-008](../adrs/platform/ADR-008-decision-and-audit-log.md), [platform/ADR-003](../adrs/platform/ADR-003-two-plane-safety-model.md), [animal_monitoring/001](../adrs/animal_monitoring/001-adr-edge-first-event-driven-architecture.md), [animal_monitoring/005](../adrs/animal_monitoring/005-adr-animal-feeding-and-health-alerting.md)

> This covers **both planes**, and it is where the transactional plane, which has no AI in it at all, gets watched. It answers "what happened, who decided, and can we prove it." Whether a model is behaving right now is a different question, answered by [AIOps](aiops.md) against the signals in [ADR-AI-003](../adrs/platform/ADR-AI-003-production-monitoring.md).

## Why this practice exists for this estate

Observability is an H-priority driving characteristic, and the money path has no AI in it, so nothing in the AI practices covers a turnstile, a refund, or a redemption ledger falling behind. Three audiences want different things from the same events: the keeper wants the next action, the vet wants history, the engineer wants to know a gateway is down. [ADR-008](../adrs/platform/ADR-008-decision-and-audit-log.md) decides what the gate records; this is how the estate reads it.

## What we do

| Practice | Rule |
|---|---|
| **Three surfaces, never merged** | Keeper app for keepers, operator dashboards for engineers, reporting for managers ([005](../adrs/animal_monitoring/005-adr-animal-feeding-and-health-alerting.md)). Every new signal is assigned a surface before it is built, and keepers are never sent to a dashboard. |
| **One correlation ID across the seam** | The same ID follows a proposal from the advisory plane, through the gate, into the action and its audit record. Without it, the two-plane model is traceable in principle and not in practice. |
| **Advisory calls carry a standard trace shape** | Capability, caller, tier, pinned model version, cache hit, retrieval score, guardrail verdict, cost and latency, on every call through Augur. Same correlation ID as the decision it feeds, so an advisory trace and its gate record join up. |
| **The transactional plane has its own signals** | Offline versus online admission latency, redemption-ledger reconciliation lag, voucher-pool depth, deny-list staleness at the gate, MQTT queue depth, payment failures. None of these involve a model, and all of them can ruin a day. |
| **SLOs are written against the driving characteristics** | Availability at the turnstile, data integrity in the ledger, and the freshness of the answer to "is the AI working." A characteristic with no measured objective is an aspiration. |
| **Dashboards have owners, like signals do** | An unowned dashboard becomes wallpaper within a month. The owner is who prunes it. |
| **The audit log is read routinely, not only in an incident** | A monthly sample of decisions is replayed from the log alone. A log nobody reads until a regulator asks is a log nobody knows is broken. |
| **Replay is drilled** | Once a quarter, reconstruct one AI-influenced decision and one plain admission end to end, using only the record. The drill fails if anyone needs to consult the live system. |
| **Edge health stays an engineering concern** | Gateway coverage, ring-buffer occupancy and calibration debt belong to [Edge Fleet](edge-fleet-and-device-lifecycle.md) and surface to operators, never to a keeper on a round. |

## How we know it's working

| Signal | Target |
|---|---|
| Decisions replayable from the log alone | 100% of the monthly sample |
| Quarterly replay drill | Runs, with findings logged |
| Signals in production with no assigned surface and owner | Zero |
| Driving characteristics with no measured objective | Zero |
| Time to answer "why was this visitor refused entry" | Minutes, from the log, without reading application logs |

## What we deliberately don't do

| We don't | Why |
|---|---|
| Answer an audit question by querying the live system | If the record cannot answer it alone, the record is incomplete and we have just hidden that |
| Put a second signal in front of keepers to explain the first | Alert budget is a design constraint; explaining a bad alert with another one spends it twice |
| Build a dashboard because the data exists | Unowned panels crowd out the ones somebody is actually watching |
| Treat a missing correlation ID as a logging bug | It means a path crossed the plane boundary unobserved, which is an architecture finding |
| Investigate the audit log without leaving a read record | The read trail is part of the control, including when it is us |

<p align="right"><a href="../../README.md">↑ Back to README</a></p>
