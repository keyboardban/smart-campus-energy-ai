# Chulalongkorn Smart Campus Energy Intelligence — Pipeline & Architecture

Diagrams below are written in **Mermaid**. Any of these turn them into a picture:
- **GitHub** — renders ```mermaid blocks automatically in a README/`.md`.
- **mermaid.live** — paste a block → *Actions → PNG/SVG*.
- **VS Code** — “Markdown Preview Mermaid Support” extension.
- **CLI** — `mmdc -i ARCHITECTURE.md -o diagram.png` (`@mermaid-js/mermaid-cli`).

---

## 1 · System overview (guaranteed offline pipeline)

```mermaid
flowchart LR
  A[("CU-BEMS CSVs<br/>7 floors · 2018-2019")] --> B["Load + resample<br/>1-min to 15-min mean power (kW)"]
  B --> C["Data-quality audit<br/>missing % per channel"]
  C --> D["Aggregate power (kW)<br/>building / floor / load-type"]
  D --> E["kW to kWh<br/>energy = mean_kW x interval_h"]
  D --> F["Hourly modeling frame"]
  F --> G["Leakage-safe features<br/>lags · rolling · calendar"]
  G --> H["Chronological split<br/>70 / 15 / 15"]
  H --> I["Model ladder"]
  I --> J["Metrics<br/>MAE · RMSE · sMAPE · MASE · peak"]
  J --> K[["Leaderboard"]]
  I --> L["Residual anomaly detection"]
  I --> M["Forecast-informed optimization"]
  K --> N{{"Multi-agent coordinator"}}
  L --> N
  M --> N
  E --> M
  N --> O[["Ranked recommendations"]]

  classDef data fill:#1b2a4a,stroke:#00B4D8,color:#fff;
  classDef model fill:#3a2417,stroke:#FF6B35,color:#fff;
  classDef out fill:#1c2e22,stroke:#06D6A0,color:#fff;
  class A data;
  class I,J model;
  class K,O,N out;
```

---

## 2 · Forecasting pipeline (leakage-safe detail)

```mermaid
flowchart TB
  subgraph FEAT["Features — use only data up to origin t"]
    direction LR
    L1["lags<br/>1,2,3,24,48,168h"]
    L2["rolling<br/>mean/std/min/max 24h"]
    L3["calendar + cyclic<br/>hour, dow, sin/cos"]
  end

  subgraph SPLIT["Chronological split (no shuffle)"]
    direction LR
    TR["Train 70%"] --> VA["Val 15%<br/>model selection"] --> TE["Test 15%<br/>report once"]
  end

  subgraph LADDER["Model ladder"]
    direction TB
    B1["Persistence (baseline)"]
    B2["Seasonal-naive (baseline)"]
    M1["Ridge"]
    M2["HistGradientBoosting"]
    M3["RandomForest"]
    M4["LightGBM / XGBoost (optional)"]
    M5["Chronos-Bolt zero-shot (optional)"]
  end

  TGT[/"Targets: Bldg_Total, AC, Light, Plug, Floor_n<br/>Horizons: 1h and 24h"/] --> FEAT
  FEAT --> SPLIT
  SPLIT --> LADDER
  LADDER --> EVAL["Evaluate on identical test window"]
  EVAL --> METR["MAE · RMSE · sMAPE · MASE · peak-error<br/>+ error by hour / weekday"]
  METR --> LB[["Leaderboard + evidence-based verdict"]]
```

---

## 3 · Multi-agent architecture

```mermaid
flowchart TD
  RAW[("Raw CU-BEMS channels")] --> DQA
  HRLY[("Hourly load series")] --> FA
  ENV[("Indoor temp / RH / lux")] --> CA

  DQA["DataQualityAgent<br/>flag unreliable channels"]
  FA["ForecastingAgent<br/>forecast + interval + confidence"]
  CA["ComfortAgent<br/>violations + veto power"]
  AA["AnomalyAgent<br/>residual z-score + rules"]
  OA["OptimizationAgent<br/>propose savings actions"]
  CO{{"CoordinatorAgent<br/>reject unsafe · score · rank"}}

  DQA -->|reliability score| CO
  FA -->|forecast + expected error| AA
  FA -->|day-ahead AC forecast| OA
  FA -->|forecast skill / MASE| CO
  CA -->|VETO comfort-violating| CO
  AA -->|ranked anomalies| CO
  OA -->|candidate actions| CO
  CO --> OUT[["Ranked recommendation table<br/>kWh · peak · comfort risk · confidence"]]

  classDef agent fill:#241a3a,stroke:#9B5DE5,color:#fff;
  classDef coord fill:#3a1f2c,stroke:#EF476F,color:#fff;
  classDef src fill:#13233f,stroke:#00B4D8,color:#fff;
  class DQA,FA,CA,AA,OA agent;
  class CO coord;
  class RAW,HRLY,ENV src;
```

---

## 4 · Coordinator decision flow (how conflicts are resolved)

```mermaid
sequenceDiagram
  autonumber
  participant O as OptimizationAgent
  participant C as CoordinatorAgent
  participant Cf as ComfortAgent
  participant F as ForecastingAgent
  participant D as DataQualityAgent

  O->>C: propose candidate actions
  loop for each action
    C->>Cf: is this comfort-safe?
    Cf-->>C: OK  /  VETO (+ reason)
    C->>F: forecast confidence for target?
    F-->>C: skill (MASE to confidence)
    C->>D: data reliability for channel?
    D-->>C: reliability score
    Note over C: score = savings x confidence x (1 - risk)<br/>vetoed actions get score 0
  end
  C-->>O: ranked table (comfort-violating actions excluded)
```

---

## 5 · Smart-campus product flow (the interactive demo)

```mermaid
flowchart LR
  REG[/"Registrar data<br/>courses · times · students · room caps · building limits"/] --> ALLOC

  ALLOC["1 · Smart Room Allocation<br/>consolidation optimizer"]
  ALLOC -->|rooms still needed| CTRL
  ALLOC -->|idle rooms| OFF["Power down<br/>whole floors / buildings"]

  CTX[/"Real conditions<br/>outdoor temp · occupancy · daylight"/] --> CTRL
  CTRL["2 · Dynamic Context-Aware Control<br/>adaptive AC setpoint per room (not fixed 25C)"]
  CTRL --> MON

  MON["3 · Anomaly Detection<br/>forecast-residual + rules"]
  MON -->|risky load left on after hours| ALERT["Auto-alert facilities<br/>lab PCs · research gear · projectors"]
  MON -->|normal 24/7 load| OKK["No alert"]

  classDef step fill:#2a1a10,stroke:#FF6B35,color:#fff;
  classDef io fill:#13233f,stroke:#00B4D8,color:#fff;
  class ALLOC,CTRL,MON step;
  class REG,CTX io;
```

---

## 6 · Rendering to an image — quick reference

| Tool | Command / action | Output |
|---|---|---|
| GitHub | commit this `.md` | inline render |
| mermaid.live | paste a block → **Actions** | PNG / SVG |
| Mermaid CLI | `npm i -g @mermaid-js/mermaid-cli` then `mmdc -i ARCHITECTURE.md -o arch.png -t dark -b transparent` | PNG/SVG per diagram |
| VS Code | *Markdown Preview Mermaid Support* extension → preview → screenshot | PNG |

> Tip: for slides, render each block individually on **mermaid.live** with theme **dark** and a transparent background, then export SVG (scales cleanly).
