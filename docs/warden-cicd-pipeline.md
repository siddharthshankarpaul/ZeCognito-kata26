# The Warden Platform: CI/CD Pipeline

One pipeline, two lanes: deterministic services get a classic test pyramid; advisory/AI changes must clear an eval gate before they can influence the Decision Gate.

---

## Pipeline

![The Warden Platform: CI/CD Pipeline](diagrams/warden-cicd-pipeline.svg)

---

## Two lanes, one pipeline

- **Deterministic services** follow a classic test pyramid, plus **architecture fitness functions** that *fail the build* if the two-plane separation is violated (e.g. an AI service reaching the transactional plane without going through the gate).
- **Advisory / AI changes** must pass **golden-dataset eval gates** and **drift checks**, then go through **shadow/canary with human sign-off** before they can influence the decision gate.
- **Model/provider swaps** are Augur **configuration changes** validated by evals, not application redeploys.
- **Edge deployment** is staged **OTA to MQTT devices**; production is continuously watched by the same eval harness that gated it.

<p align="right"><a href="../README.md">↑ Back to README</a></p>
