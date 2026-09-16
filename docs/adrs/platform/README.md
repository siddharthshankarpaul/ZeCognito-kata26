# Architecture Decision Records

Von Digitalis Estates, O'Reilly Architectural Katas 2026.

Scope: the estate-wide decisions every sub-problem depends on, how the platform is shaped, how AI is kept safe, how tickets are cryptographically verified offline, and how the AI layer itself is gated, monitored, and made provider-agnostic. Sub-problem-specific ADRs (animal welfare, footfall, visitor growth, ticketing) live in their own folders and reference these rather than repeating them.

## Template

Every record here follows the same structure: Status, Context, Decision, Diagram, Alternatives Considered, Consequences and Tradeoffs, Conclusion.

The brief asks for trade-off analysis, so the Alternatives and Consequences sections carry the real weight. Every rejected option names the specific reason it lost, and every consequence names what we gave up, not just what we gained.

## The records

### Platform architecture

| Number | Decision | The tradeoff |
|---|---|---|
| [ADR-001](ADR-001-store-and-forward-mqtt%20.md) | MQTT with QoS 1 store-and-forward buffering at the edge, idempotent cloud consumers, and application-level ordering. | Permanent eventual consistency and expected duplicates, in exchange for zero data loss over patchy WiFi. |
| [ADR-002](ADR-002-offline-ticket-signing.md) | Ed25519 asymmetric ticket signing with the private key held only in cloud KMS, self-contained signed payloads, family passes as an N-admit token, and a pre-signed voucher pool for offline gate sales. | A finite voucher pool and rotation-around-connectivity planning, in exchange for turnstile verification that never depends on the network. |
| [ADR-003](ADR-003-two-plane-safety-model.md) | Every AI component lives in a non-deterministic Advisory plane that can only propose; only the Deterministic Decision Gate can turn a proposal into a real action. | Mandatory friction on every AI-influenced action, in exchange for a bounded blast radius on every AI mistake, estate-wide. |
| [ADR-004](ADR-004-architecture-style.md) | A service-based transactional core, an event-driven MQTT edge, and CQRS read models, not a microservices mesh. | Eventually-consistent read models and a bet against outgrowing 15k/day sooner than planned, in exchange for an architecture a small team can actually run. |
| [ADR-005](ADR-005-edge-vs-cloud-cv-placement.md) | Latency/privacy/cost-sensitive computer vision (counting, safety) runs on the edge; heavier batch analysis runs in the cloud on already-anonymised data. | Per-deployment edge hardware and tuning cost, in exchange for counting and safety signals that work with no network dependency. |
| [ADR-006](ADR-006-edge-anonymiser.md) | Identity is stripped from every camera frame at the edge, on-device, before anything is published. | Compute cost on every camera deployment, in exchange for no raw or identifiable frame ever leaving the estate. |
| [ADR-007](ADR-007-cost-model-and-investment-strategy.md) | Push spend into one-time CapEx at the edge wherever it removes a recurring cloud bill; keep AI OpEx on one visible, tiered line through Augur. | Capital paid up front, before 3x growth shows up in revenue, in exchange for a cost structure that doesn't surprise Finance mid-month. |

### The AI layer

Together, [ADR-AI-002](ADR-AI-002-validation-and-verification.md) and [ADR-AI-003](ADR-AI-003-production-monitoring.md) are this platform's MLOps (what's safe to ship) and AIOps (what's safe to keep running) layers; they just aren't named that anywhere else in this repo.

| Number | Decision | The tradeoff |
|---|---|---|
| [ADR-AI-001](ADR-AI-001-provider-and-model-portability.md) | Every LLM call routes through Augur, a single provider-agnostic gateway with pinned model versions, multi-provider fallback, and a self-hosted last resort. | An extra network hop on every call, in exchange for a provider outage, price hike, or shutdown becoming a config change, not a re-architecture. |
| [ADR-AI-002](ADR-AI-002-validation-and-verification.md) | Deterministic guardrails run in code on every request; evals gate every versioned artefact (prompts, models, thresholds) before it ships. | Every prompt change runs a full eval suite, in exchange for questions that could hurt someone never reaching a model at all. |
| [ADR-AI-003](ADR-AI-003-production-monitoring.md) | Every signal has a threshold, window, named responder, and response; hard signals run on minutes, statistical signals on 7-14 day windows; one fallback path, drilled quarterly. | A dashboard/alerting build that doesn't exist yet, in exchange for catching a model that's quietly gotten worse before a keeper has to ask why. |

## The through lines

Three ideas recur across these records, and every sub-problem folder inherits them rather than re-deciding them.

1. **AI can propose, never act.** [ADR-003](ADR-003-two-plane-safety-model.md) is the one rule every sub-problem's AI component is placed against, so no folder has to justify its own safety boundary from scratch.
2. **The network is assumed unreliable, the estate keeps running anyway.** [ADR-001](ADR-001-store-and-forward-mqtt%20.md) and [ADR-002](ADR-002-offline-ticket-signing.md) both exist because patchy WiFi is a constraint every sub-problem inherits, not something each one solves independently.
3. **A provider or a model can change without warning, and the platform has to survive it.** [ADR-AI-001](ADR-AI-001-provider-and-model-portability.md) and [ADR-AI-003](ADR-AI-003-production-monitoring.md) together answer the brief's explicit question about AI-provider uncertainty at the platform level, so every sub-problem's use of Augur inherits the same protection.

## Open questions

- The real cost and cadence of key rotation and voucher-pool replenishment in [ADR-002](ADR-002-offline-ticket-signing.md), once real gate-sale volume is known.
- Whether the estate will outgrow the service-based core in [ADR-004](ADR-004-architecture-style.md) sooner than the three-year, 15k/day horizon it's designed around.
- The actual per-zone confidence thresholds and calibration data referenced by [ADR-005](ADR-005-edge-vs-cloud-cv-placement.md), which depend on data that doesn't exist until the system is running.
- Whether Augur's fallback and self-hosted last resort in [ADR-AI-001](ADR-AI-001-provider-and-model-portability.md) have actually been drilled, per the quarterly cadence [ADR-AI-003](ADR-AI-003-production-monitoring.md) requires.
