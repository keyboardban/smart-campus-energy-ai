# Architecture — Chulalongkorn Smart Campus Energy Intelligence

This document describes how the system is put together: its layers, components,
data flow, the multi-agent decision core, and how it is deployed. Diagrams are
written in **Mermaid** (GitHub renders them inline; or paste a block into
[mermaid.live](https://mermaid.live) to export PNG/SVG).

**Contents**
1. [System overview](#1--system-overview)
2. [Layered architecture](#2--layered-architecture)
3. [Data layer](#3--data-layer)
4. [Modeling layer (forecasting)](#4--modeling-layer-forecasting)
5. [Intelligence layer (multi-agent)](#5--intelligence-layer-multi-agent)
6. [Coordinator decision flow](#6--coordinator-decision-flow)
7. [Application layer (the interactive demo)](#7--application-layer-the-interactive-demo)
8. [Deployment & runtime](#8--deployment--runtime)
9. [Key design decisions](#9--key-design-decisions)
10. [Tech stack](#10--tech-stack)

---

## 1 · System overview

The system turns raw sub-metered building data into **forecasts**, **anomaly
flags**, and **comfort-aware energy-saving recommendations**, then exposes a
public, interactive demonstration of the campus-level product. Everything runs
**offline-first**: the notebook needs only the CU-BEMS dataset, and the web demo
is a static page with no backend.

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

## 2 · Layered architecture

The codebase is organized into four layers. Each layer only depends on the one
below it, which keeps the forecasting core independent of how results are shown.

```mermaid
flowchart TB
  subgraph P["Application layer"]
    WEB["Interactive web demo<br/>index.html · vanilla JS"]
    DASH["Static results dashboard<br/>pipeline_demo.html"]
  end
  subgraph INT["Intelligence layer"]
    AGENTS["6 agents + CoordinatorAgent<br/>vet · score · rank"]
  end
  subgraph MOD["Modeling layer"]
    FEAT["Feature engineering"]
    MODELS["Model ladder + metrics"]
    OPT["Optimization + anomaly"]
  end
  subgraph DATA["Data layer"]
    SRC["CU-BEMS loader"]
    QA["Quality audit"]
    AGGR["Power kW + energy kWh"]
  end
  DATA --> MOD
  MOD --> INT
  INT --> P
  classDef l fill:#16162c,stroke:#9B5DE5,color:#fff;
  class P,INT,MOD,DATA l;
```

| Layer | Responsibility | Lives in |
|---|---|---|
| **Data** | load, clean, resample, audit, aggregate (power **kW** vs energy **kWh**) | notebook §1–4 |
| **Modeling** | leakage-safe features, chronological split, model ladder, metrics, anomaly, optimization | notebook §6–10 |
| **Intelligence** | six agents with explicit responsibilities + a coordinator | notebook §11 |
| **Application** | interactive demo (`index.html`) + static dashboard (`pipeline_demo.html`) | repo root |

---

## 3 · Data layer

**Inputs.** 14 CSVs (`2018Floor1.csv` … `2019Floor7.csv`), 1-minute resolution,
columns like `z2_AC1(kW)`, `z1_Light(kW)`, `z1_S1(degC)`.

**Responsibilities.**
- **Robust loading** — auto-detects the dataset folder under `/kaggle/input`,
  coerces timestamps, drops unparseable rows, removes duplicate timestamps.
- **Resampling** — 1-min → 15-min **mean power (kW)** to cut memory ~10×.
- **Quality audit** — missing % and zero % per channel (energy meters are
  reliable; environmental sensors carry the most missing data).
- **Aggregation** — floor/building **power** series with an explicit `_kW`
  suffix, plus environment averages (°C / %RH / lux).

**The unit rule (applied once, everywhere):**
> `energy_kWh = mean_power_kW × interval_hours` (0.25 h for 15-min bins, 1.0 h for hourly).
> Power values are never summed and called kWh.

---

## 4 · Modeling layer (forecasting)

Forecasts building and per-load-type demand at **1 h** and **24 h** horizons
under a strict, leakage-safe protocol.

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

**Why a ladder?** A model is only allowed to "win" if it beats the **baselines**
on the **same** held-out window. **MASE < 1** means "better than seasonal-naïve".
The **foundation model (Amazon Chronos-Bolt)** is optional and zero-shot — it is
evaluated on the *identical* test origins so the comparison is fair, and it skips
cleanly if internet/packages are unavailable.

---

## 5 · Intelligence layer (multi-agent)

Each agent is a real class with one responsibility. They operate on the objects
the modeling layer produces and hand structured outputs to the coordinator.

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

| Agent | Responsibility | Consumes | Produces |
|---|---|---|---|
| `DataQualityAgent` | flag unreliable channels | quality report | reliability score |
| `ForecastingAgent` | forecasts + prediction interval + confidence | models, leaderboard | forecast, expected error, MASE-confidence |
| `ComfortAgent` | comfort violations; **veto** unsafe actions | env sensors, comfort bands | OK / VETO + reason |
| `AnomalyAgent` | surface & explain abnormal hours | forecast residuals + rules | ranked anomalies |
| `OptimizationAgent` | propose savings actions | day-ahead forecast | candidate actions (kWh, peak) |
| `CoordinatorAgent` | vet, score, rank | all of the above | final ranked table |

---

## 6 · Coordinator decision flow

The coordinator is where **conflicts are resolved**. It asks the ComfortAgent to
**veto** any action that would push an already-hot zone past the comfort band,
attaches a confidence from forecast skill and data quality, and ranks survivors
by `savings × confidence × (1 − risk)`.

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

## 7 · Application layer (the interactive demo)

The campus-facing product is demonstrated by `index.html` — a static, zero-dependency
page where the three features run **entirely in the browser**.

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

- **Smart Room Allocation** — a greedy **consolidation optimizer** packs classes
  into the fewest buildings/floors so the rest power down.
- **Dynamic Context-Aware Control** — ASHRAE-55 adaptive-comfort setpoints per
  room, driven by real occupancy/daylight/weather instead of a uniform 25 °C.
- **Anomaly Detection** — forecast-residual + rules flag risky loads left on
  after hours, while ignoring legitimately 24/7 loads.

---

## 8 · Deployment & runtime

There is **no server**. The notebook runs in a data-science runtime; the web
demo is static and runs client-side, hosted on Vercel with auto-deploy on push.

```mermaid
flowchart LR
  DEV["Developer"] -->|git push| REPO["GitHub repo<br/>smart-campus-energy-ai"]
  REPO -->|auto-deploy on push| VC["Vercel<br/>static hosting + CDN"]
  VC -->|HTTPS| BR["Visitor browser<br/>all 3 demos run client-side"]
  REPO -->|notebook source| NB["Kaggle / Colab<br/>Python · scikit-learn · Chronos"]
  DATA[("CU-BEMS dataset")] --> NB
  classDef a fill:#16162c,stroke:#06D6A0,color:#fff;
  class REPO,VC,NB a;
```

| Component | Runtime | Notes |
|---|---|---|
| Forecasting + agents | Kaggle / Colab (Python) | offline-safe; foundation model optional |
| Interactive demo | Vercel (static) → visitor browser | no backend, no build step |
| Source of truth | GitHub | push triggers a Vercel redeploy |

---

## 9 · Key design decisions

- **Leakage safety first.** Features use only data up to the forecast origin;
  the test window is the final calendar slice and is identical for every model.
- **Baselines are first-class.** Persistence and seasonal-naïve are always
  reported; advanced models must beat them on the same window (verdict is data-driven).
- **Foundation model is optional and guarded.** Auto-installs / downloads when
  enabled and online; skips cleanly otherwise. The pipeline never depends on it.
- **Units are explicit.** One helper converts power → energy; all cost/CO₂
  figures flow through it and are flagged as **illustrative assumptions**.
- **Agents over narrative.** "Multi-agent" is implemented as classes with
  explicit responsibilities and real coordination (veto + ranking), not prose.
- **Offline-first, no backend.** Notebook needs only the dataset; the web demo is
  static and dependency-free — trivial to host and to audit.

---

## 10 · Tech stack

`Python` · `pandas` · `NumPy` · `scikit-learn` (Ridge, HistGradientBoosting,
RandomForest, Isolation Forest) · `LightGBM`/`XGBoost` (optional) ·
**Amazon Chronos-Bolt foundation model** (`PyTorch`, optional) · `Matplotlib` ·
custom multi-agent architecture · **vanilla JS / inline SVG** web demo ·
**GitHub + Vercel** for source and hosting.

> CU-BEMS is one building (2018–2019); occupancy is a calendar proxy; energy/cost/CO₂
> numbers are illustrative, not validated bills. See the notebook for full limitations.
