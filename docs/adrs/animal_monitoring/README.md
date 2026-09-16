# Architecture Decision Records

Von Digitalis Estates, O'Reilly Architectural Katas 2026.

Scope: the animal care monitoring problem, how much and how well the animals eat, tracking animal health, and the piranha population count, across more than 200 animals in 55 enclosures on an estate with patchy WiFi and a finite hardware budget.

## Template

Every record here follows the same structure: Status, Context, Decision, Diagram, Alternatives Considered, Consequences and Tradeoffs, Conclusion.

The brief asks for trade-off analysis, so the Alternatives and Consequences sections carry the real weight. Every rejected option names the specific reason it lost, and every consequence names what we gave up, not just what we gained.

## The records

### Infrastructure and where things run

| Number | Decision | The tradeoff |
|---|---|---|
| [001](001-adr-edge-first-event-driven-architecture.md) | Edge first, event driven, with the hub as system of record. One MQTT bus, append-only events, derived read models. | Real infrastructure on the estate and permanent eventual consistency, in exchange for welfare detection that doesn't depend on the network. |
| [002](002-adr-sensor-connectivity-lorawan-not-wifi.md) | LoRaWAN for battery sensors, Ethernet where power exists, never WiFi. | Tiny bandwidth and camera placement dictated by cabling, in exchange for coverage and years of battery life. |

Store and forward across the edge-to-cloud boundary is an estate-wide contract shared with ticketing and crowd analytics, and is recorded in [ADR-001](../platform/ADR-001-store-and-forward-mqtt%20.md) rather than here.

### Feeding and health

Problem 1 asks how much and how well the animals eat. Problem 2 asks how we track their health. Both are answered by the same pipeline, so these three records are stages of that one pipeline rather than one record per problem: measure, then learn what normal looks like and spot drift from it, then tell a person and learn from what they say.

| Number | Step | Decision | The tradeoff |
|---|---|---|---|
| [003](003-adr-animal-feeding-and-health-measuring.md) | Measure | Non-invasive sensing layered per species, intake derived from a load cell, attribution by RFID that reports its own confidence level, and a quality gate plus calibration register before any baseline. | Only enclosure-level intake for shared bowls, and an uneven picture during phasing, because a bad reading that reaches a baseline becomes a bad definition of normal. |
| [004](004-adr-animal-feeding-and-health-learning-normal.md) | Learn normal and spot drift | Rhythm configured per species and proposed by retrieval with citations, normal learned per individual, plain rules shipping first and staying, persistence required before firing, and corroboration to escalate. | A deliberate delay of a day or two, and a blind first month for new arrivals, because a system that cries wolf gets switched off. |
| [005](005-adr-animal-feeding-and-health-alerting.md) | Tell a person | Three alert levels routed to a named owner, one plain sentence with its chart and three buttons, a published precision target, a hard ceiling on alerts per shift, and a verdict that's a required workflow step. | Permanent friction on every alert and uneven label quality, because an alert nobody acts on has the same lead time as no alert. |

### The piranhas

| Number | Decision | The tradeoff |
|---|---|---|
| [006](006-adr-edge-vision-for-piranha-counting.md) | Edge vision on a small tuned model, one still per camera every 15 minutes, taking the maximum of a burst. Includes the cost analysis. | A permanently low count, because the sampling rate is the real cost driver and keeper annotation is the real bill. |
| [007](007-adr-piranha-population-as-a-range.md) | Population reported as a posterior fused from the manual count, food per fish, and camera trend, always as a range with a direction. | More model complexity and no single satisfying number, in exchange for an estimate keepers still believe when they count by hand. |

### Knowledge and retrieval

The gateway, model selection, guardrails, evals, and production monitoring are estate-wide decisions and live in [docs/adrs/platform](../platform) instead (see ADR-AI-001, ADR-AI-002, and ADR-AI-003). What follows here is the part that's specific to animal care.

| Number | Decision | The tradeoff |
|---|---|---|
| [008](008-adr-rag-over-a-curated-corpus.md) | Retrieval over a closed, curated corpus with tracked provenance and mandatory citations. Clinical questions are blocked. | Permanent expert curation and patchy long-tail coverage, in exchange for answers we can date, cite, and correct. |
| [009](009-adr-retrieval-index-design.md) | Structure-aware chunking, metadata filtering, hybrid keyword and dense search, and a small self-hosted store with a pinned embedding model. | Extraction work per document format and a bet that the corpus stays small, because keepers search by enclosure number and species name. |

Retrieval isn't a separate feature here. It serves animal health at three points: proposing the species rhythm a vet signs off in [004](004-adr-animal-feeding-and-health-learning-normal.md), offering species context beside an alert card in [005](005-adr-animal-feeding-and-health-alerting.md), and answering a keeper's husbandry question.

## Supporting documents

| Document | What it covers |
|---|---|
| [C4 diagrams: Animal Care Monitoring](c4-diagrams-animal-care-monitoring.md) | System context, container, and two component-level deep-dives (where the model is called, how normal is learned and drift becomes an alert) for the measure/learn/alert pipeline (001-005, 008-009). |
| [C4 diagrams: Piranha Population Count](c4-diagrams-piranha-population-count.md) | The same four C4 levels for the separate piranha counting pipeline (001, 005-007), which shares only the message bus, the hub, and the keeper app with animal care monitoring. |

## The through lines

Four ideas recur across these records, and they're what makes this feel like a set rather than a list.

1. **The network is assumed absent.** Every animal-care path terminates on the estate. This is what drives the edge-first architecture, the sensor transport choice, store-and-forward, the narrow cloud role, the offline keeper app, and the alert delivery path.
2. **Honesty about uncertainty is a design constraint, not a disclaimer.** The population has no single scalar form, intake carries the confidence level it was measured at, the camera count is never published alone, and every generated claim carries a citation code can actually resolve.
3. **Controls are enforced, not requested.** Clinical questions are blocked before retrieval even runs, rather than just discouraged in a prompt. A degraded instrument may raise an alert but never updates a baseline. Retrieval proposes a species rhythm that only a vet can apply.
4. **Nothing is measured that nobody acts on.** Every alert has a named owner and can't close without a verdict, and that verdict is what tunes thresholds, corrects baselines, and proves whether any of this actually shortened the lead time to a vet visit.

## Open questions

- How many piranhas there actually are today, and how often the tanks get serviced, since that sets how often we get ground truth.
- Whether the estate will fund calibration labour. If not, we'd reduce the number of chemistry probes and accept less warning time.
- Whether a vet will help label footage, confirm which species can safely carry a tag, and review the retrieved species rhythms.
- Stereo vision for the piranha count, deferred pending measurement of how much error the current approach actually leaves.
