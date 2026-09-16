# C4 Diagrams: Piranha Population Count

*Supports [001](001-adr-edge-first-event-driven-architecture.md), [005](005-adr-animal-feeding-and-health-alerting.md), [006](006-adr-edge-vision-for-piranha-counting.md), and [007](007-adr-piranha-population-as-a-range.md). Diagram detail, not itself an architectural decision.*

A smoothed headcount several times a day, an alert on a meaningful drop or an escape, and a verification loop against periodic manual counts. The fish can't be tagged or handled routinely.

This is a separate pipeline from animal care monitoring, with different devices, different mathematics, and a different output shape. It shares only the message bus, the hub, and the keeper app.

---

## Level 1. System Context

```mermaid
flowchart TB
    Keeper(["Keeper<br/>counts by hand weekly, labels frames"])
    Ops(["Ops and safety<br/>responds to an escape"])
    Curator(["Curator<br/>reads stocking level and trend"])

    PPC["Piranha Population Count<br/>estimates the population as a range,<br/>alerts on a real drop or an escape"]

    ACM["Animal Care Monitoring"]
    CI["Shared CI and eval pipeline<br/>shared platform"]
    Mqtt["Store and forward contract<br/>estate wide"]
    Sms["Failover router and SMS"]

    Keeper -->|"weekly count, frame labels"| PPC
    Ops -->|"receives an escape alert"| PPC
    Curator -->|"reads the range"| PPC
    ACM -->|"food per fish"| PPC
    PPC -->|"high severity alerts"| Sms
    PPC -->|"gates every change in"| CI
    PPC -->|"forwards to the cloud using"| Mqtt

    classDef person fill:#EEEDFE,stroke:#534AB7,color:#26215C;
    classDef focus fill:#E1F5EE,stroke:#0F6E56,color:#04342C;
    classDef ext fill:#F1EFE8,stroke:#5F5E5A,color:#2C2C2A;
    class Keeper,Ops,Curator person;
    class PPC focus;
    class ACM,CI,Mqtt,Sms ext;
```

**Key.** Stadium shapes are people. Teal is the system these diagrams describe. Grey is owned elsewhere.

**Why this diagram matters.** There's no Augur gateway here on purpose: nothing in this pipeline calls a language model. The only intelligence is a small vision model we tune ourselves and a state-space model, and both produce numbers, not prose. The keeper is the single most important input, because the manual count is the only ground truth on the estate.

---

## Level 2. Container

```mermaid
flowchart TB
    Keeper(["Keeper"])
    Ops(["Ops and safety"])

    subgraph SYS["Piranha Population Count"]
        direction TB

        subgraph PH["Piranha house, on the estate"]
            direction TB
            Cam["Two PoE cameras<br/>[polarising, infrared, fixed mount]<br/>one still each every 15 min"]
            Splash["Splash sensor<br/>[LoRaWAN]<br/>surface break outside feeding"]
            Pre["Preprocess<br/>[code on the GPU box]<br/>crop, deglare, discard frames<br/>with a keeper or net in view"]
            Model["Counting model<br/>[small tuned model, GPU box]<br/>detector if spread, density map if tight"]
            Smooth["Burst maximum, then smoothing<br/>[service]<br/>higher of two cameras,<br/>median of the last two hours"]
            Cam --> Pre --> Model --> Smooth
        end

        subgraph HUB["Zone edge hub"]
            direction TB
            Broker["MQTT broker<br/>[Mosquitto, QoS 1]"]
            SSM["Population estimator<br/>[Bayesian state space model]<br/>learns the camera undercount<br/>from the manual counts"]
            Post[("Posterior store<br/>[store]<br/>a range with a direction,<br/>never a bare number")]
            Events[("Known events<br/>[store]<br/>restock, found death, transfer")]
            Rules["Plain rules and divergence check<br/>[service]<br/>10 per cent drop in 24h,<br/>a week of decline, camera<br/>against food per fish"]
            Router["Alert router<br/>[service]"]
            Sync["Sync agent<br/>[store and forward]"]
        end

        App["Keeper app<br/>[PWA, local store]<br/>manual count entry,<br/>range with a direction"]
    end

    Food["Food eaten per fish<br/>[from Animal Care Monitoring]"]
    Sms["Failover router, SMS<br/>[external]"]
    Lake[("Lakehouse<br/>[external, cloud]")]
    Annot["Annotation and retraining<br/>[external, cloud]<br/>500 to 1000 labelled frames"]

    Keeper -->|"weekly count, frame labels"| App
    Ops --> Sms
    Smooth -->|"counts only"| Broker
    Splash --> Broker
    Food --> SSM
    Broker --> SSM
    App ==>|"manual count"| SSM
    Events ==>|"observed change"| SSM
    SSM --> Post --> App
    Post --> Rules --> Router
    Broker -.->|"bypasses the model"| Router
    Router --> App
    Router --> Sms
    Post --> Sync ==>|"on reconnect"| Lake
    Pre -.->|"sampled frames"| Annot
    Annot -.->|"retrained model"| Model

    classDef person fill:#EEEDFE,stroke:#534AB7,color:#26215C;
    classDef edge fill:#E1F5EE,stroke:#0F6E56,color:#04342C;
    classDef store fill:#FAEEDA,stroke:#854F0B,color:#412402;
    classDef ext fill:#F1EFE8,stroke:#5F5E5A,color:#2C2C2A;
    classDef safety fill:#FBEAEA,stroke:#A31515,color:#4A0D0D;
    class Keeper,Ops person;
    class Pre,Model,Smooth,Broker,SSM,Rules,Router,Sync edge;
    class Post,Events store;
    class Food,Sms,Lake,Annot ext;
    class Splash safety;
    style PH fill:#F4FBF8,stroke:#0F6E56,color:#04342C
    style HUB fill:#F4FBF8,stroke:#0F6E56,color:#04342C
```

**Key.** Green runs on the estate with no internet. Amber cylinders are stores. Red is the deterministic safety signal. Bold arrows are the two inputs that move the estimate most. Dashed arrows are feedback, calibration or paths that may be absent.

**Why this diagram matters.** Three signals reach the estimator, not one, and the keeper's manual count is drawn bold because it's the only ground truth on the estate. Only counts and confidence cross the boundary out of the GPU box, and frames travel to annotation on the lowest-priority lane over a wired link and nowhere else, the bandwidth argument from [006](006-adr-edge-vision-for-piranha-counting.md) made visible. The splash arrow runs straight to the router and bypasses the estimator entirely, because statistical confidence is the wrong instrument for a fish on the floor.

---

## Level 3a. Component, the counting pipeline on the GPU box

Where the vision model runs, what is done to a frame before it gets there, and why the maximum rather than the average.

```mermaid
flowchart TD
    C1["Camera 1, one still"] --> Crop
    C2["Camera 2, one still"] --> Crop
    Crop["Crop to the tank<br/>remove glare"]
    Crop --> Discard{"Keeper or net in view?"}
    Discard -->|"yes"| Drop["Frame discarded<br/>a frame with a keeper in it<br/>is worse than no frame"]
    Discard -->|"no"| Shape{"Shoal spread or tight?"}
    Shape -->|"spread"| Det["Object detector"]
    Shape -->|"tight"| Dens["Density map model"]
    Det --> Burst
    Dens --> Burst
    Burst["Burst maximum<br/>occlusion only ever hides fish,<br/>so the maximum is closest to the truth"]
    Burst --> Pair["Higher of the two cameras"]
    Pair --> Med["Median of the last two hours"]
    Med --> Out["Smoothed count and a confidence<br/>never a population figure"]
    Sample["Sampled frames"] -.->|"wired link only"| Label["Keeper annotation<br/>the real bill in this capability"]
    Label -.->|"retrained small model"| Det
    Label -.->|"retrained small model"| Dens
    Crop -.-> Sample

    classDef det fill:#F1EFE8,stroke:#5F5E5A,color:#2C2C2A;
    classDef ai fill:#FBEAF0,stroke:#993556,color:#4B1528;
    classDef gate fill:#FAEEDA,stroke:#854F0B,color:#412402;
    classDef key fill:#E1F5EE,stroke:#0F6E56,color:#04342C;
    class C1,C2,Crop,Drop,Pair,Med,Sample,Label det;
    class Det,Dens ai;
    class Discard,Shape gate;
    class Burst,Out key;
```

**Key.** Grey is deterministic code. Pink is the only place a model runs. Amber diamonds are the two routing decisions. Green marks the two choices that carry the accuracy.

**Why this diagram matters.** Most of this pipeline is not a model. Cropping, deglaring and discarding a spoiled frame are ordinary code, and capture discipline buys more accuracy than a better model would. The burst maximum is the one line worth defending, because occlusion is one sided and can only ever hide fish, so the average is biased low by an amount that changes with how tightly the shoal packs. The output is deliberately a smoothed count with a confidence rather than a population, because the count is known to be low and publishing it alone would destroy credibility the first time a keeper counted by hand. The dashed loop on the right is the real cost in this capability, which is keeper annotation time rather than compute.

---

## Level 3b. Component, fusing three signals into a range

How a biased camera, a weekly hand count and a food signal become one honest number, and when that number is allowed to raise an alert.

```mermaid
flowchart TD
    MC["Manual count at tank service<br/>infrequent, authoritative"] ==> SSM
    FF["Food eaten per fish<br/>always on, no counting,<br/>unaffected by turbidity"] --> SSM
    CAM["Camera count<br/>frequent, noisy, biased low"] --> SSM
    KE["Known events<br/>restock, found death, transfer"] ==>|"observed change"| SSM
    SSM["State space model<br/>population as a slowly changing<br/>hidden number, with birth,<br/>death and predation terms"]
    MC -.->|"calibrates the undercount"| SSM
    SSM --> Post[("Posterior<br/>a range with a direction<br/>illustrative, around 38,<br/>likely 34 to 43, trending down")]
    Post --> Conf{"Confident the level<br/>has actually moved?"}
    Conf -->|"yes"| Alert["Change alert"]
    Conf -->|"no"| Quiet["Noise, nothing fires"]
    CAM --> Div{"Camera trend against<br/>food per fish trend"}
    FF --> Div
    Div -->|"diverging beyond tolerance"| Inv["Investigation<br/>free, needs no labels,<br/>catches what the camera<br/>cannot self report"]
    Post --> Den["Stocking level against tank volume"] --> Alert
    Splash["Splash outside feeding"] ==>|"safety, does not wait"| Safe["Top priority alert"]
    Post --> UI["Every interface shows the range<br/>no bare number anywhere"]

    classDef truth fill:#F5EEFB,stroke:#5B3B8C,color:#2C1A4A;
    classDef free fill:#E1F5EE,stroke:#0F6E56,color:#04342C;
    classDef learn fill:#EEEDFE,stroke:#534AB7,color:#26215C;
    classDef store fill:#FAEEDA,stroke:#854F0B,color:#412402;
    classDef gate fill:#FAEEDA,stroke:#854F0B,color:#412402;
    classDef safety fill:#FBEAEA,stroke:#A31515,color:#4A0D0D;
    class MC truth;
    class FF,Div,Inv free;
    class SSM learn;
    class Post,UI store;
    class Conf gate;
    class Splash,Safe safety;
```

**Key.** Purple is ground truth, infrequent but authoritative. Green costs nothing extra and needs no labels. Lavender is the statistical fusion. Amber is the posterior and the confidence gate. Red bypasses all of it. Bold arrows are the observations that move the estimate most.

**Why this diagram matters.** The manual count has two arrows for a reason. It is an observation that moves the estimate a great deal, and it is also what teaches the model how far the camera undercounts, so the correction is estimated from data rather than assumed and it moves when capture conditions change. Restocks and found deaths enter as known events, so the model treats them as observed changes rather than anomalies and nobody gets a false alarm every time stock arrives. The divergence check on the left is free continuous validation, catching the failure a camera cannot report about itself, such as a lens going cloudy. And the splash path is red because statistical confidence is the wrong instrument for a fish on the floor.
