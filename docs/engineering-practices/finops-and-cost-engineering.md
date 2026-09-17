# Engineering Practice: FinOps and Cost Engineering

**Operationalises:** [platform/ADR-007](../adrs/platform/ADR-007-cost-model-and-investment-strategy.md), [platform/ADR-AI-001](../adrs/platform/ADR-AI-001-provider-and-model-portability.md), [platform/ADR-005](../adrs/platform/ADR-005-edge-vs-cloud-cv-placement.md), [animal_monitoring/006](../adrs/animal_monitoring/006-adr-edge-vision-for-piranha-counting.md)

## Why this practice exists for this estate

The gnome business is defunct and the Countess is funding this from a fixed budget. [ADR-007](../adrs/platform/ADR-007-cost-model-and-investment-strategy.md) sets the shape, roughly $28,500 CapEx against $10,000 to $11,800 a year of OpEx, and it is explicit that those figures are illustrative placeholders. This practice is how the structure survives contact with real quotes and real growth.

The headline finding worth defending: **the largest recurring cost is keeper and vet labour, not compute.** Annotation and calibration time is budgeted as its own line, never hidden inside "AI cost."

## What we do

| Practice | Rule |
|---|---|
| **Every cost sits on a named line with an owner** | CapEx at the edge, cloud OpEx, LLM spend through Augur, labour. Four lines, four owners. Anything that does not fit one is a new line, not a rounding error. |
| **Sampling rate is treated as the cost lever it is** | [animal_monitoring/006](../adrs/animal_monitoring/006-adr-edge-vision-for-piranha-counting.md) shows how far this swings: 15-minute stills versus video is three orders of magnitude. Any new CV capability states its sampling rate in the proposal. |
| **Spend ceilings are a routing policy, not a budget alarm** | Augur's tiered degradation at the ceiling is set in [ADR-AI-001](../adrs/platform/ADR-AI-001-provider-and-model-portability.md). FinOps owns the ceiling's value and reviews it quarterly; it does not re-decide the behaviour. |
| **Unit economics, not just totals** | Cost per 1,000 visitors, per instrumented enclosure, and per AI capability. A total that grows with the 3x visitor target is fine; a unit cost that grows is a design problem. |
| **CapEx must retire an OpEx line to qualify** | Edge hardware is funded on the recurring cloud bill it removes. If it removes nothing recurring, it is judged as a plain purchase. |
| **Illustrative figures are re-quoted before purchase** | No wave of hardware is bought against a placeholder. Quotes replace estimates wave by wave, following the three funding waves in [animal_monitoring/003](../adrs/animal_monitoring/003-adr-animal-feeding-and-health-measuring.md). |
| **Monthly review, quarterly re-forecast** | The monthly review reads the four lines against forecast. The quarterly re-forecast updates assumptions, including the agent-call volume [footfall/001](../adrs/footfall_and_popularity/001-adr-crowd-and-popularity-analytics.md) flags as needing rework once more consumers use the MCP server. |

## How we know it's working

| Signal | Target |
|---|---|
| LLM spend against the tiered ceiling | Inside ceiling; a breach degrades by tier rather than surprising Finance |
| Labour line visible and separately tracked | Always; it is the largest OpEx line |
| Unit cost per 1,000 visitors | Flat or falling as volume grows toward 15,000/day |
| Hardware bought against a placeholder figure | Zero |
| Capabilities with no attributable cost | Zero |

## What we deliberately don't do

| We don't | Why |
|---|---|
| Argue for the edge on price alone | [animal_monitoring/006](../adrs/animal_monitoring/006-adr-edge-vision-for-piranha-counting.md) says plainly that cloud inference at our sampling rate would be affordable. The real reasons are bandwidth and accuracy, and we should not lean on an argument we cannot defend |
| Claim a saving the brief does not support | [ADR-007](../adrs/platform/ADR-007-cost-model-and-investment-strategy.md) refuses to invent a figure nobody could audit. The case is the failure modes closed, not a projected return |
| Let welfare calls stop at a budget ceiling | A cost control that can silently stop a welfare alert is not a cost control |
| Hide annotation time inside model cost | It is the scarcest resource on the estate and the line most often left off an AI estimate |

<p align="right"><a href="../../README.md">↑ Back to README</a></p>
