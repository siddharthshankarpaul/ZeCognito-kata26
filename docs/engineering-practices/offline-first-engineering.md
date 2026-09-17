# Engineering Practice: Offline-First Engineering

**Operationalises:** [platform/ADR-001](../adrs/platform/ADR-001-store-and-forward-mqtt%20.md), [platform/ADR-002](../adrs/platform/ADR-002-offline-ticket-signing.md), [ticketing/003](../adrs/ticketing_and_access_control/003-adr-reentry-and-gate-connectivity.md), [animal_monitoring/001](../adrs/animal_monitoring/001-adr-edge-first-event-driven-architecture.md), [animal_monitoring/004](../adrs/animal_monitoring/004-adr-animal-feeding-and-health-learning-normal.md), [animal_monitoring/005](../adrs/animal_monitoring/005-adr-animal-feeding-and-health-alerting.md)

## Why this practice exists for this estate

The brief names patchy WiFi twice, and nearly every platform decision bends around it. The ADRs above decide the mechanisms. This practice is the discipline that stops "offline-first" from quietly decaying into "works offline if nothing unusual happens," which is what normally occurs when every developer has good connectivity and the test suite runs on a laptop with a working network.

The rule is simple: **the network being down is the normal case, not the error case.**

## What we do

| Practice | Rule |
|---|---|
| **The edge holds the system of record** | The zone hub is authoritative, the cloud is a downstream consumer ([animal_monitoring/001](../adrs/animal_monitoring/001-adr-edge-first-event-driven-architecture.md)). Nothing on the estate waits for the cloud to agree. |
| **Every consumer is idempotent** | QoS 1 guarantees at-least-once, so duplicates are expected, not exceptional ([ADR-001](../adrs/platform/ADR-001-store-and-forward-mqtt%20.md)). Every event carries a device timestamp and sequence number, and ordering is handled at the application layer. |
| **Buffer at every hop** | Device ring buffers (24 hours on feeder controllers), gateway buffering with 4G fallback, hub store-and-forward. A radio gap becomes a delay, not a loss. |
| **Degrade visibly, never silently** | A system running on stale or local-only data says so. If the cloud is unreachable for more than a week the hub computes baselines locally **and declares it** ([animal_monitoring/004](../adrs/animal_monitoring/004-adr-animal-feeding-and-health-learning-normal.md)). |
| **Every user surface that matters offline is offline-first** | The turnstile verifies signatures with no round trip ([ADR-002](../adrs/platform/ADR-002-offline-ticket-signing.md)); the keeper app works offline because the detection behind it did ([animal_monitoring/005](../adrs/animal_monitoring/005-adr-animal-feeding-and-health-alerting.md)). |
| **Test with the network cut, not mocked** | Every offline path has a test that physically severs connectivity, including reconnect. A mocked outage never reproduces a reconnect storm. |
| **Reconnect is a designed path** | Backfill is rate-limited and ordered so a zone coming back after a long outage cannot swamp the hub or reorder itself into a wrong baseline. |
| **Quarterly "WiFi off" drill** | One zone runs a full shift disconnected, with keepers working normally. A drill that does not run counts as a failed drill, matching the [ADR-AI-003](../adrs/platform/ADR-AI-003-production-monitoring.md) rule. |

## How we know it's working

| Signal | Target |
|---|---|
| Readings lost to a full buffer | Zero |
| Duplicate events causing a double effect | Zero; duplicates are expected and absorbed |
| Turnstile admissions completed with no network | 100% of attempts, within a second |
| Quarterly disconnected-shift drill | Runs, with findings logged |
| Surfaces showing stale data without saying so | Zero |

## What we deliberately don't do

| We don't | Why |
|---|---|
| Sign off an offline path that was reasoned about but never demonstrated with the network cut | A mocked outage never reproduces a reconnect storm, and reconnect is where offline designs actually fail |
| Test the offline path more lightly because it is "the fallback" | At the gates it is the path that runs when everything else has already gone wrong |
| Backfill an estimate so a chart looks continuous | It converts a known gap into an unknown wrong number, and something downstream will train on it |
| Debug an ordering problem by adjusting device clocks | Battery devices drift by design; sequence numbers and application-level ordering are the fix |
| Add a feature to the keeper app that needs connectivity to complete | It puts the keeper back online at exactly the moment the estate is not |

<p align="right"><a href="../../README.md">↑ Back to README</a></p>
