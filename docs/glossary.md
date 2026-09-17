## Glossary

Every term below is used somewhere in this repo without being re-explained each time. This page is the one place to look it up. Terms are grouped, not alphabetical, so related ones sit together.

### The platform and its named services

| Term | Definition | Where it's decided |
|---|---|---|
| **Warden** | The name of the overall AI-assisted platform this repo designs, covering ticketing, footfall, animal welfare, and visitor growth. | README §3 |
| **Augur** | The single gateway every AI capability calls through: pinned model versions, multi-provider fallback, a self-hosted last resort, spend ceilings by tier. No service holds a provider credential directly. | [platform/ADR-AI-001](adrs/platform/ADR-AI-001-provider-and-model-portability.md) |
| **Ark** | The advisory service behind animal welfare monitoring: feed/health anomaly detection and the piranha population estimate. | [animal_monitoring](adrs/animal_monitoring/README.md) |
| **Lookout** | Edge computer-vision inference, running on-estate so counting and safety signals never depend on the network. | [platform/ADR-005](adrs/platform/ADR-005-edge-vs-cloud-cv-placement.md) |
| **Guide** | The GenAI concierge visitors talk to directly; also the single delivery surface for consented return-visit recommendations. | [visitor_growth_profitability/003](adrs/visitor_growth_profitability/003-adr-consent-gated-personalisation.md) |
| **Profitability Advisor** | The advisory service that proposes pricing and investment moves from aggregate signals only (zone occupancy, revenue, season), never from an individual visitor's history. | [visitor_growth_profitability/002](adrs/visitor_growth_profitability/002-adr-refusal-of-individualised-pricing.md) |
| **Queue Management Agent** | The narrow, read-only advisory agent that turns crowd counts into a staffing recommendation. | [footfall_and_popularity/001](adrs/footfall_and_popularity/001-adr-crowd-and-popularity-analytics.md) |

### The safety model

| Term | Definition | Where it's decided |
|---|---|---|
| **Two-plane model** | Every AI component lives in a non-deterministic Advisory plane that can only propose. Money and access decisions live in a separate, deterministic Transactional plane. | [platform/ADR-003](adrs/platform/ADR-003-two-plane-safety-model.md) |
| **Deterministic Decision Gate** | The one bridge between the two planes; the only place an AI proposal can turn into a real action, and always with a human or a fixed rule deciding. | [platform/ADR-003](adrs/platform/ADR-003-two-plane-safety-model.md) |
| **Guardrail** | A deterministic, in-code check applied to every AI request before or after the model runs: schema shape, citation resolution, blocklist. Needs no labelled data, so it works from the first request. | [platform/ADR-AI-002](adrs/platform/ADR-AI-002-validation-and-verification.md) |
| **Eval** | The release-time gate that decides whether a changed prompt, model, threshold, or tool tier is safe to ship, compared against the live version inside a tolerance band. | [platform/ADR-AI-002](adrs/platform/ADR-AI-002-validation-and-verification.md) |
| **Canary prompt** | A fixed prompt run daily against every provider; a changed fingerprint in the answer means a provider silently swapped the model behind a stable name, and freezes promotion until re-evaluated. | [platform/ADR-AI-003](adrs/platform/ADR-AI-003-production-monitoring.md) |
| **Calibration / ECE (expected calibration error)** | How well a model's stated confidence matches its real-world accuracy, fit per zone against independent ground truth. A new calibration table that scores worse than the live one is rejected. | [platform/ADR-AI-003](adrs/platform/ADR-AI-003-production-monitoring.md), [footfall_and_popularity/confidence-calibration-design-note](adrs/footfall_and_popularity/confidence-calibration-design-note.md) |
| **Kill switch** | A config flag, not a separate code path, that forces an AI capability (or the whole estate) into its no-model fallback. Every flip writes an audit record. | [platform/ADR-AI-003](adrs/platform/ADR-AI-003-production-monitoring.md) |
| **Fitness function** | An automated or monitored check that fails the build, or raises a named alert, the moment an architectural rule is broken (e.g. an AI service reaching the transactional plane directly). | [docs/fitness-functions.md](fitness-functions.md) |
| **RAG (retrieval-augmented generation)** | Grounding a model's answer in a curated, citable set of documents instead of relying on what the model already "knows," so every claim traces back to a source. | [animal_monitoring/008](adrs/animal_monitoring/008-adr-rag-over-a-curated-corpus.md) |

### The edge and the network

| Term | Definition | Where it's decided |
|---|---|---|
| **Store-and-forward** | The MQTT buffering pattern that lets an edge device keep working through a WiFi outage, then drains its backlog on reconnect. | [platform/ADR-001](adrs/platform/ADR-001-store-and-forward-mqtt%20.md) |
| **QoS 1** | The MQTT delivery guarantee used estate-wide: at-least-once. Duplicates are expected, not a bug, and are handled by idempotent cloud consumers. | [platform/ADR-001](adrs/platform/ADR-001-store-and-forward-mqtt%20.md) |
| **Edge Anonymiser** | Strips identity from every camera frame on-device before anything is published; no raw or identifiable frame ever leaves the estate. | [platform/ADR-006](adrs/platform/ADR-006-edge-anonymiser.md) |

### Ticketing

| Term | Definition | Where it's decided |
|---|---|---|
| **Deny list** | The small, "today only" list of passes gates check against; scoped to just what's actually revoked so it stays small no matter how many tickets the estate has ever sold. | [ticketing_and_access_control/002](adrs/ticketing_and_access_control/002-adr-revocation-deny-list.md) |
| **N-admit token** | A single signed credential (a family pass) that admits a fixed number of people, verified entirely offline at the turnstile. | [platform/ADR-002](adrs/platform/ADR-002-offline-ticket-signing.md) |

### Data and architecture patterns

| Term | Definition | Where it's decided |
|---|---|---|
| **CQRS** | Separating the write path (commands, the source of truth) from the read path (query-optimised views), so analytics reads never compete with or block a transactional write. | [platform/ADR-004](adrs/platform/ADR-004-architecture-style.md) |
| **Read model** | A denormalised, query-optimised view built off the event stream (e.g. `occupancy_view`, `dwell_view`), rebuilt from events rather than being the system of record itself. | [platform/ADR-004](adrs/platform/ADR-004-architecture-style.md), [footfall_and_popularity/001](adrs/footfall_and_popularity/001-adr-crowd-and-popularity-analytics.md) |
| **MCP (Model Context Protocol) server** | The read-only, access-controlled tool-calling surface an AI agent calls instead of a bespoke point-to-point integration; every tool is scoped per calling agent. | [footfall_and_popularity/001](adrs/footfall_and_popularity/001-adr-crowd-and-popularity-analytics.md) |

<p align="right"><a href="../README.md">↑ Back to README</a></p>
