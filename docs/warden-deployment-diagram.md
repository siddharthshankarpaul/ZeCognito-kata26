# The Warden Platform: Deployment Diagram

**Client:** Von Digitalis Estates · 
**Notation:** C4 Deployment diagram (Level 3: deployment view) · Companion to the System Container diagram.

This view maps each container onto infrastructure and shows the single governed estate → cloud crossing. Rule of thumb it encodes: **the edge does perception, the cloud does cognition**; only Lookout (CV inference) runs on-estate, every other AI service runs in the cloud.

---

## Deployment diagram: C4 Level 3

![The Warden Platform: Deployment Diagram — Deployment diagram: C4 Level 3](diagrams/warden-deployment.svg)

---

## Key

| Notation | Meaning |
|---|---|
| **Green outer zone "On-Estate"** | Physical estate infrastructure, runs offline through WiFi outages. |
| **Blue outer zone "Cloud Region"** | Managed cloud, single region, multi-AZ. |
| **Inner subgraph** | A deployment node (edge cluster, estate gateway, k8s namespace, managed service). |
| **`[device]` dashed node** | Physical hardware (camera, sensor, reader). |
| **`[container]` / plane-coloured box** | A deployable container, coloured by plane (blue = transactional, amber = advisory, green = edge). |
| **Thick arrow ⇒** | The estate↔cloud crossing: MQTT/TLS, QoS 1, store-and-forward. The only path over the wire. |
| **Thin arrow →** | Local wiring or in-cloud infra link; protocol on the label. |
| **Red node** | KMS / Decision Gate, trust-critical. |

---

## Hardware & sizing notes (indicative, exact counts belong in the hardware-budget ADR)

- **Edge Cluster node:** small-form-factor GPU/NPU (Jetson-class) per area cluster; runs Lookout CV, the Anonymiser, the local MQTT broker, and validator cache. ~10–12 clusters cover the 55 enclosures + 40 rides, grouped by proximity, not one node per enclosure.
- **Sensors:** MQTT-capable microcontrollers (feed-weight, water quality, environment), the "MQTT hardware budget" the brief provides for.
- **Estate Gateway:** one ruggedised on-site server aggregating all zone brokers into a single backhaul uplink.
- **The crossing:** MQTT over TLS, QoS 1, store-and-forward is the **only** estate→cloud path. During a WiFi outage the edge keeps admitting visitors and buffering telemetry; it drains on reconnect. Uplink carries telemetry/counts/admissions; downlink carries entitlement cache + revocations.
- **Key custody:** the Ed25519 **private** signing key never leaves cloud KMS; edge validators hold only the **public** key, so offline ticket verification works with zero secret exposure on-site.
- **Cloud:** managed event streaming, managed OLTP per service, object-store data lake; transactional and advisory containers run in separate namespaces so an advisory failure can't affect the money/access path.

<p align="right"><a href="../README.md">↑ Back to README</a></p>
