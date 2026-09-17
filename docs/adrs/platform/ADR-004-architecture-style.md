# ADR-004: Architecture Style, Service-Based Core with Event-Driven Edge and CQRS

## Status
Accepted

## Date
16 September 2026

## Context
- The estate is a small operation today (5,000 visitors/day) that has to scale to 15,000/day within three years, with a small ops and engineering team throughout, no sudden hiring wave is coming with it.
- The edge (turnstiles, sensors, cameras across 55 enclosures and 40 rides) produces a constant stream of small events over a patchy network, which is a fundamentally different problem from the transactional core (ticketing, payments, pricing) that needs strong consistency.
- The same underlying events (a ticket scan, an occupancy count) are read very differently by different consumers: a keeper wants "is this animal okay right now," a manager wants "what's the monthly trend," and a model wants "give me a clean training set." Forcing one data shape to serve all three is where systems either get slow or get hacked around.
- A large microservices estate is a good answer to a large team and a large, differentiated scaling problem. Neither is true here yet.

## Decision

| Aspect | Decision | Rationale |
|---|---|---|
| **Core style** | A service-based transactional core (not a microservices mesh): a small number of coarse-grained services (ticketing/admissions, pricing, access control, welfare, footfall analytics) each owning their own data. | Matches the estate's team size, gets clear ownership boundaries without paying for a distributed system the current scale doesn't need. |
| **Edge style** | Event-driven at the edge: every device publishes onto MQTT, with store-and-forward buffering (`ADR-001`) so the patchy network never blocks a scan or a reading. | The edge's problem is unreliable connectivity and high event volume, not transactional consistency, so it gets a different architecture than the core. |
| **Read side** | CQRS: read models (occupancy views, dwell views, training tables, dashboards) are derived projections off the event log, never the system of record themselves. | Lets each consumer get the shape of data it actually needs, without the write side having to compromise its own model to serve every reader. |
| **Growth path** | Scaling from 5k to 15k visitors/day is handled by scaling read models and edge nodes horizontally, not by re-architecting the core services. | The pattern is chosen up front specifically because it has to survive 3x growth without a rewrite partway through. |

## Diagram

![ADR-004: Architecture Style, Service-Based Core with Event-Driven Edge and CQRS](../../diagrams/adrs/platform-adr-004-architecture-style.svg)

| Symbol | Meaning |
|---|---|
| Green | The event-driven edge, optimised for unreliable connectivity and volume. |
| Blue | The service-based transactional core, optimised for consistency and clear ownership. |
| Gray | CQRS read models, one shape of data per consumer, all derived from the same event log. |

## Alternatives Considered

| Alternative | Why Rejected |
|---|---|
| Full microservices, one service per fine-grained capability | Coordination and operational cost that a small ops/engineering team can't absorb, for a scaling problem the estate doesn't actually have yet at 5k-15k visitors/day. |
| A single monolith for both edge and core | The edge's problem (unreliable connectivity, high event volume, no strong consistency needed) and the core's problem (strong consistency for money and access) are different enough that one architecture style serving both would compromise one of them. |
| One shared read model for every consumer | Forces a keeper's "is this animal okay right now" view, a manager's monthly trend, and a model's training set into the same shape, which either under-serves all three or turns into ad hoc, uncoordinated workarounds per consumer. |
| Design for a larger future team and scale now | Over-building for a team and a scale that don't exist yet costs real time today, against a hard deadline, for a benefit that may never be needed if growth doesn't reach projections. |

## Consequences and Tradeoffs

| Type | Point | Detail |
|---|---|---|
| ✅ Positive | Matches the team that has to run it | A small number of coarse-grained services is operable by a small team, without the coordination tax of a large microservices estate. |
| ✅ Positive | The edge and the core each get the architecture their actual problem needs | Instead of one style compromising to fit both. |
| ✅ Positive | Growth is absorbed by scaling read models and edge nodes | Not by re-architecting the transactional core partway through the estate's growth. |
| ⚠️ Trade-off | Read models are eventually consistent | A dashboard or a training table can lag the write side by seconds to minutes; this is acceptable everywhere it's used because no read model in this architecture drives a real-time money or access decision. |
| ⚠️ Trade-off | Coarse-grained services still need internal discipline | Nothing stops a coarse-grained service from becoming an undisciplined monolith over time; the module-boundary approach in `ticketing_and_access_control/001` is one example of how that's kept in check. |
| ⚠️ Trade-off | This is a bet against the estate outgrowing 15k/day sooner than planned | If growth wildly exceeds the three-year target, some of these services would need to be split further; that's a real but deliberately deferred cost. |

## Conclusion
A service-based core for the transactional plane, an event-driven edge with store-and-forward buffering, and CQRS read models on top, chosen specifically because it fits a small team, survives a patchy network, and absorbs 3x growth by scaling out rather than by rewriting the core.
