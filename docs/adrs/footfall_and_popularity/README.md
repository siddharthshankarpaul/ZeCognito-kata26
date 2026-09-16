# Architecture Decision Records

Von Digitalis Estates, O'Reilly Architectural Katas 2026.

Scope: the footfall and popularity problem, understanding which parts of the park are busy, forecasting queues and dwell time, and giving ops a staffing recommendation, on an estate with patchy WiFi and a finite hardware budget shared with animal care and ticketing.

## Template

Every record here follows the same structure: Status, Context, Decision, Diagram, Alternatives Considered, Consequences and Tradeoffs, Conclusion. The two supporting documents (C4 diagrams and the confidence calibration note) are implementation detail, not decisions, and can change without reopening the ADRs they support.

The brief asks for trade-off analysis, so the Alternatives and Consequences sections carry the real weight. Every rejected option names the specific reason it lost, and every consequence names what we gave up, not just what we gained.

## The records

| Number | Decision | The tradeoff |
|---|---|---|
| [001](001-adr-crowd-and-popularity-analytics.md) | Beam/IR counters park-wide as the default, computer vision added only in zones proven dense by the data, a lightweight forecast, and one read-only advisory Queue Management Agent behind an MCP server. | Beam counters stay weak in dense crowds until CV is justified there, and a few minutes of staleness is baked in, in exchange for an honest, cheap, privacy-safe count and no AI anywhere near money or access. |
| [002](002-adr-phased-rollout-crowd-and-popularity-analytics.md) | Three build phases, each gated on visitor volume rather than a calendar date: ground truth and platform interface, then cross-zone prediction, then platform reuse for pricing. | The build order is fixed by dependency, so a team eager to start a later phase has to wait for the earlier one's evidence to exist first. |

## Supporting documents

| Document | What it covers |
|---|---|
| [C4 diagrams](c4-diagrams.md) | System context, container, and two component-level deep-dives (the agent/MCP boundary, and the confidence calibration mechanism). |
| [Confidence calibration design note](confidence-calibration-design-note.md) | How the illustrative 0.6 confidence threshold becomes a real, per-zone number: ground-truth sources, fitting method, evaluation, recalibration cadence, and cold start. |

Store and forward across the edge-to-cloud boundary is an estate-wide contract shared with ticketing and animal care, and is recorded in [ADR-001](../platform/ADR-001-store-and-forward-mqtt%20.md) rather than here. The gateway, model selection, guardrails, evals, and production monitoring are also estate-wide decisions, recorded in [docs/adrs/platform](../platform).

## The through lines

Three ideas recur across these records.

1. **The count comes first, AI earns its place on top.** Occupancy, dwell, and throughput are solved with beam counters and arithmetic before any model is involved. Vision and forecasting are added only where the deterministic signal alone isn't good enough.
2. **AI can propose, never act.** The Queue Management Agent is read-only, has no persistent memory, and every recommendation goes through a human at Ops Review before anything real happens. No AI component ever computes, approves, or touches a monetary figure.
3. **Growth is gated on evidence, not a date.** Each build phase in [002](002-adr-phased-rollout-crowd-and-popularity-analytics.md) depends on data the previous phase produced, so the schedule follows the estate's own visitor-volume curve.

## Open questions

- Whether the chokepoint assumption behind beam/IR counting holds in open plazas and festival grounds, or only in corridors and gates.
- What the real confidence threshold should be, once enough calibration data exists to run the procedure in the [design note](confidence-calibration-design-note.md).
- What the FinOps assumptions become once the MCP server opens to more consumers than the Queue Management Agent alone.
