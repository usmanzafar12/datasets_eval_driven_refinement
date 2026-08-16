# Evaluation Datasets

This folder contains the evaluation datasets used in the project. All files share the same schema family (`format_version: "iterative_refinement_v1"`) and are organized into two domains: **smart charging** and **EV routing**.

Every dataset is split into three sets: `train`, `validate`, and `final_evaluate`.

| File | Domain | Type | Split summary | Criteria |
| --- | --- | --- | --- | --- |
| [`ev2gym_simulated_dataset.json`](ev2gym_simulated_dataset.json) | Smart charging | EV2Gym rollout benchmark | 45 / 15 / 15 | `C1`–`C5` |
| [`smart_charging_eval_dataset_old.json`](smart_charging_eval_dataset_old.json) | Smart charging | Legacy short-horizon benchmark | 45 / 15 / 15 | `C1`–`C5` |
| [`routing_real.json`](routing_real.json) | EV routing | Query–answer benchmark (real Doha graph) | 32 / 16 / 24 | `C1`–`C8` |
| [`routing_synthetic.json`](routing_synthetic.json) | EV routing | Query–answer benchmark (synthetic graph) | 32 / 16 / 24 | `C1`–`C8` |
| [`doha_ev_graph_dataset.json`](doha_ev_graph_dataset.json) | EV routing | Underlying real-world graph asset | — | — |
| [`synthetic_graph.json`](synthetic_graph.json) | EV routing | Underlying synthetic graph asset | — | — |

> **Important:** Criterion IDs such as `C2_01` are reused across files but do **not** refer to the same physical scenario. Treat each file as a separate benchmark family. The `C1`–`C5` labels in the smart-charging files are unrelated to the `C1`–`C8` labels in the routing files.

---

## Paper ↔ File Mapping

The paper (`main.tex`) refers to the four benchmark families by acronym. The table below maps those names to the files in this folder.

| Paper name | Full name | Benchmark file | Graph asset |
| --- | --- | --- | --- |
| **SEVR** | Simulated EV Routing | [`routing_synthetic.json`](routing_synthetic.json) | [`synthetic_graph.json`](synthetic_graph.json) |
| **REVR** | Real EV Routing | [`routing_real.json`](routing_real.json) | [`doha_ev_graph_dataset.json`](doha_ev_graph_dataset.json) |
| **SSC** | Synthetic Smart Charging | [`smart_charging_eval_dataset_old.json`](smart_charging_eval_dataset_old.json) | — |
| **RSC** | Real Smart Charging | [`ev2gym_simulated_dataset.json`](ev2gym_simulated_dataset.json) | — |

The implementation and prompt-artifact naming uses a separate convention: **SEVR** → `routing_synthetic`, **REVR** → `routing_real`, **SSC** → `smart_charging_legacy`, and **RSC** → `smart_charging_simulated`.

---

## Smart Charging Datasets

Two smart-charging benchmarks share the criterion labels `C1`–`C5` but use different physical models.

### `ev2gym_simulated_dataset.json` (paper name: **RSC**)

- **Environment:** EV2Gym
- **Physical setting:** one house, one shared **3.6 kW** charger, **1–3 ports** (power is shared across active ports)
- **Time model:** minute-of-day on a `0..1440` horizon (evening home charging)
- **Ground truth:** EV2Gym rollout under the configured shared-power charger

**Criteria**

| ID | Name | Expected behavior |
| --- | --- | --- |
| `C1` | `feasible_baseline` | 1–3 EVs, enough headroom, all EVs resolve |
| `C2` | `fcfs_scarcity_tie_break` | 2–3 EVs, real scarcity window, earlier arrival gets priority, at least one EV unresolved |
| `C3` | `parallel_splitting_benefit` | 3–5 EVs, overlapping windows, charging is split across ports rather than strictly sequential |
| `C4` | `zero_power_boundary_windows` | 2–4 EVs, at least one `0 kW` interval; no charging during the outage |
| `C5` | `congestion_infeasible_heavy` | 4–5 EVs, heavy overlap, total demand intentionally infeasible, multiple unresolved EVs |

**Schema (per sample)**

```jsonc
{
  "id": "C2_01",
  "criteria_id": "C2",
  "criteria_name": "fcfs_scarcity_tie_break",
  "split": "train",
  "expected_parameters": {
    "site_capacity_kw": 16.0292,        // house service limit
    "num_ports": 2,
    "demand_profile_kw": { "1155-1185": 12.4292 }, // raw house demand
    "available_power_kw": { "1155-1185": 3.6 },    // EV charging headroom = site - demand
    "power_kw": { "1155-1185": 3.6 },              // alias of available_power_kw
    "evs": [
      { "id": "EV1", "req_kwh": 4.8609, "arr_min": 1155, "dep_min": 1293 }
    ]
  },
  "expected_result": {
    "EV1": "1155-1185m (3.6kW, P1) | 1185-1259m (1.92kW, P1) | 1259-1260m (1.2kW, P1)",
    "EV2": "1185-1260m (1.92kW, P2) | 1260-1269m (3.6kW, P2) | 1269-1270m (2.4kW, P2)",
    "EV3": "unresolvable",
    "EV4": "unresolvable",
    "EV5": "unresolvable"
  },
  "sample_metadata": {
    "bucket": "C2",
    "notes": ["num_ports_2"],
    "resolved_count": 2,
    "total_delivered_kwh": 7.168,
    "partially_resolved_count": 0
  }
}
```

- `expected_result` is always a map with keys `EV1`–`EV5`.
- A resolved EV uses minute-of-day schedule segments like `1155-1185m (3.6kW, P1)`.
- A partially served EV keeps its realized prefix and ends with `| unresolvable`.
- An untouched or fully infeasible EV is simply `unresolvable`.

### `smart_charging_eval_dataset_old.json` (paper name: **SSC**)

- **Environment:** legacy benchmark (not EV2Gym)
- **Physical setting:** higher-power abstract charging cases, short horizon
- **Time model:** relative minutes on a `0..60` horizon
- **Ground truth:** the stored benchmark schedule itself

Same criterion labels `C1`–`C5`, same `EV1`–`EV5` output map, but schedule segments use relative minutes (e.g. `47-52m (40kW, P1)`) and higher power values. Do not apply EV2Gym home-charging assumptions to this file.

---

## Routing Datasets

Both routing datasets are **query–answer benchmarks** with stored `expected_result` labels. They are built on different graphs and should be interpreted separately.

- `routing_real.json` — queries answered on the **real Doha graph** (`doha_ev_graph_dataset.json`).
- `routing_synthetic.json` — queries answered on a **synthetic graph** (`synthetic_graph.json`) generated with a fixed random seed.

**Criteria**

| ID | Name |
| --- | --- |
| `C1` | Direct Energy Reachability |
| `C2` | Direct End SoC Requirement |
| `C3` | Source-to-Charger Reachability (Green) |
| `C4` | Charger-to-Sink Reachability (Green) |
| `C5` | End SoC Requirement (Green) |
| `C6` | Source-to-Charger Reachability (Regular) |
| `C7` | Charger-to-Sink Reachability (Regular) |
| `C8` | End SoC Requirement (Regular) |

**Schema (per sample)**

```jsonc
{
  "id": "eval_C1_008_Real",
  "criteria_id": "C1",
  "split": "final_evaluate",
  "expected_parameters": {
    "source_label": 6,                    // start node
    "sink_label": 13,                     // destination node
    "start_soc_percentage": 10,           // starting state of charge (%)
    "battery_capacity_kwh": 70.0,
    "consumption_kwh_per_100km": 15.0,
    "cost_attribute": "distance_km",
    "end_soc_percentage": null,           // null = no terminal reserve requirement
    "green_charger_only": false           // restrict chargers to green chargers only
  },
  "expected_result": {
    "route_type": "Direct",               // "Direct" | "Via Charger X" | null
    "cost_m": 13235,                      // total route cost in meters
    "path": [6, 1, 12, 13]                // ordered node sequence
  }
}
```

- `route_type` is `Direct`, `Via Charger X`, or `null`.
- `null` means the query is **infeasible** (not a missing answer). When `route_type` is `null`, `cost_m` and `path` are also `null`.

### `doha_ev_graph_dataset.json` (graph asset)

This file is not a per-query benchmark; it is the underlying real-world graph for `routing_real.json`. It contains `metadata`, `nodes` (with charger annotations and `is_green_charger`), and `edges` (road-distance weights in meters). Nodes are representative Doha anchors/POIs, not an exhaustive road network.

### `synthetic_graph.json` (graph asset)

This file is the underlying synthetic graph for `routing_synthetic.json`. It is not a per-query benchmark.

- `nodes` — 20 nodes (ids `0`–`19`), each with:
  - `id`
  - `is_charging_station` (boolean)
  - `x`, `y` (planar coordinates)
- `edges` — weighted edges, each with:
  - `source`
  - `target`
  - `weight`

Unlike the real Doha graph, this file has no `metadata` block and no `is_green_charger` annotation; charger nodes are marked with `is_charging_station`.

---

## Loading the Data

All files are plain JSON and can be loaded with any JSON parser:

```python
import json

with open("ev2gym_simulated_dataset.json", "r", encoding="utf-8") as f:
    smart_charging = json.load(f)

with open("routing_real.json", "r", encoding="utf-8") as f:
    routing = json.load(f)

samples = routing["evaluation_samples"]
train = [s for s in samples if s["split"] == "train"]
```

```python
import pandas as pd

# Flatten routing samples into a table
df = pd.json_normalize(samples)
```

---

## Additional Documentation

- `CRITERIA.md` — detailed smart-charging criterion definitions and benchmark-distinction notes.
- `routing_criteria.md` — detailed routing criterion definitions and graph semantics.

## License & Citation

_TODO: add license and citation details._
