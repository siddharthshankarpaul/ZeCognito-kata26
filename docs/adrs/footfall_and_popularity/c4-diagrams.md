# C4 Diagrams: Crowd & Popularity Analytics

*Supports [001](001-adr-crowd-and-popularity-analytics.md). Diagram detail, not itself an architectural decision.*

## Level 1: System Context

```mermaid
flowchart TB
    Guest(["Guest<br/>wants live wait times"])
    Ops(["Ops & staffing<br/>deploys staff by zone"])
    Finance(["The Countess / Finance<br/>reviews trend reports"])

    CPA["Crowd & Popularity Analytics<br/>counts, forecasts, advises"]

    Ticketing["Ticketing & Pricing"]
    Concierge["Guest Concierge Agent"]
    Gateway["Common LLM Gateway<br/>shared platform"]
    CI["Common CI Pipeline<br/>shared platform"]

    Guest -->|"reads live wait time (via API)"| CPA
    Ops -->|"reads ranked zone report (via API)"| CPA
    Finance -->|"reads monthly trend report (via API)"| CPA
    CPA -->|"shares occupancy with (via API)"| Ticketing
    Concierge -->|"reads popularity + forecast (via MCP)"| CPA
    CPA -.->|"Step 3 · simulate_scenario (via MCP)"| Ticketing
    CPA -->|"routes model calls through"| Gateway
    CPA -->|"runs eval regression in"| CI

    classDef person fill:#EEEDFE,stroke:#534AB7,color:#26215C;
    classDef focus fill:#E1F5EE,stroke:#0F6E56,color:#04342C;
    classDef ext fill:#F1EFE8,stroke:#5F5E5A,color:#2C2C2A;
    class Guest,Ops,Finance person;
    class CPA focus;
    class Ticketing,Concierge,Gateway,CI ext;
```

| Symbol | Meaning |
|---|---|
| Stadium shapes | People (actors). |
| Rectangles | Software systems. |
| Purple | Actors. |
| Teal | The system this document describes. |
| Gray | Other systems it depends on or serves, owned elsewhere. |
| Dashed | Step 3 reuse, not yet built. |
| "(via API)" / "(via MCP)" labels | Which facade each relationship uses. |

---

## Level 2: Container

```mermaid
flowchart TB
    subgraph CPA["Crowd & Popularity Analytics"]
        direction TB
        Edge["Edge counters<br/>IR + CV, emits count/density"]
        Ingest["Ingestion & validator<br/>schema check, dedupe, buffer"]
        Recon["Reconciliation + calculators<br/>occupancy, dwell, queue-wait"]
        RM[("Read model store<br/>occupancy · dwell · forecast views")]
        CalibData[("Calibration dataset<br/>CV estimate vs. actual, per zone")]
        Calib["Confidence calibration service<br/>per-zone isotonic fit"]
        Forecast["Forecast service<br/>lightweight time-series"]
        API["REST / GraphQL API<br/>for software clients"]
        MCP["Crowd analytics MCP server<br/>read-only tools"]
        Agent["Queue management agent<br/>advisory, calls MCP tools"]
        Twin["Twin simulation service, Step 2<br/>transition graph + queueing calc"]
    end

    MQTT["Edge MQTT broker, external"]
    LLM["Common LLM Gateway, external"]
    CIPipe["Common CI Pipeline, external"]
    SoftwareClients["Guests · Ops · Finance, external"]
    OtherAgents["Guest Concierge · future Pricing agent, external"]

    MQTT --> Edge --> Ingest --> Recon --> RM
    Recon --> CalibData
    CalibData --> Calib
    Calib -.->|"calibrated score, periodic"| Edge
    Calib --> CIPipe
    Forecast --> RM
    RM --> Forecast
    RM --> API
    RM --> MCP
    RM --> Twin
    MCP --> Agent
    MCP -.-> OtherAgents
    Twin --> MCP
    Agent --> LLM
    Twin -.-> LLM
    Agent --> CIPipe
    Twin -.-> CIPipe
    API --> SoftwareClients

    classDef det fill:#F1EFE8,stroke:#5F5E5A,color:#2C2C2A;
    classDef store fill:#E6F1FB,stroke:#185FA5,color:#042C53;
    classDef ai fill:#FBEAF0,stroke:#993556,color:#4B1528;
    classDef step2 fill:#EEEDFE,stroke:#534AB7,color:#26215C;
    classDef ext fill:#FAEEDA,stroke:#854F0B,color:#412402;
    class Edge,Ingest,Recon,API,MCP,Calib det;
    class RM,CalibData store;
    class Agent,Forecast ai;
    class Twin step2;
    class MQTT,LLM,CIPipe,SoftwareClients,OtherAgents ext;
```

| Symbol | Meaning |
|---|---|
| Gray | Deterministic container, including the REST API, the MCP tool surface, and the confidence calibration service. |
| Blue cylinder | Data stores: the CQRS read models and the calibration dataset. |
| Pink | AI/learned container (calls an LLM or produces a forecast). |
| Purple | Step 2 container, required next but not yet built. |
| Amber | External system or consumer outside this bounded context. |
| Dashed arrow | A relationship that starts at Step 2/3, or a periodic/batch update rather than a live call. |

The Twin container is drawn solid, matching every other Step 1 container, because it's a required next step, not an optional maybe. Only its outward arrows to the LLM Gateway and CI Pipeline stay dashed, since those calls don't happen until Step 2 ships. The same dashing marks the Step 3 relationship where other agents could call the twin's `simulate_scenario` tool directly through MCP, once it's proven. The confidence calibration service is new this round; its full mechanism is in the [Confidence Calibration Design Note](confidence-calibration-design-note.md).

---

## Level 3a: Component, Queue Management Agent and MCP Server

The targeted AI-subsystem deep-dive: where exactly the LLM is called, and what it is, and isn't, allowed to do on its own.

```mermaid
flowchart TD
    subgraph AgentBox["Queue management agent"]
        direction TB
        Loop["Orchestration loop<br/>decides what data it needs"]
        Compose["Compose recommendation<br/>+ stated reason"]
    end

    subgraph McpBox["Crowd analytics MCP server"]
        direction TB
        Access["Access control<br/>per calling agent"]
        Registry["Tool registry<br/>get_zone_occupancy · get_zone_forecast · get_popularity_ranking"]
    end

    Loop -->|"tool call"| Access
    Access -->|"allowed"| Registry
    Registry -->|"reads"| RM[("Read model store")]
    Registry -->|"structured data"| Loop
    Loop --> LLMCall["LLM Gateway call<br/>reasoning only, no tool access"]
    LLMCall --> Compose
    Compose --> Guard{"Schema-valid<br/>output?"}
    Guard -- "yes" --> OPS["Ops review"]
    Guard -- "no" --> FALLBACK["Fallback: plain sorted table<br/>no LLM output shown"]

    classDef ai fill:#FBEAF0,stroke:#993556,color:#4B1528;
    classDef det fill:#F1EFE8,stroke:#5F5E5A,color:#2C2C2A;
    classDef store fill:#E6F1FB,stroke:#185FA5,color:#042C53;
    classDef gate fill:#FAEEDA,stroke:#854F0B,color:#412402;
    class Loop,Compose,LLMCall ai;
    class Access,Registry,OPS,FALLBACK det;
    class RM store;
    class Guard gate;
```

| Symbol | Meaning |
|---|---|
| Pink | Components that reason or call an LLM. |
| Gray | Deterministic components, including the entire MCP server. |
| Blue cylinder | The CQRS read-model store. |
| Amber diamond | The guardrail deciding whether the LLM's output is trusted or discarded. |

**Why this diagram matters:** the LLM itself never calls a tool with open-ended autonomy. The orchestration loop, plain deterministic code, decides what data is needed and fetches it through MCP's access-controlled tool registry before the LLM is invoked. The LLM's only job is composing a recommendation and a stated reason from data it's handed, and that output still has to clear a schema guardrail before a human ever sees it. This is the concrete answer to "where does non-determinism enter the pipeline": it enters in exactly one place, and it's fenced on both sides.

---

## Level 3b: Component, Confidence Calibration Mechanism

The second targeted AI-subsystem deep-dive: how a raw CV model score becomes a trustworthy, per-zone confidence number, and what breaks the circularity of validating CV against the exact sensor (beam counters) it's meant to outperform.

```mermaid
flowchart TD
    Z["Zone-empty reconciliation<br/>occupancy hits zero overnight"] --> D[("Calibration dataset<br/>CV estimate vs. actual")]
    MA["Periodic manual audit<br/>spot-checks dense-period footage"] --> D
    D --> F["Per-zone calibration fit<br/>isotonic regression"]
    F --> C["Calibrated confidence score<br/>feeds the live CV node"]
    F --> T["Threshold tuning<br/>from a precision-recall curve"]
    C --> G["CI eval gate<br/>gates any recalibration change"]
    T --> G
    G -.->|"approved change ships"| F

    classDef det fill:#F1EFE8,stroke:#5F5E5A,color:#2C2C2A;
    classDef store fill:#E6F1FB,stroke:#185FA5,color:#042C53;
    classDef learn fill:#EEEDFE,stroke:#534AB7,color:#26215C;
    classDef ship fill:#FBEAF0,stroke:#993556,color:#4B1528;
    classDef ev fill:#FAECE7,stroke:#993C1D,color:#4A1B0C;
    class Z,MA det;
    class D store;
    class F,T learn;
    class C ship;
    class G ev;
```

| Symbol | Meaning |
|---|---|
| Gray | Ground-truth sourcing, deterministic/procedural. |
| Blue cylinder | The labelled calibration dataset. |
| Purple | The statistical fitting/tuning process. |
| Pink | The calibrated artefact that actually ships to production. |
| Coral | The governance gate any change must clear. |
| Dashed | An approved change flowing back into the live calibration. |

**Why this diagram matters:** the beam counter can't be the calibration reference, because it's specifically unreliable in the exact dense conditions CV's confidence score most needs validating against. Calibrating against a broken reference would just teach CV to agree with a wrong answer. Two independent sources break that circularity: zone-empty reconciliation (free, continuous, but blind mid-day) and periodic manual audits (sparse, but the only real ground truth in dense conditions). The full mechanism, including cold-start behaviour for a brand-new CV zone with no calibration history yet, is in the [Confidence Calibration Design Note](confidence-calibration-design-note.md).
