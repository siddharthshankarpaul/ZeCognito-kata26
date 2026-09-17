# Engineering Practice: Agentic Development Lifecycle (ADLC)

**Operationalises:** [platform/ADR-003](../adrs/platform/ADR-003-two-plane-safety-model.md), [platform/ADR-AI-002](../adrs/platform/ADR-AI-002-validation-and-verification.md), [CI/CD pipeline](../warden-cicd-pipeline.md)

> **ADLC is how we *build* the Warden Platform**, not how its runtime agents behave in front of a visitor. That is [footfall/001](../adrs/footfall_and_popularity/001-adr-crowd-and-popularity-analytics.md)'s job.

## Why this practice exists for this estate

The binding constraint on delivery is not engineering hours, it is **domain expert time**. A vet costs money and is not on site; a keeper reviewing an acceptance criterion is a keeper not doing their round. Pulling either into every requirements conversation does not scale, and skipping them produces software that is confidently wrong about animals.

So: agents do analysis, implementation and first-pass review in tandem, **persona agents** stand in for scarce experts during that first pass, and a deterministic gate decides what ships without a human.

## The spine: ADLC mirrors the architecture it builds

[ADR-003](../adrs/platform/ADR-003-two-plane-safety-model.md) says AI proposes and only a deterministic gate acts. ADLC applies that rule to our own delivery process.

| In the product | In our delivery process |
|---|---|
| Advisory plane proposes | Development agents propose a change |
| Deterministic Decision Gate decides | Deterministic merge gate decides, in code |
| Humans where stakes are real | Named humans review anything classified complex |
| Immutable Decision & Audit Log | Every change traceable to its prompt, model version and ADR |

If we would not let an AI open a turnstile, we do not let one merge a change to the code that opens turnstiles.

## What we do

| Agent | Remit | Output |
|---|---|---|
| **Analyst** | Affected ADRs, affected plane, acceptance criteria, open questions | An analysis, never code |
| **Implementer** | Writes the change against an approved analysis | A diff, with tests |
| **Reviewer** | Correctness, ADR conformance, two-plane violations | Findings, not approvals |
| **Test** | Tests, including the fitness functions enforcing [ADR-003](../adrs/platform/ADR-003-two-plane-safety-model.md) | Test code |
| **Personas** | Review the *analysis* as a stakeholder would | Objections and questions |

Orchestration between them is plain deterministic code: the loop decides what happens next, the model only composes.

### Persona agents

Grounded stand-ins for the estate's roles, used to pressure-test an analysis before scarce human attention is spent on it. **Keeper** ("can I answer this in two taps with both hands full?"), **Vet** ("does this assume a rhythm nobody signed off?"), **Curator/Finance** ("what does it cost, on which line?"), **Visitor** ("is this using something I did not agree to?").

Three rules keep them honest:

1. **Personas raise objections, never approvals.** Their output is an input to the gate, not a signature.
2. **Personas are versioned artefacts** under [ADR-AI-002](../adrs/platform/ADR-AI-002-validation-and-verification.md). Changing one runs the eval suite.
3. **Personas are calibrated against the real thing.** Mandatory keeper verdicts from [005](../adrs/animal_monitoring/005-adr-animal-feeding-and-health-alerting.md) give us real judgement to score against. Disagreement rate is tracked, because a persona that agrees with everything is worthless, and a collapsing objection rate freezes it.

### The gate

Classification is **deterministic code, not an agent's judgement**, for the same reason [ADR-AI-002](../adrs/platform/ADR-AI-002-validation-and-verification.md) keeps guardrails in code.

**Auto-approve requires all of:** advisory plane, tests or docs only · no change to gate policy, fitness functions, cryptography, key handling, pricing bounds, consent, anonymisation or retention · no new dependency, MCP tool or schema change · evals, fitness functions and tests green · no unresolved persona objection · no ADR status change.

**Always human, however small the diff:**

| Area | Reviewer |
|---|---|
| Transactional plane: money, access, entitlement, audit | Platform engineer |
| The Decision Gate or a fitness function guarding it | Platform engineer, second reviewer |
| Species rhythm, alert thresholds, alert levels | **The real vet or keeper** |
| Prompts, models, routes, indexes, calibration tables | The capability's named owner |
| EU AI Act surface: pricing, consent, anonymisation, retention | Compliance owner |

Every agent-authored change carries its analysis, the persona objections and their resolution, the model and prompt version, and the ADR it serves. An agent that cannot name an ADR has found either a gap or a change nobody asked for, and both stop for a human.

## How we know it's working

| Signal | Target |
|---|---|
| Escaped defects in auto-approved changes | Zero in the transactional plane, by construction |
| Persona objections later confirmed by the real expert | The measure of whether personas save expert time or waste it |
| Persona disagreement rate against held-out keeper verdicts | Inside the calibration band; a collapse freezes the persona |
| Changes merged with no traceable ADR | Zero, enforced at the gate |
| Share of merges that were auto-approved | Tracked, never targeted |

## What we deliberately don't do

| We don't | Why |
|---|---|
| Let a Vet persona sign off a species rhythm | [004](../adrs/animal_monitoring/004-adr-animal-feeding-and-health-learning-normal.md) requires a real vet, and every individual baseline is built on that rhythm. A simulated one would sit underneath every welfare alert on the estate |
| Let a Keeper persona supply a verdict | Synthetic verdicts would poison the eval sets, thresholds and baselines at once, and corrupt the ground truth the persona is calibrated against |
| Let an agent auto-approve a change to the gate that judges agents | Self-modifying approval logic |
| Accept an agent's confidence as the approval signal | A model judging its own output is not a control |
| Report ADLC progress as automation percentage | The number that matters is escaped defects and expert time saved |

<p align="right"><a href="../../README.md">↑ Back to README</a></p>
