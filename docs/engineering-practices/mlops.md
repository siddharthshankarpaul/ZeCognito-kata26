# Engineering Practice: MLOps

**Operationalises:** [platform/ADR-AI-002](../adrs/platform/ADR-AI-002-validation-and-verification.md), [animal_monitoring/004](../adrs/animal_monitoring/004-adr-animal-feeding-and-health-learning-normal.md), [animal_monitoring/006](../adrs/animal_monitoring/006-adr-edge-vision-for-piranha-counting.md), [footfall/001](../adrs/footfall_and_popularity/001-adr-crowd-and-popularity-analytics.md)

> [ADR-AI-002](../adrs/platform/ADR-AI-002-validation-and-verification.md) is the **decision** about what is safe to ship. This practice is the **lifecycle** around it: where artefacts live, who owns them, what triggers a retrain, and how a model reaches a GPU box in an animal house.

## Why this practice exists for this estate

Models here are small, numerous and unglamorous: per-species rhythms, per-individual baselines, two piranha counting models, per-zone calibration tables, a footfall forecast, plus prompts and indexes. None is a large training job. All of them can rot quietly, and several of them ship to hardware nobody can reach without a site visit.

## What we do

| Practice | Rule |
|---|---|
| **Everything that changes behaviour is a versioned artefact** | Prompts, models, routes, indexes, thresholds, calibration tables, schemas, tool tiers, and the [ADLC](agentic-development-lifecycle.md) personas. All registered, all pinned, none edited in place. |
| **Every artefact has a named owner** | Not a team. A person who is called when its signal moves. |
| **Golden sets are curated, not accumulated** | Each set mixes a random slice of ordinary traffic with known corrections, sized by the power needed to detect a difference worth acting on ([ADR-AI-002](../adrs/platform/ADR-AI-002-validation-and-verification.md)). Corrections alone would skew toward whatever defect someone happened to notice. |
| **Verdicts are the labelling pipeline** | Mandatory keeper verdicts ([animal_monitoring/005](../adrs/animal_monitoring/005-adr-animal-feeding-and-health-alerting.md)) are the estate's main source of labelled data. Routine keeper work is the training set, and it is budgeted as labour, not assumed free. |
| **Retraining is triggered, not scheduled** | A trigger is a signal breach, a corpus change, a new species, or a changed capture setup. Baselines are the exception: they train nightly in the cloud and ship to the hub as versioned configuration ([animal_monitoring/004](../adrs/animal_monitoring/004-adr-animal-feeding-and-health-learning-normal.md)). |
| **Abnormal periods never enter training** | A confirmed illness or treatment course is excluded, which is what stops a slow decline from becoming the new definition of normal. |
| **Re-embedding is a release, not a refresh** | Changing the embedding model silently invalidates the index. The index is rebuilt, re-evaluated and shipped as one versioned change ([ADR-AI-001](../adrs/platform/ADR-AI-001-provider-and-model-portability.md), [animal_monitoring/009](../adrs/animal_monitoring/009-adr-retrieval-index-design.md)). |
| **Edge rollout is part of the release** | A CV model ships with its per-zone calibration table as one versioned artefact, staged a zone at a time over the cabled OTA path that [Edge Fleet](edge-fleet-and-device-lifecycle.md) owns. A re-aimed camera invalidates the model and forces a re-release. |
| **Low-volume capabilities qualify by replay** | At 55 calls a day a percentage rollout is a handful of samples. Those qualify against recorded inputs instead; only the concierge has the traffic for a live percentage rollout. |

## How we know it's working

| Signal | Target |
|---|---|
| Artefacts in production with no registry entry or owner | Zero |
| Releases where the eval suite was skipped | Zero |
| Index rebuilt without re-evaluation after an embedding change | Zero |
| Golden-set size against the computed minimum per capability | At or above, per capability |
| Annotation hours consumed against budget | Tracked monthly; it is a real line, not a favour |

## What we deliberately don't do

| We don't | Why |
|---|---|
| Accept an eval score without reading it against that capability's published annotator ceiling | A score above the ceiling is a measurement error, not a win |
| Promote a high-risk capability on a judge-model verdict alone | The human-labelled calibration slice is what makes the judge's number mean anything |
| Queue a retrain without naming the trigger that fired it | An untriggered retrain is usually someone reacting to a dashboard, and it consumes annotation budget that has an owner |
| Promote a CV model before its per-zone calibration table has been fitted and evaluated alongside it | Another zone's table imports another zone's lighting bias |
| Edit a prompt, threshold or index in place to fix something quickly | It leaves production behaviour with no version anyone can roll back to |

<p align="right"><a href="../../README.md">↑ Back to README</a></p>
