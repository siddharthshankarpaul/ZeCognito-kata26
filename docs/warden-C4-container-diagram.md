# The Warden Platform:  C4 Container Diagram

**Client:** Von Digitalis Estates
**Notation:** C4 Container diagram (Level 2) with a deterministic/advisory plane overlay 
**Style:** Service-based core + event-driven edge + CQRS

---

## C4 Level 2 (Container)

```mermaid
flowchart TB
  %% ============ PEOPLE ============
  VIS(["Visitor<br/>[Person]"]):::person
  STAFF(["Estate Ops &amp; Countess<br/>[Person]"]):::person
  KEEP(["Keepers &amp; Vets<br/>[Person]"]):::person
 
  %% ============ EXTERNAL SYSTEMS ============
  PSP["Payment Gateway<br/>[External]"]:::ext
  LLM["LLM Providers<br/>[External]"]:::ext
  NOTIF["Email / Push / SMS<br/>[External]"]:::ext
 
  %% ============ SYSTEM BOUNDARY ============
  subgraph WARDEN["WARDEN PLATFORM  ·  every box = [Container]"]
    direction TB
 
    subgraph EDGE["On-estate edge · offline-first"]
      direction TB
      CAM["Cameras"]:::edge
      ANON["Edge Anonymiser"]:::edge
      SENS["MQTT Sensors"]:::edge
      RDR["Gate Readers"]:::edge
      LOOK["Lookout<br/>Edge AI"]:::advedge
      VAL["Offline Validator"]:::edge
      BRK["Edge MQTT Broker"]:::edge
      SYNC["Edge Gateway / Sync"]:::edge
    end
 
    ING["Ingestion Bridge"]:::tx
    BUS[["Event Backbone"]]:::tx
 
    subgraph TX["Transactional plane · deterministic"]
      direction TB
      PWA["Visitor PWA"]:::tx
      PORTAL["Ops Portal"]:::tx
      APIGW["API Gateway / BFF"]:::tx
      IDC["Identity &amp; Consent"]:::tx
      CART["Cart"]:::tx
      TICK["Ticketing"]:::tx
      PAY["Payments"]:::tx
      ACC["Access &amp; Entitlement"]:::tx
      PAPPLY["Pricing"]:::tx
      OBS["Observability &amp; Audit"]:::tx
    end

    GATE{{"Decision Gate<br/>the ONLY governed bridge"}}:::gate
 
    subgraph ADV["Advisory plane · non-deterministic AI"]
      direction TB
      AUG["Augur<br/>LLM Gateway"]:::adv
      GUIDE["Guide"]:::adv
      PAI["Pricing Advisor"]:::adv
      ARK["Ark<br/>Animal Welfare"]:::adv
      FOOT["Footfall Analytics"]:::adv
      RET["Retention"]:::adv
      PROFIT["Profitability Advisor"]:::adv
      EVAL["Eval &amp; Validation"]:::adv
    end
 
    STREAM["Stream Processing"]:::data
    LAKE[("Data Lake / Features")]:::data
    READ[("Read Models / BI")]:::data
  end
 
  %% ---------- DETERMINISTIC (solid) ----------
  VIS -->|HTTPS| PWA
  STAFF -->|HTTPS| PORTAL
  KEEP -->|HTTPS| PORTAL
  PWA --> APIGW
  PORTAL --> APIGW
  APIGW --> IDC
  IDC -->|entitlement| TICK
  APIGW --> CART --> TICK
  TICK --> PAY -->|charge| PSP
  PAY --> ACC
  ACC -->|signed ticket| PWA
  ACC --> BUS
  RDR -->|scan| VAL
  VAL -->|admission| BRK
  ACC -->|revocations / cache| SYNC
  SYNC --> ING --> BUS
  BUS -->|reconcile| ACC
  READ --> PORTAL
  OBS --> PORTAL
 
  %% ---------- CONCIERGE (dotted) ----------
  PWA -.->|chat| GUIDE
  GUIDE -.-> AUG
  GUIDE -.->|proposed cart| CART
  PWA -->|human confirm + pay| PAY
 
  %% ---------- DYNAMIC PRICING (dotted -> gate -> solid) ----------
  FOOT -.->|demand| PAI
  PAI -.->|proposed price| GATE
  GATE -->|approved| PAPPLY
  PAPPLY --> TICK
 
  %% ---------- EDGE TELEMETRY + CAMERA PRIVACY ----------
  SENS --> BRK
  CAM -.->|raw frames| ANON
  ANON -.->|anon frames| LOOK
  LOOK -.->|counts| BRK
  BRK --> SYNC
  ING --> STREAM --> LAKE
  BUS --> STREAM
 
  %% ---------- WELFARE / PIRANHA (dotted) ----------
  ARK -.->|features| LAKE
  LOOK -.->|piranha + behaviour| ARK
  ARK -.-> AUG
  ARK -.->|alert| NOTIF
  NOTIF -.-> KEEP
  ARK -.->|welfare model| READ
 
  %% ---------- FOOTFALL (dotted) ----------
  LOOK -.->|occupancy / dwell| FOOT
  FOOT -.-> LAKE
  FOOT -.->|popularity model| READ
 
  %% ---------- RETENTION + CONSENT (dotted) ----------
  RET -.->|history| LAKE
  RET -.->|consent check| IDC
  RET -.-> AUG
  RET -.->|offer| NOTIF
  NOTIF -.-> VIS
 
  %% ---------- PROFITABILITY (dotted) ----------
  PROFIT -.->|revenue+cost+footfall| LAKE
  ARK -.->|care cost| PROFIT
  FOOT -.->|popularity| PROFIT
  PROFIT -.-> AUG
  PROFIT -.->|investment model| READ
 
  %% ---------- PORTABILITY + V&V + AUDIT (dotted / logged) ----------
  AUG -.->|routed + fallback| LLM
  AUG -.->|log call| OBS
  GATE -->|log decision| OBS
  EVAL -.->|evidence| OBS
  EVAL -.->|judge| AUG
  EVAL -.->|monitor| GUIDE
  EVAL -.->|monitor| ARK
  EVAL -.->|monitor| PAI
  EVAL -.->|monitor| FOOT
  EVAL -.->|monitor| RET
  EVAL -.->|monitor| PROFIT
 
  %% ---------- FEEDBACK INTO EVAL (dotted) ----------
  KEEP -.->|alert verdict = truth| EVAL
  STAFF -.->|spot-count labels| EVAL
 
  %% ============ STYLES ============
  classDef person fill:#08427b,stroke:#052e56,color:#ffffff;
  classDef edge fill:#e8f5e9,stroke:#2e7d32,color:#1b5e20;
  classDef advedge fill:#fff3e0,stroke:#2e7d32,stroke-dasharray:4 3,color:#4e342e;
  classDef tx fill:#e3f2fd,stroke:#1565c0,color:#0d47a1;
  classDef adv fill:#fff3e0,stroke:#e65100,stroke-dasharray:4 3,color:#4e342e;
  classDef data fill:#f5f5f5,stroke:#616161,color:#212121;
  classDef gate fill:#ffebee,stroke:#c62828,stroke-width:2px,color:#b71c1c;
  classDef ext fill:#fafafa,stroke:#9e9e9e,stroke-dasharray:2 2,color:#424242;
```
 
---
 
## Key (notation)
 
| Notation | Meaning |
|---|---|
| **`[Person]` dark rounded node** | Human actor, outside the boundary. |
| **`[External]` dashed node** | System Warden depends on but doesn't own. |
| **Outer box "Warden Platform"** | System boundary, every box inside is a `[Container]`. |
| **Solid arrow →** | Deterministic path: money, access, entitlements, identity, audit. |
| **Dotted arrow ⇢** | Advisory / AI path: inference, recommendations, analytics. |
| **Blue** | Transactional plane container (deterministic). |
| **Amber dashed** | Advisory plane container (AI). |
| **Green** | On-estate edge container. |
| **Green dashed** | Edge AI (Lookout). |
| **Red hexagon** | Decision gate, the only place AI can influence money/access, after rules approve. |
| **Grey cylinder** | Data / read store (CQRS read side). |
 
---
 
## Container legend (full descriptions)
 
| Node | Container | What it does |
|---|---|---|
| PWA | Visitor Web / PWA | Buy, plan, concierge UI; holds the signed ticket. |
| PORTAL | Operations Portal | Estate, animal, and commercial dashboards for staff. |
| APIGW | API Gateway / BFF | Auth, routing, throttling. |
| IDC | Identity & Consent | Profile, consent record, returning-visitor ID. |
| CART | Booking / Cart | Order assembly. |
| TICK | Ticketing | Tickets, family/season pass, N-admit entitlement token. |
| PAY | Payments | Charges via external gateway. |
| ACC | Access & Entitlement | Issues Ed25519-signed tickets; validates & reconciles admissions. |
| PAPPLY | Pricing | Applies approved price within guardrails. |
| GATE | Decision Gate | Deterministic rules engine: bounds & policy on AI proposals. |
| OBS | Observability & Audit | Logs, metrics, traces, model calls, gated decisions, eval evidence, immutable. |
| ING | Ingestion Bridge | Cloud MQTT ingestion from the estate. |
| BUS | Event Backbone | Messaging; CQRS commands & events. |
| CAM | Cameras | On-estate capture for footfall and welfare CV; raw frames never leave this node. |
| LOOK | Lookout (Edge AI) | On-estate CV: footfall count, dwell, piranha count, behaviour. |
| ANON | Edge Anonymiser | On-node de-identification; no faces leave the edge. |
| SENS | MQTT Sensors | Feed-weight, water quality, environment. |
| RDR | Gate Readers | QR / RFID / BLE. |
| VAL | Offline Validator | Ed25519 verify + local cache; works offline. |
| BRK | Edge MQTT Broker | Store-and-forward, QoS 1. |
| SYNC | Edge Gateway / Sync | Aggregates & resumes on connectivity. |
| AUG | Augur (LLM Gateway) | Model-agnostic routing, fallback, cost caps. |
| GUIDE | Guide (Concierge) | Agentic cart; human-gated checkout. |
| PAI | Dynamic Pricing Advisor | Demand-based; no individual profiling. |
| ARK | Ark (Animal Welfare) | Health anomaly, feeding analysis, piranha population. |
| FOOT | Footfall Analytics | Popularity, occupancy, dwell. |
| RET | Retention & Personalisation | Consent-checked offers. |
| PROFIT | Profitability / Investment Advisor | Fuses revenue, care cost, footfall. |
| EVAL | Eval & Validation Harness | Golden sets, LLM-as-judge, shadow/canary, drift. |
| STREAM / LAKE / READ | Data platform | Stream processing → data lake / feature store → CQRS read models. |
 
---
 
## Sub-problem coverage (traceability)
 
| Sub-problem | Primary path |
|---|---|
| Tickets & family passes, access | `VIS → PWA → APIGW → IDC → TICK → PAY → PSP`; `ACC` signed tickets; `VAL` offline; reconcile via `BUS` |
| Footfall / popularity | `CAM → ANON → LOOK → FOOT → READ → PORTAL` |
| Animal welfare | `SENS/CAM → ANON → LOOK/BRK → LAKE → ARK → NOTIF → KEEP` |
| Piranha population | `CAM → ANON → LOOK → ARK → READ` |
| Growth / profitability / retention | `RET → NOTIF → VIS`; pricing `FOOT → PAI → GATE → PAPPLY → TICK`; `PROFIT → READ → PORTAL` |
| Identity & consent | `APIGW → IDC → TICK`; `RET ⇢ IDC` |
| Model portability | all AI `⇢ AUG ⇢ LLM` |
| V&V + monitoring + feedback | `EVAL ⇢ {GUIDE, ARK, PAI, FOOT, RET, PROFIT}`; `KEEP/STAFF ⇢ EVAL`; `AUG/GATE/EVAL → OBS` |
 
---

<p align="right"><a href="../README.md">↑ Back to README</a></p>