# Hybrid Intelligent Proactive Routing Agent — Chicago Traffic

Signal-aware route planning for Chicago's arterial road network. An **XGBoost** model predicts segment
speeds by hour, day and season; those predictions are turned into **derived traffic-signal delays**, and a
from-scratch **A\*** search plans routes over a time-dependent road graph — compared against Dijkstra and a
signal-agnostic A\* baseline.

Built as a team project for an NTU AI algorithms course, using the
[Chicago Traffic Tracker](https://data.cityofchicago.org/browse?q=Chicago%20Traffic%20Tracker%20Historical%20Congestion%20Estimates%20by%20Segment)
open dataset.

## How it works

```
Chicago Traffic Tracker CSV
        │
        ▼
 Perception layer ── Chicago_speed_predict.ipynb
   cleaning · cyclical time features · segment/time baselines · XGBoost regressor
        │  predicted speed per (segment, hour)
        ▼
 Decision layer ──── astar_traffic_signal.ipynb
   signal delay  = 30% of (predicted − free-flow) delay   (HCM 6th ed. guidance)
   road graph    = nodes: intersections · edges: segments weighted by travel + signal time
   A* search     = haversine distance / free-flow speed (admissible heuristic)
        │
        ▼
 Benchmarks · hour-by-hour sweep · ablations · scalability · interactive Folium map
```

### 1. Speed prediction (XGBoost)

- Cleans sensor faults (`-1` = sensor offline, speeds > 70 mph, zero-length segments, missing coordinates).
- Engineers temporal features: sin/cos encoding of hour and month, day of week, quarter, weekend flag and
  coarse time blocks (overnight, morning rush, midday, evening rush, evening).
- Adds hierarchical baselines — segment × day × hour, segment × time block, segment × month — with
  fallbacks so unseen combinations never produce NaNs.
- Trains an `XGBRegressor` (hist trees, early stopping) on a chronological 80/20 split.

**Test-set results:** MAE **3.18 mph** · RMSE **4.23 mph** · R² **0.633**

### 2. Signal-aware routing (A\*)

- **Signal penalty** derived from physics, not a separate dataset: free-flow speed is the 95th percentile of
  predicted speeds, and a time-of-day fraction of the excess delay (≈30% at baseline, per the Highway Capacity
  Manual) is attributed to signals at the downstream intersection.
- **Two graphs with identical topology** — one weighted by travel + signal time, one by travel time only —
  so any route difference is caused only by the signal penalty.
- **A\*** implemented from scratch with `heapq`; the heuristic is straight-line (haversine) distance at
  free-flow speed, which never overestimates. Correctness is checked by matching Dijkstra's cost on the
  no-signal graph.

### 3. Experiments

| Section | What it measures |
|---|---|
| Single-query demo | End-to-end path, A\* vs Dijkstra cost check, driving vs signal-wait breakdown |
| 30-query benchmark | Cost distributions, signal overhead, nodes explored per method |
| Hour-by-hour sweep | How signal overhead tracks predicted congestion across 24 hours |
| Ablation study | Effect of removing the signal penalty, the dynamic fraction and the XGBoost speeds |
| Peak vs off-peak | 8 am vs 2 pm — does signal-aware routing change more routes at peak? |
| Scalability | Nodes explored vs graph size for Dijkstra, A\* and bidirectional A\* |
| Route map | Both routes on an interactive Folium map, with high-delay junctions highlighted |

## Repository layout

| File | Purpose |
|---|---|
| `Chicago_speed_predict.ipynb` | EDA and XGBoost speed model (perception layer); written for Google Colab |
| `astar_traffic_signal.ipynb` | Signal penalties, graph construction, A\*, benchmarks and maps (decision layer) |
| `routing_code_explanation.txt` | Section-by-section walkthrough of the routing notebook |
| `requirements.txt` | Python dependencies |

## Running it

```bash
python3 -m venv .venv && source .venv/bin/activate
pip install -r requirements.txt
jupyter notebook
```

**Quick start (no download):** `astar_traffic_signal.ipynb` runs end to end on generated synthetic data with
`USE_SYNTHETIC = True` (the default).

**Real data:**

1. Download the *Chicago Traffic Tracker – Historical Congestion Estimates by Segment* CSV from the
   [City of Chicago Data Portal](https://data.cityofchicago.org/browse?q=Chicago%20Traffic%20Tracker%20Historical%20Congestion%20Estimates%20by%20Segment)
   (the file used here is ~214 MB, so it is not committed).
2. Run `Chicago_speed_predict.ipynb` with `file_path` pointing at the CSV; it saves the model as
   `chicago_speed_precision_final.pkl`.
3. In `astar_traffic_signal.ipynb`, set `USE_SYNTHETIC = False`, point `TRAFFIC_PATH` at the CSV and place the
   `.pkl` next to the notebook (without it, the notebook falls back to segment × day × hour averages).
4. Adjust `QUERY_HOUR`, `FROM_LOCATION` and `TO_LOCATION` to explore different trips and times.

## Tech stack

Python · pandas · NumPy · XGBoost · scikit-learn · NetworkX · Matplotlib · Seaborn · Folium
