---
name: ramly-ram-analysis
description: Runs reliability, availability and maintainability (RAM) studies with Ramly. It builds reliability block diagrams (series, k-out-of-n, standby), chooses failure and repair distributions, runs discrete-event Monte Carlo or exact analytical/Markov studies, and explains availability, downtime, bad actors, spares, crews and costs. Use when the user asks about system availability, uptime, reliability, MTBF/MTTR, redundancy (2-of-3, N+1, standby), RBDs, spare parts, maintenance strategy, SIL/PFD of safety functions, or wants to compare design options for a plant, process, fleet or facility.
license: MIT
compatibility: Needs the Ramly MCP server (https://mcp.ramly.io/mcp) connected with the user's Ramly account.
metadata:
  author: Ramly LLC
  version: "1.0.0"
  homepage: https://ramly.io
---

# RAM studies with Ramly

Ramly runs the same engine a reliability engineer uses on ramly.io: a discrete-event
**Monte Carlo** simulator and an exact **analytical / Markov (CTMC)** solver. Your job is
to turn the user's description into a sound model, run it, and explain what the
numbers mean for their decision. Engineering judgement matters more than speed.

## Tools

| Tool | Use |
|---|---|
| `get_account` | Plan, run credits used/left, simulation cap per run. Check before large studies. |
| `list_models`, `get_model` | Existing models (including examples) in Ramly's model format. |
| `create_model`, `update_model` | Build or change a model. `update_model` is a full replace: start from `get_model`. |
| `run_simulation` | Runs a study. **Uses one monthly run credit.** Returns results, or a job id if still running. |
| `get_job`, `list_jobs`, `compare_jobs` | Results, history, and side-by-side comparison of 2–5 runs. |
| `calculate_system_availability`, `component_availability`, `required_mtbf_for_target` | Instant closed-form estimates (exponential, independent components). No credit used. |

## Workflow

1. **Frame the question.** Before modelling, establish:
   - the **system boundary** and what "available" means: any output, full output, or meeting demand;
   - the **study horizon**, plus any mission time for reliability;
   - **redundancy**: how many units must run, hot vs standby;
   - the **maintenance setup**: repair crews, spares and lead times, planned shutdowns, proof tests;
   - the **decision** the study supports: meet a target, pick a design, size spares.

   Ask only what you can't reasonably assume. If the user wants speed, proceed with clearly stated assumptions.

2. **Sanity-estimate first (optional, free).** `calculate_system_availability` with MTBF/MTTR gives an order-of-magnitude answer and catches data errors before spending credits.

3. **Build the model** with `create_model`. See [modeling patterns](references/modeling-patterns.md) and [distributions](references/distributions.md).
   - The root block is `SYSTEM`. Block names are unique, with **no spaces**. Component type names may have spaces.
   - **Structure:** `SERIES` means all children are needed. `PARALLEL` with `k` means k-of-n hot redundancy; `k` defaults to **1**, so set it explicitly. `STANDBY` with `k` means k running, the rest cold or warm standby.
   - **Distributions** are strings in hours, e.g. `"WEIBULL 12000 1.4"`, `"EXPONENTIAL 50000"`, `"LOGNORMAL 2.5 0.6"`.
   - Put the source of each failure rate in `data_reference`.
   - Fix everything listed in `run_blockers` in the response before running.

4. **Pick the solver.**
   - `analytical` is exact and instant. It handles independent series, k-of-n, standby, common cause, multi-state and hidden failures with proof tests. It **does not** handle repair crews, spare pools, storage buffers or preventive maintenance.
   - `monte_carlo` (the default) handles everything, including Weibull ageing, spares, crews, PM, buffers, throughput and costs.
   - For Monte Carlo, start at **1,000 simulations** for exploration. Use the `recommended_simulations` from the convergence result for final numbers, within the plan cap shown by `get_account`.

5. **Confirm before spending credits.** Tell the user how many runs you plan, e.g. "baseline plus 2 options = 3 run credits", unless they already asked you to go ahead.

6. **Run** with `run_simulation`. If it returns a job id with status `queued` or `running`, call `get_job` a little later. Don't start duplicate runs.

7. **Interpret.** See [interpreting results](references/interpreting-results.md). Lead with the answer to the user's question, then cover:
   - system availability and annual downtime;
   - the spread (P10/P90) and what drives it;
   - the worst blocks (`blocks_worst_first`, `caused_system_downtime_hours`) and importance rankings;
   - spare service levels, crew waiting time, and costs when modelled;
   - whether the result converged.

8. **Improve and compare.** Change one thing at a time:
   - redundancy, MTTR (logistics, spares, crews), PM interval, proof-test interval;
   - re-run, then use `compare_jobs` against the baseline;
   - state the change in availability, downtime hours and cost.

9. **Hand off.** Give the `web_url` of the model or job. The user can open the full results, charts and a PDF/Word report on ramly.io. Summarise assumptions, data sources and limitations.

## Guardrails

- **Data provenance.**
  - Never present numbers as coming from OREDA, IEEE 493, NPRD or a vendor unless the user supplied them from that source.
  - If you assume values, label them "indicative assumptions" in `data_reference` and in your answer, and suggest the user replace them with their own CMMS history or licensed data.
- **Units are hours.** `EXPONENTIAL <mean>` takes the **MTTF** (mean time to failure), not a failure rate. Convert rates with mean = 1/λ, e.g. λ = 2×10⁻⁵ /h → `EXPONENTIAL 50000`.
- **Weibull scale ≠ MTTF.** MTTF = η·Γ(1+1/β). See [distributions](references/distributions.md).
- **Lognormal parameters are in log space** (mean and standard deviation of ln t), not hours.
- **Repair is hands-on time.** Put waiting for people, parts or access in `logistics`, or model spares and crews explicitly. Mean down time = logistics + repair.
- **Don't overstate precision.** Report availability to about 3–4 significant figures. A 0.01-point difference inside the confidence interval is noise.
- **Respect credits.**
  - One `run_simulation` uses one monthly run credit.
  - Don't loop runs automatically.
  - If a plan limit is reached, say so plainly and stop.
- **Models are the user's.** They appear in their ramly.io workspace. Use clear names and don't overwrite a model the user didn't ask you to change: create a copy for "what-if" options.

## Worked examples

[examples](references/examples.md) contains three complete studies:
- a 2-of-3 pump station with a shared crew and spares;
- an analytical gas compression train with a standby compressor;
- a 1oo2 safety-instrumented function with proof testing (PFD).
