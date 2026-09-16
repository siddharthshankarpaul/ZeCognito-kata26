# C4 Diagrams: Animal Care Monitoring

*Supports [001](001-adr-edge-first-event-driven-architecture.md), [002](002-adr-sensor-connectivity-lorawan-not-wifi.md), [003](003-adr-animal-feeding-and-health-measuring.md), [004](004-adr-animal-feeding-and-health-learning-normal.md), [005](005-adr-animal-feeding-and-health-alerting.md), [008](008-adr-rag-over-a-curated-corpus.md), and [009](009-adr-retrieval-index-design.md). Diagram detail, not itself an architectural decision.*

How much and how well the animals eat, and how we track their health, are one pipeline running on one set of containers, so they share these diagrams.

---

## Level 1. System Context

```mermaid
flowchart TB
    Keeper(["Keeper<br/>runs the round, answers alert cards"])
    Vet(["Veterinarian<br/>signs rhythms and case drafts"])
    Curator(["Curator and Finance<br/>read lead time and care cost"])

    ACM["Animal Care Monitoring<br/>measures intake and condition,<br/>learns each animal's normal,<br/>tells a keeper when it drifts"]

    Piranha["Piranha Population Count"]
    Augur["Augur LLM Gateway<br/>shared platform"]
    CI["Shared CI and eval pipeline<br/>shared platform"]
    Mqtt["Store and forward contract<br/>estate wide"]
    Sms["Failover router and SMS"]

    Keeper -->|"hand fed meals, verdicts"| ACM
    Vet -->|"signs rhythms and drafts"| ACM
    Curator -->|"reads reporting"| ACM
    ACM -->|"alerts with no data connection"| Sms
    ACM -->|"food per fish"| Piranha
    ACM -->|"routes model calls through"| Augur
    ACM -->|"gates every change in"| CI
    ACM -->|"forwards to the cloud using"| Mqtt

    classDef person fill:#EEEDFE,stroke:#534AB7,color:#26215C;
    classDef focus fill:#E1F5EE,stroke:#0F6E56,color:#04342C;
    classDef ext fill:#F1EFE8,stroke:#5F5E5A,color:#2C2C2A;
    class Keeper,Vet,Curator person;
    class ACM focus;
    class Piranha,Augur,CI,Mqtt,Sms ext;
```

**Key.** Stadium shapes are people. Teal is the system these diagrams describe. Grey is a system it depends on or serves, owned elsewhere.

**Why this diagram matters.** Three people need three different things from this system: the keeper wants the next action, the vet wants a signed record, and the curator wants a number. The only thing flowing to the piranha count is food eaten per fish, which is the cheapest population signal on the estate and costs nothing extra to produce.

---

## Level 2. Container

```mermaid
flowchart TB
    Keeper(["Keeper"])
    Vet(["Veterinarian"])

    subgraph SYS["Animal Care Monitoring"]
        direction TB

        subgraph DEV["Device layer, chosen per species"]
            direction LR
            Feeder["Feeder controller<br/>[MCU, load cell, LoRaWAN or Ethernet]<br/>dispenses, weighs the bowl every 15s"]
            Rfid["RFID reader with weigh pad<br/>[LoRaWAN or Ethernet]<br/>which tag is at the bowl, and weight"]
            Sense["Other sensing layers<br/>[LoRaWAN, PoE, short range radio]<br/>environment everywhere, wearables,<br/>aimed cameras for the untaggable"]
        end

        subgraph S1["Edge hub, stage 1. Measure"]
            direction TB
            Broker["MQTT broker<br/>[Mosquitto, QoS 1]"]
            Gate["Ingest and quality gate<br/>[service]<br/>range, rate of change, timestamp"]
            Cal[("Calibration register<br/>[store]<br/>owner and due date per instrument")]
            Derive["Intake derivation and attribution<br/>[service]<br/>dispensed minus leftover, mass curve,<br/>tag gives the animal"]
            TS[("Time series, 90 days<br/>[store]<br/>each intake figure carries<br/>its attribution level")]
        end

        subgraph S2["Edge hub, stage 2. Learn normal and spot drift"]
            direction TB
            Spec[("Species rhythm<br/>[versioned config, vet signed off]")]
            Base[("Baselines per individual<br/>[versioned config]<br/>seasonal median and spread,<br/>illness periods excluded")]
            Events[("Known events<br/>[store]")]
            Drift["Rules and drift detector<br/>[service]<br/>six plain rules, robust z score,<br/>persistence, corroboration"]
            Phys["Physical danger path<br/>[service]<br/>no baseline, no waiting"]
        end

        subgraph S3["Edge hub, stage 3. Tell a person"]
            direction TB
            Board[("Feed board read model<br/>[store]<br/>three words per animal")]
            Router["Alert router<br/>[service]<br/>review, act, urgent"]
            Verdict["Verdict service<br/>[service]<br/>required to close an alert"]
            Sync["Sync agent<br/>[store and forward]"]
        end

        subgraph RAG["Retrieval layer, husbandry knowledge"]
            direction TB
            Corpus[("Curated corpus<br/>[cloud]<br/>provenance and review date per document")]
            Ingest["Chunk, enrich, embed<br/>[pipeline]<br/>structure aware, pinned model"]
            Index[("Hybrid index<br/>[Postgres and pgvector]<br/>keyword plus dense, fused and reranked")]
            Replica[("Index replica on the hub<br/>serves species context beside a card")]
            Propose["Rhythm proposal<br/>[retrieval, every value cited]"]
            Copilot["Husbandry copilot<br/>[retrieval, mandatory citations]"]
            Corpus --> Ingest --> Index
            Index --> Propose
            Index --> Copilot
            Index -.->|"replicated"| Replica
        end

        App["Keeper app<br/>[PWA, local store]<br/>feed board, hand fed meals, alert card<br/>source of truth for capture"]
    end

    LoRa["LoRaWAN gateways<br/>[external]"]
    Sms["Failover router, SMS<br/>[external]"]
    Lake[("Lakehouse and nightly training<br/>[external, cloud]<br/>backtested against real vet visits")]
    Write["Writing workflows<br/>[external, cloud]<br/>morning briefing, vet case draft"]
    Augur["Augur LLM gateway<br/>[external]"]

    Keeper --> App
    Feeder --> LoRa
    Rfid --> LoRa
    Sense --> LoRa
    LoRa -->|"[MQTT]"| Broker
    Broker --> Gate --> Derive --> TS
    Cal -.->|"overdue means degraded"| Gate
    App -->|"hand fed meals"| TS
    TS --> Drift --> Router
    Spec --> Base --> Drift
    Events -.->|"suppresses what it explains"| Drift
    TS --> Phys ==>|"urgent"| Router
    TS --> Board --> App
    Router --> App
    Router --> Sms -.->|"no data needed"| App
    Replica -.->|"beside the card"| App
    Copilot -->|"husbandry answers"| Keeper
    App --> Verdict -.->|"tunes thresholds"| Drift
    Verdict --> TS --> Sync
    Sync ==>|"on reconnect"| Lake
    Lake -.->|"baselines back"| Base
    Lake --> Write --> Augur
    Write -.->|"optional"| App
    Copilot --> Augur
    Propose -->|"candidates, each cited"| Vet
    Vet ==>|"signs each field"| Spec
    Verdict -.->|"labels feed evals"| Augur

    classDef person fill:#EEEDFE,stroke:#534AB7,color:#26215C;
    classDef edge fill:#E1F5EE,stroke:#0F6E56,color:#04342C;
    classDef store fill:#FAEEDA,stroke:#854F0B,color:#412402;
    classDef ext fill:#F1EFE8,stroke:#5F5E5A,color:#2C2C2A;
    classDef urgent fill:#FBEAEA,stroke:#A31515,color:#4A0D0D;
    classDef rag fill:#F5EEFB,stroke:#5B3B8C,color:#2C1A4A;
    class Keeper,Vet person;
    class Broker,Gate,Derive,Drift,Router,Verdict,Sync edge;
    class TS,Cal,Spec,Base,Events,Board store;
    class LoRa,Sms,Lake,Write,Augur ext;
    class Phys urgent;
    class Ingest,Propose,Copilot,Corpus,Index,Replica rag;
    style S1 fill:#F4FBF8,stroke:#0F6E56,color:#04342C
    style S2 fill:#F4FBF8,stroke:#0F6E56,color:#04342C
    style S3 fill:#F4FBF8,stroke:#0F6E56,color:#04342C
    style RAG fill:#FAF7FC,stroke:#5B3B8C,color:#2C1A4A
    style DEV fill:#F8F7F4,stroke:#6B6255,color:#2E2A24
```

**Key.** Green runs on the estate with no internet. Amber cylinders are stores. Purple is the retrieval layer. Red bypasses the baseline. Bold arrows mark the three paths that must never be blocked: the physical danger path, the vet's sign-off, and the one crossing of the unreliable link.

**Why this diagram matters.** The three green stages (Measure, Learn Normal, Tell a Person) are the same three records as the ADR set, so the diagram and the ADRs describe the same shape. Feeding and health aren't two separate systems, because the feed bowl is the earliest health instrument on the estate and its output is the same series in the same store. Everything in the detection path runs entirely on the estate with no internet, so a whole keeper round can complete with the link down; the cloud only ever suggests a better baseline, and the hub decides whether to use it.

Two arrows are worth following closely. The calibration register feeds into the quality gate, because an instrument past its due date marks its readings degraded, and a degraded reading may still raise an alert but can never update a baseline. And the species-context arrow reaches the card from the hub's own index replica, never mixed in with the animal's own data, so a general care sheet is never mistaken for a measurement.

---

## Level 3a. Component, where the model is called

The AI subsystem deep dive. Where exactly a model runs, on what input, and what it is not allowed to do.

```mermaid
flowchart TD
    subgraph Assemble["Fact assembly, deterministic code"]
        direction TB
        Pull["Pull last 24 to 72 hours<br/>alerts, feeds, keeper notes, sensor health"]
        Number["Number every fact as f1, f2, f3<br/>strip staff names"]
        Pull --> Number
    end

    Bundle[("Facts bundle<br/>the only thing a model may see")]
    Retr["Retrieval<br/>chunks from the curated corpus,<br/>each with its provenance"]

    subgraph Gw["Augur gateway"]
        direction TB
        Pre["Pre checks<br/>facts bundle only, no images,<br/>retrieved text wrapped as data"]
        Blocked{"Blocked class?<br/>dose, treatment, first aid"}
        Model["Model<br/>writes prose, low temperature"]
        Post["Post checks<br/>schema, every citation resolves,<br/>every number matches its fact,<br/>clinical blocklist clean"]
        Pre --> Blocked
        Blocked -->|"no"| Model --> Post
    end

    HB["No generation<br/>emergency protocol and duty vet<br/>scored as a success"]
    Retry{"First failure?"}
    Out["Delivered to a person<br/>briefing, case draft, husbandry answer"]
    Digest["Facts digest<br/>information kept, polish lost"]

    Number --> Bundle --> Pre
    Retr --> Pre
    Blocked -->|"yes"| HB
    Post -->|"passes"| Out
    Post -->|"fails"| Retry
    Retry -->|"retry with the errors"| Post
    Retry -->|"no"| Digest

    classDef det fill:#F1EFE8,stroke:#5F5E5A,color:#2C2C2A;
    classDef ai fill:#FBEAF0,stroke:#993556,color:#4B1528;
    classDef store fill:#E6F1FB,stroke:#185FA5,color:#042C53;
    classDef gate fill:#FAEEDA,stroke:#854F0B,color:#412402;
    classDef block fill:#FBEAEA,stroke:#A31515,color:#4A0D0D;
    class Pull,Number,Out,Digest det;
    class Model,Retr ai;
    class Bundle store;
    class Pre,Post,Blocked,Retry gate;
    class HB block;
```

**Key.** Grey is deterministic code. Pink is the only place a model runs. Blue is the facts bundle, the only input a model ever sees. Amber are the checks on both sides of it. Red never reaches a model at all.

**Why this diagram matters.** Non-determinism enters at exactly one pink box, and it's fenced on both sides. Code assembles the facts and numbers them first, so the model can't introduce a fact of its own. Every claim coming back has to cite a fact ID that actually exists, and every number has to match the fact it cites, so faithfulness becomes something we can test, not just judge by reading. Two whole classes of question never reach a model at all, and a refusal that escalates counts as a success, not a failure. If the checks fail twice, the keeper still gets the facts digest, so the underlying content is never withheld, only the polished wording is.

---

## Level 3b. Component, how normal is learned and drift becomes an alert

The second deep dive. How a reading becomes a baseline, and what has to be true before anybody is interrupted.

```mermaid
flowchart TD
    Care["Curated corpus<br/>care sheets and references"] --> Prop["Retrieval proposes<br/>feed frequency, fasting tolerance,<br/>seasonal patterns, each cited"]
    Prop --> VetGate{"Vet accepts, edits<br/>or rejects each field"}
    VetGate -->|"rejected"| Nothing["No effect on detection"]
    VetGate -->|"signed"| Spec[("Species rhythm<br/>versioned config")]

    Hist[("Years of history<br/>lakehouse")] --> Train["Nightly training<br/>seasonal rolling median and spread<br/>known illness periods excluded"]
    Vets["Recorded vet visits"] --> Backtest{"Backtest<br/>does it predict real events?"}
    Train --> Backtest
    Backtest -->|"passes"| Base[("Baselines per individual<br/>versioned config on the hub")]
    Spec --> Base
    Local["Cloud away over a week<br/>hub computes from its own 90 days"] -.-> Base

    Read["A reading"] --> Z["Robust z score<br/>spreads from this animal's median"]
    Base --> Z
    Z --> Pers{"Persists?<br/>N of the last M"}
    KE["Known events<br/>diet change, move, treatment"] -.->|"suppresses what it explains"| Pers
    Pers -->|"no"| Nothing
    Pers -->|"yes"| Corr{"Second signal<br/>within 48 hours?"}
    Corr -->|"no"| Look["Worth a look"]
    Corr -->|"yes, escalate one level"| Act["Act"]
    Read --> Danger["Temperature out of range"] ==>|"urgent, no waiting"| Act
    Look --> Verd["Verdict from a keeper"]
    Act --> Verd
    Verd -.->|"the label that tunes it"| Base

    classDef det fill:#F1EFE8,stroke:#5F5E5A,color:#2C2C2A;
    classDef store fill:#E6F1FB,stroke:#185FA5,color:#042C53;
    classDef learn fill:#EEEDFE,stroke:#534AB7,color:#26215C;
    classDef gate fill:#FAEEDA,stroke:#854F0B,color:#412402;
    classDef ai fill:#FBEAF0,stroke:#993556,color:#4B1528;
    classDef urgent fill:#FBEAEA,stroke:#A31515,color:#4A0D0D;
    class Read,Vets,Local,Verd,Nothing,Look det;
    class Spec,Base,Hist store;
    class Train,Z learn;
    class VetGate,Backtest,Pers,Corr gate;
    class Prop,Care ai;
    class Danger,Act urgent;
```

**Key.** Pink is retrieval, which proposes and never applies. Purple is the statistical work. Amber diamonds are the four gates a signal has to pass before a person is interrupted. Blue cylinders are versioned configuration. Red is the path that skips all of it.

**Why this diagram matters.** Three separate things decide what normal means, and none of them is a model. A vet signs the species rhythm, a nightly job learns the individual baseline and has to beat a backtest against real vet visits before it ships, and known illness periods are excluded so a slow decline never becomes the new normal. A deviation then passes two more gates, persistence and corroboration, before anybody is interrupted, which is the trade of lead time for precision that keeps keepers reading the cards. The red path exists because a failed heater is a fact rather than a deviation, and waiting for three of four days would be absurd. The verdict loop at the bottom is what makes the whole thing measurable.
