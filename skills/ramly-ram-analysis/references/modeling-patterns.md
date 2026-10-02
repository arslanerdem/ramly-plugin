# Modeling patterns (Ramly model format)

A model has `component_types` (failure/repair behaviour) and a `system` block tree.
Every leaf is a `COMPONENT` block pointing at a component type. Each physical item
gets its own block name, so three identical pumps are three blocks, `PUMP_A`,
`PUMP_B` and `PUMP_C`, sharing one type.

## Structure

| Real-world arrangement | Block |
|---|---|
| All items needed (a process train, power supply chain) | `{"type": "SERIES", "children": [...]}` |
| 1 of 2 running, both hot (duplex) | `{"type": "PARALLEL", "k": 1, "children": [A, B]}` |
| 2 of 3 needed (N+1) | `{"type": "PARALLEL", "k": 2, "children": [A, B, C]}` |
| Duty/standby with switch-over | `{"type": "STANDBY", "k": 1, "children": [DUTY, STANDBY]}`. The first k run; the others start when one fails. |
| Nested (trains of equipment, redundant trains) | Blocks inside blocks, e.g. `SYSTEM = SERIES[ FEED, PARALLEL k=2 [TRAIN_1, TRAIN_2, TRAIN_3], EXPORT ]`, each train a `SERIES` |

`k` counts the children that must be **up** for the block to be up. In the Ramly
format it defaults to 1, so always set it explicitly.

## Standby

- **Cold standby** (doesn't age while idle): only `failure` / `repair` on the type.
- **Warm standby** (can fail while idle): add `standby_failure`, e.g. `"EXPONENTIAL 200000"`.
- Standby is solved exactly by the analytical solver (Markov) and simulated in Monte Carlo.

## Common-cause failures

Use `ccf_groups` when redundant items can fail together (shared utility, design defect,
environment):

```json
"ccf_groups": [{ "name": "PUMP_CCF", "components": ["PUMP_A", "PUMP_B", "PUMP_C"],
                 "failure": "EXPONENTIAL 400000", "repair": "LOGNORMAL 3.2 0.5" }]
```

**Beta-factor conversion:** with an independent failure rate λ and beta β (typically
2–10%), λ_ccf ≈ β·λ, so mean = 1/(β·λ). CCF often dominates highly redundant
designs, so leaving it out makes redundancy look better than it is.

## Repair resources

**Logistics delay** (mobilisation, permits, access): `"logistics": "TRIANGULAR 4 12 48"`
on the type.

**Shared repair crews** limit how many repairs run at once. Monte Carlo only:
```json
"crew_groups": [{ "name": "MECH", "num_crews": 1, "assigned_blocks": ["SYSTEM"] }]
```
Assigning a block covers everything under it. The nearest assignment wins.

**Spare parts.** A failed item is swapped from stock (fast) and the stock is
replenished (slow). Monte Carlo only:
```json
"spare_pools": [{ "name": "PUMP_SPARES", "initial_quantity": 1, "replenishment": "NORMAL 336 48" }],
"component_types": [{ "name": "Pump", "failure": "...", "repair": "...",
                      "spare_pool": "PUMP_SPARES", "replacement": "CONSTANT 6" }]
```
When stock is empty, the repair waits for replenishment, which shows as a lower
`service_level`.

## Preventive maintenance

- **Per component type** (each item serviced on its own clock): `pm_interval_hours` + `pm_duration`.
  The defaults are `pm_resets_age: true` (as good as new) and `repair_resets_age: true`.
  Set `repair_resets_age: false` for minimal repair (as bad as old) with Weibull wear-out.
- **Planned shutdown of a block** (turnaround or outage):
  ```json
  "pm_schedules": [{ "name": "Annual turnaround", "target_block": "SYSTEM",
                     "interval_hours": 8760, "duration": "CONSTANT 120", "cost": 250000 }]
  ```
- PM only pays off for **wear-out** (Weibull β > 1). For exponential failures it only adds downtime.

## Hidden failures and proof testing (safety functions)

Items that fail silently (ESD valves, trip transmitters, relief devices):
```json
{ "name": "ESD valve", "failure": "EXPONENTIAL 200000", "repair": "CONSTANT 24",
  "detection": "hidden", "inspection_interval_hours": 8760, "inspection_duration": "CONSTANT 4" }
```
- A failure stays undetected until the next proof test. The analytical solver reports `pfd` (PFD_avg and peak over the test interval).
- Voting: 1oo2 = `PARALLEL k=1`, 2oo3 = `PARALLEL k=2`.
- A subsystem-wide test can be set with `inspection_plans` targeting a block.

## Buffers, throughput and products (Monte Carlo)

- **Storage** keeps a block "producing" while it's down until the buffer runs dry:
  `"storage": [{ "capacity": 12, "fill_rate": 0.5, "assigned_to": "COMPRESSION" }]`.
  It drains 1 unit/h, so capacity is hours of cover, and it refills at `fill_rate` per hour while up.
- **Throughput:** `throughput_capacity` on blocks (e.g. 3 × 50 t/h trains behind a 100 t/h
  plant) and top-level `"throughput": {"capacity": 100, "unit": "t/h"}`. Losing one train
  then only costs output above the plant capacity.
- **Products** with demand and recipes: `products[]` with `produced_by`, `capacity`, `demand`, `inputs`.

## Costs

`downtime_cost_per_hour` at model level, plus `repair_cost` / `pm_cost` on types
(and `cost` on PM schedules). Results include a cost total over the study.

## Multi-state equipment (analytical only)

`state_model` on a type gives partial-capacity states with exponential transitions:
```json
"state_model": { "states": [ {"name": "full", "capacity": 1, "available": true},
                             {"name": "derated", "capacity": 0.5, "available": true},
                             {"name": "down", "capacity": 0, "available": false} ],
                 "transitions": [ {"from": "full", "to": "derated", "distribution": "EXPONENTIAL 4000"},
                                  {"from": "derated", "to": "down", "distribution": "EXPONENTIAL 2000"},
                                  {"from": "derated", "to": "full", "distribution": "EXPONENTIAL 24"},
                                  {"from": "down", "to": "full", "distribution": "EXPONENTIAL 72"} ] }
```
Monte Carlo treats such types as binary up/down and warns.

## Outputs to request

- `importance_measures: true` gives a Birnbaum/RAW/RRW ranking per component. It costs extra compute on big models; use it for "what drives downtime?".
- `sensitivity_analysis: true` gives MTTF/MTTR ±20 % per type. Use it for "where should we invest?".
- `tracked_blocks` limits which blocks get KPIs. By default it's SYSTEM plus every non-component block.

## Simulation settings

```json
"simulation": { "num_simulations": 1000, "duration_hours": 87600, "time_window_hours": 8760, "rng_seed": 42 }
```
- The horizon is at most 10 years.
- `time_window_hours` must divide `duration_hours`. Results are reported per window, and the headline is the final window.
- Keep the seed fixed when comparing options, so differences come from the design rather than random noise.
