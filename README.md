# Wind Farm Analytics Opportunity Assessment — TLDR

An exploratory analysis of the public [Penmanshiel Wind Farm dataset](https://zenodo.org/records/5946808) (14 × Senvion MM82 turbines, 28.7 MW, ~5 years of 10-minute SCADA data) to find **failure-agnostic, farm-agnostic** analytics that improve turbine performance, reliability and availability.

## The Idea

Build a **power-curve performance monitor**: learn what "normal" power output looks like at each wind speed during a clean baseline period, then flag when a turbine produces less than expected. It is designed to be a first-layer detector that works on any farm without being tuned to a specific failure mode.

## What Was Done

- **Scoped to Turbine 12** (fewest status events → most clean data) across its full lifetime (262,702 rows, Jul 2016 – Jun 2021).
- **Cleaned the data** by removing readings that fail quality filters: logged status events, zero data availability, gusty wind (std dev ≥ 2 m/s), and curtailment (available capacity below planned). This left ~48% of the data, mostly clean from Q3 2018 onward.
- **Selected a baseline window** (Q3 2018 – Q2 2019) where the cleaned data forms the expected S-shaped power curve, binned into 0.5 m/s bins (IEC 61400-12). 36 of 51 bins were well-populated, covering the full operating range (0–18 m/s).
- **Built and ran the monitor**, scoring each reading against its baseline bin and tracking a 24-hour rolling performance score (% of rated power).

## Key Results

- **Baseline is unbiased** — mean residual during the baseline period was **+0.05%** with 0% below the alert threshold (the sanity check passed).
- **216,705 readings scored**, overall mean residual **−0.18%**.
- **3 sustained underperformance alerts** detected (rolling score below −5% of rated power):

  | Period | Duration | Mean Score | Worst |
  |---|---|---|---|
  | Aug 2016 | 47 hrs | −20.6% | −27.6% |
  | Nov 2016 | 21 hrs | −7.6% | −8.3% |
  | **Dec 2020** | **121 hrs** | **−23.5%** | **−37.2%** |

- **Alerts matched real faults.** Cross-referencing against the status logs confirmed the monitor catches genuine operational problems, not noise:
  - Aug 2016 alert → externally-reduced output + a generator-brush service stop.
  - Dec 2020 alert (worst case) → cascade of max-wind-speed stops, cable autounwind, manual stop and pitch battery faults.

## Why It Scales

- Needs only **four common SCADA inputs**: density-adjusted wind speed, power, an availability flag, and a capacity/setpoint column.
- Makes **no assumption about the failure mode** — learns "normal" from the data itself.
- **Multi-turbine ready** via a `TURBINES` list; each turbine gets its own baseline.
- **New farm deployment** needs only a clean baseline window and matching column names; rated power is derived automatically and the −5% threshold is relative, so it means the same on a 2 MW or 4 MW turbine.
- **Scoring is fully vectorised** — hundreds of thousands of readings score in milliseconds.

## Limitations

Demonstrated on one turbine only; the monitor flags *that* a turbine underperforms but not *which component* is at fault; risk of baseline contamination by undetected gradual degradation; nacelle-anemometer wind-speed bias; and a fixed, non-adaptive −5% threshold with a manually-chosen baseline window.

## Next Steps

1. **Normal-behaviour modelling** (temperature/vibration/electrical) as a second stage to localise faults to a subsystem.
2. **Run across all 14 turbines** and compare scores (simultaneous dips = site-level issue; isolated dip = component fault).
3. **Automate baseline selection** (longest clean high-availability window).
4. **Data-driven, adaptive alert thresholds** per turbine.
5. **Seasonal baseline refresh** to counter slow drift.

The methods should be used to support one another, so the power curve analysis identifies performance anomalies, normal behaviour of systems/components can be used to identify faults, and predicting sensors can be used to check the calibration of the sensors taking the measurements.

The metrics produced for each turbine can be compared with the rest of the fleet to identify the characteristics of turbines with good/bad performance.

The aim would be to create fleet level analytics which assess turbines against each other as well as individual turbine analytics to identify potential faults and support maintenance activities and operational decisions.