# Engineering Practice: AIOps

**Operationalises:** [platform/ADR-AI-003](../adrs/platform/ADR-AI-003-production-monitoring.md)

> **Our use of the term:** operating AI in production, the response layer on top of the signals defined in [ADR-AI-003](../adrs/platform/ADR-AI-003-production-monitoring.md). Not using AI to run IT operations. [MLOps](mlops.md) decides what is safe to ship; this decides what is safe to keep running.

## Why this practice exists for this estate

[ADR-AI-003](../adrs/platform/ADR-AI-003-production-monitoring.md) already defines the signals, the windows, the drills and the kill switch, but not who is actually on call when one of them fires. This practice is the operating rhythm that gives each of those a person, a calendar and a rehearsed response, and it deliberately restates none of the mechanism.

## What we do

| Practice | Rule |
|---|---|
| **One rota, platform-wide** | Not a rota per capability. The responder on duty covers every signal, and knows which ones they must not judge alone. |
| **Welfare signals escalate to a person, not a page** | A welfare responder involves the duty vet rather than forming a view about the animal. The rota's job is routing, not diagnosis. |
| **The drill calendar has an owner and a date** | The four quarterly drills are booked at the start of the quarter, with named participants, including the keeper for the fallback usability review. An unbooked drill is already a failed one. |
| **Incidents are classified on blast radius** | Who was affected, whether an AI-influenced action reached the transactional plane, and whether the audit log can replay it. Classification decides the postmortem depth, not how loud the alert was. |
| **Every incident ends in an owned change** | A changed threshold, a changed response, or a changed design, with a name against it. Failed drills get the same treatment as live incidents. |
| **Kill-switch authority is published** | Who currently holds capability-level and estate-level authority is written down and reviewed when people change roles. The mechanism is the ADR's; knowing who can use it at 3am is ours. |
| **A frozen promotion needs an owner to unfreeze it** | When a canary fingerprint stops promotion, a named person owns the rerun and the decision to resume. Freezes do not expire quietly. |

## How we know it's working

| Signal | Target |
|---|---|
| Signals with a named responder currently on rota | 100% |
| Quarterly drills booked at quarter start | 4 of 4, before the quarter begins |
| Incidents closed without an owned change | Zero |
| Frozen promotions with no named owner | Zero |
| Median time from signal to a human acknowledging it | Inside the window the signal promises |

## What we deliberately don't do

| We don't | Why |
|---|---|
| Let a responder judge a welfare case themselves | The rota routes; the duty vet decides. Speed is not a reason to skip that |
| Close an incident because the metric recovered | Recovery is not a cause. Without the postmortem the same signal fires again next month |
| Carry a drill over to next quarter | A deferred drill reads as scheduled and behaves as skipped |
| Leave kill-switch authority attached to someone who changed roles | The one control where a stale name is the same as no control |
| Silence a noisy signal instead of fixing its threshold | Silencing removes the evidence that the threshold was wrong |

<p align="right"><a href="../../README.md">↑ Back to README</a></p>
