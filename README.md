# Graph ETA Prediction

> Predicting delivery ETAs using graph analytics and network intelligence for smarter logistics operations.

**Live Web Application:** [himanshu-mandal-7d3.streamlit.app](https://himanshu-mandal-7d3.streamlit.app/)

---

## Table of Contents

- [Overview](#overview)
- [Live Demo](#live-demo)
- [Project Structure](#project-structure)
- [Pipeline Architecture](#pipeline-architecture)
- [Models & Performance](#models--performance)
- [Key Outputs](#key-outputs)
- [Getting Started](#getting-started)
- [Running the App](#running-the-app)
- [Tech Stack](#tech-stack)

---

## Overview

Standard ETA engines (such as OSRM) estimate delivery times based purely on static road distance and speed assumptions — ignoring real-world network chokepoints, hub handling inefficiencies, and historical delay patterns.

This project builds a **graph-based intelligence layer** on top of raw delivery trip data to:

- Model the logistics network as a directed graph (hubs as nodes, corridors as edges).
- Compute structural network metrics: betweenness centrality, PageRank, clustering coefficient, and degree distributions.
- Generate **16-dimensional graph embeddings per hub** (32 dimensions total per trip leg encoding source and destination hub topology and historical delay profiles).
- Train and compare **seven models** across baseline and graph-enhanced configurations (OSRM Baseline, Random Forest, CatBoost, and XGBoost).
- Identify bottleneck hubs, detect SLA breach corridors, and quantify recoverable revenue at risk.

---

## Live Demo

Explore the interactive dashboard deployed on Streamlit Cloud:

👉 **[Launch Streamlit Dashboard](https://himanshu-mandal-7d3.streamlit.app/)**

The application enables you to inspect model benchmarks, audit high-risk bottleneck hubs and corridors, assess revenue at risk, review the automated strategy memo, and run real-time graph-enhanced ETA predictions.

---

## Project Structure

```
Graph_ETA_Prediction/
│
├── Dataset/
│   └── delivery_data.csv               # Raw delivery trip dataset
│
├── plots/                              # Generated visualization charts (9 plots)
│   ├── bottleneck_hubs.png             # Top 10 bottleneck hubs by SLA breach & centrality
│   ├── cb_graph_importance.png         # CatBoost + Graph feature importance
│   ├── delay_analysis.png              # Delay ratio distributions by route type & time of day
│   ├── ftl_carting_tradeoff.png        # FTL vs Carting delay comparison across risk tiers
│   ├── model_comparison.png            # 7-model performance comparison (MAE & Acc@15%)
│   ├── network_graph.png               # Network topology visualization of top 50 hubs
│   ├── rf_graph_importance.png         # Random Forest + Graph feature importance
│   ├── xgb_baseline_importance.png     # XGBoost baseline feature importance
│   └── xgb_graph_importance.png        # XGBoost + Graph grouped feature importance
│
├── app_data/                           # Pipeline outputs consumed by Streamlit app
│   ├── audit_df.csv                    # Hub-level audit with risk tier classifications
│   ├── corridors_df.csv                # Corridor-level delay statistics
│   ├── top_breach_corridors.csv        # Top chronic delay corridors
│   ├── revenue_risk.csv                # Revenue at risk segmented by hub profile & route type
│   ├── results.pkl                     # Evaluation metrics, recovery numbers, and memo text
│   ├── rf_graph_model.pkl              # Trained Random Forest + Graph model
│   ├── cb_graph_model.pkl              # Trained CatBoost + Graph model
│   ├── xgb_graph_model.pkl             # Trained XGBoost + Graph model (Best model)
│   ├── enhanced_features.pkl           # Feature list for inference
│   ├── embeddings.pkl                  # 16-dim hub graph embeddings dictionary
│   └── le_time.pkl                     # Time-of-day label encoder
│
├── Graph_ETA_Prediction_Full_Pipeline.ipynb   # End-to-end data, graph & ML pipeline
├── App.py                                     # Streamlit interactive web dashboard
├── strategy_memo.txt                          # Auto-generated operations strategy memo
└── requirements.txt                           # Python dependencies
```

---

## Pipeline Architecture

```
Raw Delivery CSV
       │
       ▼
Data Cleaning & Temporal Feature Engineering
       │
       ▼
Trip Leg Aggregation → Corridor Delay Statistics (median, p90, trip volume)
       │
       ▼
Directed Graph Construction (NetworkX)
       │
       ▼
Graph Metrics (Betweenness Centrality, PageRank, Clustering Coefficient, Degree)
       │
       ▼
Hub Risk Audit → SLA Breach Identification → Revenue Impact Quantification
       │
       ▼
16-dim Graph Embeddings per Hub (32-dim Source + Destination vector)
       │
       ▼
ML Training & Comparison (OSRM vs. RF / CatBoost / XGBoost × Baseline vs. Graph)
       │
       ▼
Model Evaluation (MAE, R², Acc@15%) + Streamlit App + Strategy Memo
```

---

## Models & Performance

All models were evaluated using an **80 / 10 / 10 trip-level split** partitioned strictly by unique `trip_uuid` (Train: 11,853 trips / 115,939 rows; Validation: 1,482 trips / 14,606 rows; Test: 1,482 trips / 14,322 rows) to guarantee zero cross-scan data leakage. Evaluated metrics include Mean Absolute Error (**MAE** in minutes), Coefficient of Determination (**$R^2$**), and Accuracy within 15% tolerance (**Acc@15%**):

| Model | Features | Val MAE (min) ↓ | Val $R^2$ ↑ | Val Acc@15% ↑ | Test MAE (min) ↓ | Test $R^2$ ↑ | Test Acc@15% ↑ |
|---|---|:---:|:---:|:---:|:---:|:---:|:---:|
| **OSRM Baseline** | Pure distance & static speed | 203.77 | 0.6314 | 4.3% | 199.99 | 0.6455 | 4.2% |
| **Random Forest Baseline** | Clean pre-dispatch (13 features) | 46.48 | 0.9791 | 47.4% | 47.95 | 0.9759 | 46.7% |
| **CatBoost Baseline** | Clean pre-dispatch (13 features) | 44.40 | 0.9802 | 49.5% | 46.48 | 0.9792 | 48.3% |
| **XGBoost Baseline** | Clean pre-dispatch (13 features) | 40.75 | 0.9819 | 54.9% | 44.62 | 0.9786 | 52.8% |
| **Random Forest + Graph** | Baseline + 32-dim hub embeddings | 42.88 | 0.9824 | 49.9% | 45.10 | 0.9791 | 48.1% |
| **CatBoost + Graph** | Baseline + 32-dim hub embeddings | 38.43 | 0.9848 | 56.3% | 42.40 | 0.9820 | 54.7% |
| **XGBoost + Graph ⭐** | Baseline + 32-dim hub embeddings | **37.50** | **0.9838** | **61.2%** | **40.41** | **0.9813** | **60.1%** |

### Feature Details

* **Baseline Features (13 features):** `osrm_time`, `osrm_distance`, `segment_osrm_time`, `segment_osrm_distance`, `source_center_enc`, `destination_center_enc`, `route_type_FTL`, `time_of_day_encoded`, `day`, `month`, `weekday`, `od_hour`, `od_weekday`. *(Note: Leaky post-hoc flags `actual_distance_to_destination`, `is_cutoff`, and `cutoff_factor` were eliminated to prevent data leakage).*
* **Graph Embeddings (16 dims per hub, 32 dims total per trip leg):**
  * *Topological metrics (6 dims):* Normalized betweenness centrality, normalized in-degree, normalized out-degree, clustering coefficient, normalized PageRank, out/in degree ratio.
  * *Outbound delay profile (5 dims):* Mean delay ratio, max delay ratio, std delay ratio, trip count, p90 delay ratio.
  * *Inbound delay profile (5 dims):* Mean delay ratio, max delay ratio, std delay ratio, trip count, p90 delay ratio.

---

## Key Outputs

| Output File | Description |
|---|---|
| `plots/bottleneck_hubs.png` | Top 10 hubs by SLA breach trip volume and centrality |
| `plots/network_graph.png` | Network graph topology visualization of top 50 hubs |
| `plots/delay_analysis.png` | Delay ratio distribution by route type and time of day |
| `plots/model_comparison.png` | Side-by-side MAE and Acc@15% comparison across all 7 models |
| `plots/xgb_baseline_importance.png` | Feature importance for the XGBoost baseline model |
| `plots/rf_graph_importance.png` | Feature importance for Random Forest + Graph model |
| `plots/cb_graph_importance.png` | Feature importance for CatBoost + Graph model |
| `plots/xgb_graph_importance.png` | Grouped feature importance for the best XGBoost + Graph model |
| `plots/ftl_carting_tradeoff.png` | FTL vs. Carting delay comparison segmented by hub risk tier |
| `strategy_memo.txt` | Automated executive strategy memo with bottleneck analysis & intervention ROI |

---

## Getting Started

### 1. Clone the repository

```bash
git clone https://github.com/Himanshu7d3/Graph_ETA_Prediction.git
cd Graph_ETA_Prediction
```

### 2. Install dependencies

```bash
pip install -r requirements.txt
```

### 3. Run the analysis notebook

Open and run all cells in `Graph_ETA_Prediction_Full_Pipeline.ipynb` top to bottom. This processes the raw dataset, builds the graph embeddings, trains the models, and populates `app_data/` and `plots/`.

### 4. Run the Streamlit app locally

```bash
streamlit run App.py
```

---

## Running the App

### Online Deployed App
Access the live deployed version directly in your browser:
🔗 **[himanshu-mandal-7d3.streamlit.app](https://himanshu-mandal-7d3.streamlit.app/)**

### Local Streamlit App
Once the notebook has run and populated `app_data/`:

```bash
streamlit run App.py
```

---

## Tech Stack

| Library | Purpose |
|---|---|
| `pandas` | Data loading, cleaning, manipulation, and aggregations |
| `numpy` | Numerical operations, vector math, and embedding construction |
| `networkx` | Directed graph construction and centrality metric computations |
| `scikit-learn` | Random Forest models, Decision Trees, preprocessing, and evaluation metrics |
| `xgboost` | Gradient boosted trees for baseline and graph-enhanced ETA models |
| `catboost` | Gradient boosting regressor with categorical handling |
| `matplotlib` | Visualizations, feature importance plots, and network figures |
| `streamlit` | Interactive operational web application ([`App.py`](App.py)) |
