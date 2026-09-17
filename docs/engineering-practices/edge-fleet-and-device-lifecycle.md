# Engineering Practice: Edge Fleet and Device Lifecycle

**Operationalises:** [platform/ADR-001](../adrs/platform/ADR-001-store-and-forward-mqtt%20.md), [platform/ADR-005](../adrs/platform/ADR-005-edge-vs-cloud-cv-placement.md), [platform/ADR-006](../adrs/platform/ADR-006-edge-anonymiser.md), [animal_monitoring/002](../adrs/animal_monitoring/002-adr-sensor-connectivity-lorawan-not-wifi.md), [animal_monitoring/003](../adrs/animal_monitoring/003-adr-animal-feeding-and-health-measuring.md), [animal_monitoring/006](../adrs/animal_monitoring/006-adr-edge-vision-for-piranha-counting.md)

## Why this practice exists for this estate

Most of this platform is not in a data centre. It is roughly 300 devices across 55 enclosures and 40 rides, on two transports that do not share a firmware path, some of it behind doors that take two people and a safety routine to open. You cannot SSH into a tortoise house, and a device that needs a site visit costs a keeper's morning, not a deploy slot. Device lifecycle therefore gets the same standing as the software pipeline.

## What we do

| Practice | Rule |
|---|---|
| **Enrol before publish** | A device is registered with its zone, enclosure, transport, owner and calibration due date before it is allowed on the bus. An unregistered device that appears is quarantined and counted. |
| **The registry is the inventory** | Firmware and model version, install date, battery fit date, calibration owner and due date live in one place. If it is not in the registry it does not exist for planning. |
| **Camera placement is a logged change** | [006](../adrs/animal_monitoring/006-adr-edge-vision-for-piranha-counting.md) rules that re-aiming a camera invalidates the model, so mount and lighting are recorded with the device and a physical adjustment forces a model re-release. |
| **Calibration debt is a funded number** | Tracked on the operator surface next to device health and gateway coverage, with a ceiling. Breaching the ceiling schedules a maintenance round; it is a planning failure, not a keeper's backlog. Degradation is automatic from the due date, never a judgement call. |
| **Work is batched into rounds** | A round is planned per zone from the registry: batteries, calibration, pending firmware, replacements. One visit clears the enclosure. Dangerous enclosures are only ever visited on a planned cycle. |
| **Swap, don't repair in place** | A failed device is replaced with a pre-enrolled spare and diagnosed off-site. Spares are held per class and sized quarterly against observed failure rates. |
| **Retire explicitly** | A removed device is closed off with an end date rather than deleted, and any baseline that depended on it is flagged. An enclosure dropping from individual to enclosure-level attribution must show that change in confidence, not hide it. |

### Firmware and model rollout

The [CI/CD pipeline](../warden-cicd-pipeline.md) calls edge deployment "staged OTA to MQTT devices." That holds for one half of the fleet only, and the split is the whole practice:

| Class | Path |
|---|---|
| **Hubs, GPU boxes, PoE cameras, mains feeders** | Genuine staged OTA over Ethernet: one zone, then a cohort, then the estate, with automatic rollback on a failed health check. Edge CV models follow this path, shipped with their per-zone calibration table under [MLOps](mlops.md). |
| **LoRaWAN battery devices** | Not OTA in any useful sense. [002](../adrs/animal_monitoring/002-adr-sensor-connectivity-lorawan-not-wifi.md) accepts that duty-cycled radio makes firmware a gateway-mediated or physical job, so updates are batched into planned rounds against the battery calendar. |

The consequence, stated rather than buried: **battery sensor firmware is close to immutable in practice.** We keep sensor firmware boring, put changeable logic in the hub where we can reach it, and treat any sensor-firmware change as a funded field campaign.

## How we know it's working

| Signal | Target |
|---|---|
| Calibration debt against ceiling | Below, reviewed monthly |
| Unplanned enclosure entries caused by device failure | Trending to zero; each one reviewed |
| Enclosures meeting the two-gateway rule | 100%, continuously |
| Readings lost to a full ring buffer | Zero; a non-zero month is a coverage defect |
| Devices on the bus but not in the registry | Zero, enforced at the bus |

## What we deliberately don't do

| We don't | Why |
|---|---|
| Schedule a sensor firmware change without funding the round that delivers it | The radio cannot carry it, so an unfunded update leaves the fleet in an unknown state |
| Approve a camera move as maintenance | It is a model change, and treating it as a ladder job silently invalidates the model |
| Dispatch a keeper per failed device | Rounds exist so enclosure time is spent once, not three times |
| Put device health in front of keepers | It competes with welfare alerts for the attention the alerting design is protecting |
| Close a calibration item without re-fitting | Marking it done resets the clock while the drift stays |

<p align="right"><a href="../../README.md">↑ Back to README</a></p>
