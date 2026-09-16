## Architectural "-ility"

The worksheet (shown in the [README](../README.md#2-architectural--ility)) is a standard [Architecture Characteristics Worksheet](diagrams/characteristics.png) (Mark Richards / DeveloperToArchitect.com), filled in for **Von Digitalis Estates – AI-Assisted Platform (Warden)**. It has four sections: **Top 3 Driving**, the full **Driving Characteristics** list (up to 7), **Implicit Characteristics**, and **Others Considered**, and this page maps every entry on it to a concrete scenario and the architectural response that satisfies it.

Priority key: **H** = must hold or the submission fails · **M** = strongly desired · **L** = nice to have.

### Driving characteristics (worksheet order)

All 7 candidates from the worksheet, in the order they appear on it. ⭐ marks the top 3 we designed around first, everything else on the worksheet exists to protect these.

| Top 3 | Pri | Characteristic | Concrete scenario | Architectural response |
|---|---|---|---|---|
| ⭐ | **H** | **Availability** | WiFi drops at a turnstile at peak; visitors must still get in | Offline-verifiable signed tickets + local redemption ledger + MQTT store-and-forward ([platform/ADR-001](adrs/platform/ADR-001-store-and-forward-mqtt%20.md), [platform/ADR-002](adrs/platform/ADR-002-offline-ticket-signing.md)) |
| ⭐ | **H** | **Scalability** | 3× visitor growth (5,000 → 15,000/day) over three years | Service-based core + event-driven edge + CQRS read models ([platform/ADR-004](adrs/platform/ADR-004-architecture-style.md)) |
| ⭐ | **H** | **Reliability** | An AI "helpfully" discounts or opens a gate | **Two-plane model**: AI can only *propose*; a deterministic gate decides ([platform/ADR-003](adrs/platform/ADR-003-two-plane-safety-model.md), README §3.1) |
| | **H** | **Observability** | "Is the AI working right now?" must be answerable | Advisory telemetry taps into the Eval harness; immutable Decision & Audit Log ([platform/ADR-AI-003](adrs/platform/ADR-AI-003-production-monitoring.md)) |
| | **H** | **Data integrity** | A screenshotted or duplicated QR is presented at the gate | Ed25519 signatures + local double-spend ledger; access decisions are deterministic, never AI ([platform/ADR-002](adrs/platform/ADR-002-offline-ticket-signing.md), [ticketing_and_access_control/002](adrs/ticketing_and_access_control/002-adr-revocation-deny-list.md)) |
| | **M** | **Recoverability** | Best model/provider changes, raises prices, or shuts down mid-operation | Augur provider-agnostic gateway: routing, caching, fallback, budget guard ([platform/ADR-AI-001](adrs/platform/ADR-AI-001-provider-and-model-portability.md)) |
| | **M** | **Testability** | GenAI answer or welfare model silently degrades in production | Eval & Validation harness: golden sets, drift monitoring, fitness functions, human feedback ([platform/ADR-AI-002](adrs/platform/ADR-AI-002-validation-and-verification.md)) |

### Implicit characteristics

Characteristics the worksheet flags as implicit, not chosen as top drivers, but critical enough to design for explicitly.

| Pri | Characteristic | Concrete scenario | Architectural response |
|---|---|---|---|
| **H** | **Security** | Faces / behaviour captured on CCTV-style CV | Edge Anonymiser (no faces leave the estate) + consent service; EU AI Act / Digital Fairness Act ([platform/ADR-006](adrs/platform/ADR-006-edge-anonymiser.md), [visitor_growth_profitability/002](adrs/visitor_growth_profitability/002-adr-refusal-of-individualised-pricing.md)) |
| **M** | **Feasibility (cost/time)** | Edge hardware and LLM spend must stay within a modest estate budget | Cheap MQTT sensors + edge inference where it pays; cloud LLM only through Augur's cache/budget guard ([platform/ADR-AI-001](adrs/platform/ADR-AI-001-provider-and-model-portability.md), [platform/ADR-005](adrs/platform/ADR-005-edge-vs-cloud-cv-placement.md)) |
| **M** | **Maintainability** | New attractions/enclosures added later | New edge nodes publish to MQTT; new advisory models plug into Augur without touching the core |

### Others considered

Named on the worksheet but not treated as driving or implicit, included for completeness and due diligence.

| Pri | Characteristic | Concrete scenario | Architectural response |
|---|---|---|---|
| **M** | **Configurability** | Pricing must flex by segment/time/demand without exploiting or profiling individuals | Deterministic pricing engine with pre-set floors/ceilings; individualised/profiling-based pricing **deliberately rejected** ([visitor_growth_profitability/002](adrs/visitor_growth_profitability/002-adr-refusal-of-individualised-pricing.md)) |
| **L** | **Performance** | A turnstile must validate a ticket in under a second, even fully offline | Offline Ed25519 signature verification at the edge, no round-trip required ([platform/ADR-002](adrs/platform/ADR-002-offline-ticket-signing.md)) |

<p align="right"><a href="../README.md">↑ Back to README</a></p>
