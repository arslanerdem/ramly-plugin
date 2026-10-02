# Interpreting Ramly results

`run_simulation` / `get_job` return a `summary`. **Headline KPIs are for the final time
window**, matching the ramly.io results page. `availability_by_window` shows the trend
(useful for ageing equipment).

## System

| Field | Meaning | How to say it |
|---|---|---|
| `availability` | Mean fraction of time the system is up | "99.42 % available" |
| `downtime_hours_per_window` | (1 − A) × window length | "≈ 51 h of downtime per year" (for 8,760 h windows) |
| `availability_p10` / `p50` / `p90` | Percentiles across simulated histories | "In 1 year out of 10, availability falls below P10" |
| `failures_per_window`, `mtbf_hours`, `mttr_hours` | System-level outage count and durations | Outages that actually stop the system, not component failures |
| `reliability_at_window_end` | Analytical: probability of no system failure up to then | Mission reliability |

A wide P10–P90 spread means rare, long outages (CCF, slow spares, long repairs) are
driving risk. Averages hide this, so mention it whenever P10 is materially below the
mean.

## Where downtime comes from

- `blocks_worst_first` lists sub-systems sorted by availability (SYSTEM excluded). `caused_system_downtime_hours` is the system downtime attributed to that block: the most direct "bad actor" list.
- `component_types` gives `failures_per_year` and `mean_down_time_hours` per type. For hidden failures it also gives `pfd_avg` and `mean_time_undetected_hours`.
- `importance_top` (when importance was requested):
  - **Birnbaum** = A(component perfect) − A(component failed): how much the system depends on it.
  - **RAW** (risk achievement worth) = how many times worse unavailability gets if it's failed. High RAW means it's critical to keep working.
  - **RRW** (risk reduction worth) = how many times better unavailability gets if it never failed. High RRW means improving it pays.
- `sensitivity_top`: system availability when each type's MTTF or MTTR moves ±20 %. The biggest swings show where reliability or maintainability investment helps most.

## Support resources

- `spare_pools[].service_level`: the fraction of demands met from stock. Below about 0.95 usually means stock-outs are adding downtime; try `initial_quantity + 1` or a shorter `replenishment`.
- `crew_groups[].avg_queue_wait_hours`: time repairs waited for a free crew. Material waiting means crew capacity is a constraint.
- `cost_total_over_study`: repair + PM + downtime cost over the whole study. Compare options on total cost, not only availability.

## Accuracy

- **Monte Carlo** `convergence`:
  - `ci_95_half_width` is the ± uncertainty on the mean availability.
  - If `converged` is false, or the difference between two options is smaller than about 2× the half-width, **re-run with `recommended_simulations`** before drawing conclusions.
  - Keep the same `rng_seed` across compared options.
- **Analytical** results are exact for the model as specified (`convergence.exact: true`), with no sampling noise. They don't cover crews, spares, storage or PM.
- **`warning`** on multi-state types: Monte Carlo simplified them to up/down. Use the analytical solver for partial-capacity results.

## Analytical extras

- `reliability`: `mission_reliability` (probability of no system failure over the mission) and `mttf_hours` (mean time to first system failure, without repair).
- `pfd`: average and peak probability of failure on demand over the proof-test interval. Compare PFD_avg with the SIL bands:

| SIL | PFD_avg |
|---|---|
| SIL 1 | 10⁻² to 10⁻¹ |
| SIL 2 | 10⁻³ to 10⁻² |
| SIL 3 | 10⁻⁴ to 10⁻³ |

  These are low-demand mode values. Architectural constraints and systematic capability are separate requirements.

## A good summary

1. **Answer first:** "The design meets the 98 % target: 98.6 % expected, 97.9 % in a bad year (P10)."
2. **Drivers:** the top two or three blocks or types and why (failure frequency vs long repairs).
3. **Recommendation** with the quantified effect, from a compared run where possible.
4. **Assumptions and data sources**, and what would change the conclusion.
5. **Link** to the job's `web_url` for the full results and the PDF report.
