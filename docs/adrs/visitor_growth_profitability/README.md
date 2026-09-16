# Architecture Decision Records

Von Digitalis Estates, O'Reilly Architectural Katas 2026.

Scope: growing visitor numbers, increasing profitability, and driving repeat visits, on an estate that needs to go from 5,000 to 15,000 daily visitors within three years without compromising the trust of the families who visit.

## Template

Every record here follows the same structure: Status, Context, Decision, Diagram, Alternatives Considered, Consequences and Tradeoffs, Conclusion.

The brief asks for trade-off analysis, so the Alternatives and Consequences sections carry the real weight. Every rejected option names the specific reason it lost, and every consequence names what we gave up, not just what we gained. Two of these records are explicitly "why not" decisions, refusing a capability that would have made more money in the short term, because that refusal is itself an architectural decision worth documenting as carefully as anything we chose to build.

## The records

| Number | Decision | The tradeoff |
|---|---|---|
| [001](001-adr-pricing-and-investment-advisor.md) | A classical ML model scores demand and opportunity, a separate LLM call drafts the rationale, and only a deterministic pricing engine can actually change a live price, within pre-set floors and ceilings. | Slower than fully automated pricing, and two models to maintain instead of one, in exchange for every proposal being explainable and incapable of touching money on its own. |
| [002](002-adr-refusal-of-individualised-pricing.md) | Pricing varies by ticket type, time, and demand only. Individual-level data is never given to the pricing path at all. | Real revenue left on the table, in exchange for staying clearly on the right side of the EU AI Act, the Digital Fairness Act, and family visitors' trust. |
| [003](003-adr-consent-gated-personalisation.md) | Return-visit recommendations run only on consented, approved signals (ticket-scan history, stated preferences), delivered through Guide, and are suppressible at any time. | Reach is limited to visitors who opt in, and the signal is narrower than unconstrained tracking would give, in exchange for an actual, trustworthy mechanism for repeat visits. |
| [004](004-adr-growth-measured-by-experiments.md) | Every pricing, upsell, or retention change states its success criteria up front and gets a held-out comparison where one is possible; where it isn't, the weaker before/after evidence is labelled as such. | More upfront design work per change, and small real effects get reported as inconclusive rather than proven, in exchange for growth claims that survive scrutiny instead of resting on a lucky week. |

The gateway, model selection, guardrails, and estate-wide production monitoring are recorded in [docs/adrs/platform](../platform) rather than here (see ADR-AI-001, ADR-AI-002, and ADR-AI-003). Offline-verifiable ticketing, which the repeat-visit signal in [003](003-adr-consent-gated-personalisation.md) depends on, is recorded in [platform/ADR-002](../platform/ADR-002-offline-ticket-signing.md).

## The through lines

Three ideas recur across these records.

1. **AI can propose, never act on money.** The Advisor in [001](001-adr-pricing-and-investment-advisor.md) only ever emits a cited proposal; the Deterministic Pricing Engine is the only thing that can change a live price, and investment decisions always reach a person.
2. **The most profitable option isn't always the one we build.** [002](002-adr-refusal-of-individualised-pricing.md) refuses individualised pricing specifically because it would work, and documents that refusal with the same rigor as any decision we did make.
3. **A good week doesn't prove anything.** [004](004-adr-growth-measured-by-experiments.md) exists because weather, holidays, and seasonal swings can make almost any change look like it worked, so every claim of success has to survive a real comparison first.

## Open questions

- What the actual floors and ceilings should be for the Deterministic Pricing Engine, and who on Ops/Finance owns keeping them current.
- What minimum effect size is actually worth acting on, given the estate's visitor volume, before [004](004-adr-growth-measured-by-experiments.md)'s experiment design can be fully specified.
- What non-price rewards ([003](003-adr-consent-gated-personalisation.md)) are worth offering a returning visitor, and whether early access or content is more effective than a flat discount.
- How consent is captured at first purchase, before there's a returning-visitor relationship for Guide to build on.
