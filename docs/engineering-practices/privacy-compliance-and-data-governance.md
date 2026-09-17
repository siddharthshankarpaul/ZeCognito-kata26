# Engineering Practice: Privacy, Compliance and Data Governance

**Operationalises:** [platform/ADR-005](../adrs/platform/ADR-005-edge-vs-cloud-cv-placement.md), [platform/ADR-006](../adrs/platform/ADR-006-edge-anonymiser.md), [platform/ADR-AI-002](../adrs/platform/ADR-AI-002-validation-and-verification.md), [platform/ADR-AI-003](../adrs/platform/ADR-AI-003-production-monitoring.md), [growth/002](../adrs/visitor_growth_profitability/002-adr-refusal-of-individualised-pricing.md), [growth/003](../adrs/visitor_growth_profitability/003-adr-consent-gated-personalisation.md)

## Why this practice exists for this estate

The estate points cameras at places where visitors and animals both appear, and it wants more repeat visits. Those two facts put it squarely inside the EU AI Act and the Digital Fairness Act. Four ADRs each draw part of the boundary; this practice is where they are enforced as one thing rather than four.

## What we do

| Practice | Rule |
|---|---|
| **Classify before you build** | Every new data flow is classified at design time: anonymous, pseudonymous, or identifying. An unclassified flow does not pass review. |
| **Anonymise before egress** | Identity is stripped on-device, before publish ([ADR-006](../adrs/platform/ADR-006-edge-anonymiser.md)). Only anonymised data crosses the estate boundary ([ADR-005](../adrs/platform/ADR-005-edge-vs-cloud-cv-placement.md)). No raw frame leaves, for footfall or welfare. |
| **Consent is checked, not stored and forgotten** | Consent is its own record, verified before every use and withdrawable as easily as it was given ([growth/003](../adrs/visitor_growth_profitability/003-adr-consent-gated-personalisation.md)). Absence of consent is a hard stop. |
| **Pseudonymise at the prompt boundary** | Visitor IDs are pseudonymised before reaching any model, and deletion resolves the pseudonym ([ADR-AI-003](../adrs/platform/ADR-AI-003-production-monitoring.md)). Staff names are stripped from fact bundles ([ADR-AI-002](../adrs/platform/ADR-AI-002-validation-and-verification.md)). |
| **Retention is tiered and enforced in code** | Traces with prompts and retrieved context 90 days; verdicts and versions 13 months; any AI-influenced financial or safety decision 7 years as a decision record. |
| **Refusals are documented as decisions** | Individualised and profiling-based pricing is refused outright ([growth/002](../adrs/visitor_growth_profitability/002-adr-refusal-of-individualised-pricing.md)). Pricing varies by segment, time and demand, never by who you are. |
| **A DPIA per camera deployment** | Each new camera placement records purpose, what it captures incidentally, and why a less invasive option was rejected, before it is installed. |

## How we know it's working

| Signal | Target |
|---|---|
| Identifiable frames leaving the estate | Zero, enforced at the Anonymiser |
| Personalised contact without current consent | Zero, checked per use |
| Data flows in production with no classification | Zero |
| Deletion requests resolved through to pseudonyms | 100%, within the statutory window |
| Records past their retention tier | Zero, swept automatically |

## What we deliberately don't do

| We don't | Why |
|---|---|
| Price by individual, or profile to price | Refused on EU AI Act and Digital Fairness Act grounds, not deferred |
| Anonymise only footfall cameras | Welfare cameras catch visitors incidentally; there is no principled reason they deserve less protection |
| Treat consent as a purchase checkbox | A box ticked once and never re-checked is not consent, it is a record of one afternoon |
| Keep model traces indefinitely because storage is cheap | They carry prompts and retrieved context, which is exactly what should not sit around |

<p align="right"><a href="../../README.md">↑ Back to README</a></p>
