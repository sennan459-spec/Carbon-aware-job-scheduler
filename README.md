# Carbon-aware job scheduler

Flexible computer jobs, like ML training, builds and backups, often don't need to run right away. They just need to finish by a deadline. The electricity grid's carbon intensity (grams of CO2 per kWh) rises and falls through the day as solar and wind come and go. This project moves flexible jobs into the cleanest hours that still meet their deadlines, using 96-hour carbon intensity forecasts from [CarbonCast](https://github.com/UMass-LIDS/CarbonCast). CarbonCast is a model that takes region(s) as input and outputs a 96 hour forecast of carbon intensity for the region at a specific time.

**Goal:** a result of the form *"cut emissions X% across N real AI jobs"*, compared with running every job the moment it's submitted.

> Status: in progress. Results will be added here once the simulator runs.

## How it works

The project replays the past to answer a "what if" question:

1. **What happened.** Each real job runs as soon as it was submitted, and is charged the grid's actual carbon intensity during the hours it ran.
2. **What if.** The same jobs, on the same real grid, but the scheduler picks each start time. It only sees the forecast that would have existed at the time, so it can't peek at the future.
3. **The difference** between the two is the emissions the scheduler saves.

```
CarbonCast forecasts ─┐
                      ├─► Scheduler ─► start times ─► Scorer ─► Report
Alibaba job trace ────┘                                 ▲
                                 Actual grid intensity ─┘
```

The **simulator** steps through the test period one day at a time. Each day it shows the scheduler that day's 96-hour forecast and the jobs that have just arrived. The **scorer** then uses actual intensity, never the forecast, to work out each job's emissions:

```
emissions = power (kW) × sum of actual intensity (gCO2/kWh) over the hours the job ran
X = 100 × (1 − scheduled emissions / run-immediately emissions)
```

## Scheduling policies

| Policy | What it does |
| --- | --- |
| `run_immediately` | Starts each job as soon as it arrives (the baseline) |
| `random_in_window` | Picks any start time that still meets the deadline |
| `night_only` | Runs at a fixed hour (e.g. 2am), a common rule of thumb |
| `forecast_greedy` | Picks the start with the lowest total forecast intensity (the scheduler itself) |
| `oracle` | Same as greedy but uses actual intensity: the best possible result |

The gap between `forecast_greedy` and `oracle` shows how much forecast error costs.

## Data

### Carbon intensity forecasts: CarbonCast
Hourly 96-hour forecasts plus actual carbon intensity, starting with California (CISO). CarbonCast itself is not included in this repo; it's run once separately and its output is saved as a CSV:

```
data/forecasts/CISO.csv
issued_at, target_hour, forecast_gco2_kwh, actual_gco2_kwh
```

Maji, Shenoy, Sitaraman. *CarbonCast: Multi-Day Forecasting of Grid Carbon Intensity.* BuildSys '22.

### Jobs: Alibaba spot GPU trace (2026)
Real AI jobs from Alibaba's GPU clusters ([`cluster-trace-v2026-spot-gpu`](https://github.com/alibaba/clusterdata/tree/master/cluster-trace-v2026-spot-gpu)). The trace has 466,867 jobs over about 6 months. We use the **51,154 "Spot" jobs**: low-priority work that Alibaba already runs on spare capacity, which makes it a natural fit for flexible jobs.

Fields used: `submit_time`, `duration` (seconds), `gpu_request`, `worker_num`, `gpu_model`, `job_type`.

The trace doesn't record two things, so we add them and state them in the results:
- **Deadline:** each job gets a slack (e.g. 12, 24 or 48 hours) after submission. Results are shown for each slack level.
- **Power draw:** estimated from GPU model and GPU count.

The trace's timestamps are relative (seconds from the first job), so they're mapped onto the CarbonCast test period. The jobs ran in Alibaba's data centers, not on the California grid; this study asks what would happen if they had run there.

Please cite the trace's paper if you reuse it: Duan et al., *GFS: A Preemptive Scheduling Framework for GPU Clusters with Predictive Spot Management*, ASPLOS '26.

## Results (coming)

- Headline: X% emissions cut across N jobs, with deadlines met and a range over several runs
- One example week: the real intensity curve with jobs drawn where they ran, before and after
- Emissions by policy
- Savings vs. how much slack each job has (0 to 72 hours)

## Project layout (planned)

```
data/
  forecasts/     CarbonCast output (CSV)
  jobs/          Alibaba trace and the cleaned job list
scheduler/
  jobs.py        load the trace, add deadlines and power
  policies.py    the five policies
  simulator.py   day-by-day replay
  scorer.py      emissions and the X% number
  api.py         small FastAPI service around the scheduler
results/         charts and tables
tests/
```

## Setup (planned)

```
pip install -r requirements.txt
pytest
python -m scheduler.simulator --region CISO --slack 24
```

Built with Python, pandas, NumPy, matplotlib, FastAPI and pytest.
