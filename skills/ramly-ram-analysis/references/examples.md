# Worked examples

All failure data below are **indicative assumptions for illustration**. Replace them with
the user's own data.

## 1. Cooling water pump station (Monte Carlo)

**The question:**
- Three pumps, two needed (2-of-3), feeding one heat exchanger.
- One mechanical crew covers the site.
- One spare pump cartridge is held, with a 2-week re-order lead time.
- What's the availability, and is a second spare worth it?

```json
{
  "name": "Cooling water: baseline",
  "simulation": { "num_simulations": 1000, "duration_hours": 87600, "time_window_hours": 8760, "rng_seed": 42 },
  "importance_measures": true,
  "downtime_cost_per_hour": 4000,
  "component_types": [
    { "name": "Centrifugal pump", "failure": "WEIBULL 14000 1.5", "repair": "LOGNORMAL 2.1 0.6",
      "logistics": "TRIANGULAR 2 4 12", "spare_pool": "PUMP_CARTRIDGES", "replacement": "CONSTANT 6",
      "repair_cost": 6000, "data_reference": "Indicative assumption; replace with site CMMS history" },
    { "name": "Plate heat exchanger", "failure": "EXPONENTIAL 70000", "repair": "LOGNORMAL 3.4 0.5",
      "repair_cost": 15000, "data_reference": "Indicative assumption" }
  ],
  "spare_pools": [{ "name": "PUMP_CARTRIDGES", "initial_quantity": 1, "replenishment": "NORMAL 336 48" }],
  "crew_groups": [{ "name": "MECH", "num_crews": 1, "assigned_blocks": ["SYSTEM"] }],
  "system": { "type": "SERIES", "children": [
    { "name": "PUMPS", "type": "PARALLEL", "k": 2, "children": [
      { "name": "PUMP_A", "component_type": "Centrifugal pump" },
      { "name": "PUMP_B", "component_type": "Centrifugal pump" },
      { "name": "PUMP_C", "component_type": "Centrifugal pump" } ] },
    { "name": "HX_1", "component_type": "Plate heat exchanger" } ] }
}
```

**Then:**
1. Run the baseline.
2. `get_model`, set `initial_quantity: 2`, and save as a **new** model ("Cooling water: 2 spares") with `create_model`.
3. Run it and `compare_jobs` the two runs.
4. Report the downtime-hours difference and the cost difference against the cost of holding a spare.

**Expected outcome** (1,000 runs, seed 42; numbers move slightly with the seed):

| | Baseline (1 spare) | 2 spares |
|---|---|---|
| Availability | ≈ 0.9995, about 4.4 h downtime/yr | ≈ 0.99955, about 3.9 h/yr |
| Spare service level | ≈ 0.95 | ≈ 0.999 |
| Final-year cost | ≈ $32k | ≈ $30k |

Weigh the about $2k/yr saving against the cost of holding the extra cartridge.

## 2. Gas compression train with standby (analytical)

**The question:** Two compressors on duty/standby (cold), plus an after-cooler and a
separator in series. What are the steady-state availability and the one-year mission
reliability?

```json
{
  "name": "Compression train: duty/standby",
  "solver": "analytical",
  "simulation": { "duration_hours": 8760, "time_window_hours": 8760, "reliability_mission_time_hours": 8760 },
  "component_types": [
    { "name": "Centrifugal compressor", "failure": "EXPONENTIAL 9000", "repair": "EXPONENTIAL 72" },
    { "name": "After-cooler", "failure": "EXPONENTIAL 60000", "repair": "EXPONENTIAL 24" },
    { "name": "Separator", "failure": "EXPONENTIAL 150000", "repair": "EXPONENTIAL 36" }
  ],
  "system": { "type": "SERIES", "children": [
    { "name": "COMPRESSORS", "type": "STANDBY", "k": 1, "children": [
      { "name": "K_101A", "component_type": "Centrifugal compressor" },
      { "name": "K_101B", "component_type": "Centrifugal compressor" } ] },
    { "name": "E_101", "component_type": "After-cooler" },
    { "name": "V_101", "component_type": "Separator" } ] }
}
```

**Expected outcome:** availability ≈ 0.9993, one-year mission reliability ≈ 0.61, MTTF ≈ 13,600 h. The analytical solver is instant and exact for this structure. Report availability,
`reliability.mission_reliability` and `reliability.mttf_hours`. To add a shared repair
crew or spares, switch to Monte Carlo, because the analytical solver doesn't support them.

## 3. Safety-instrumented function, 1oo2 transmitters (analytical PFD)

**The question:** A high-pressure trip with two transmitters voting 1oo2, a logic
solver and one shutdown valve. Proof tests run yearly. Which SIL does PFD_avg support,
and what does a 6-monthly test give?

```json
{
  "name": "HP trip SIF: annual proof test",
  "solver": "analytical",
  "simulation": { "duration_hours": 8760, "time_window_hours": 8760 },
  "component_types": [
    { "name": "Pressure transmitter (DU)", "failure": "EXPONENTIAL 1000000", "repair": "CONSTANT 8",
      "detection": "hidden", "inspection_interval_hours": 8760 },
    { "name": "Logic solver (DU)", "failure": "EXPONENTIAL 10000000", "repair": "CONSTANT 8",
      "detection": "hidden", "inspection_interval_hours": 8760 },
    { "name": "Shutdown valve (DU)", "failure": "EXPONENTIAL 400000", "repair": "CONSTANT 24",
      "detection": "hidden", "inspection_interval_hours": 8760 }
  ],
  "ccf_groups": [{ "name": "PT_CCF", "components": ["PT_A", "PT_B"],
                   "failure": "EXPONENTIAL 20000000", "repair": "CONSTANT 8" }],
  "system": { "type": "SERIES", "children": [
    { "name": "SENSORS", "type": "PARALLEL", "k": 1, "children": [
      { "name": "PT_A", "component_type": "Pressure transmitter (DU)" },
      { "name": "PT_B", "component_type": "Pressure transmitter (DU)" } ] },
    { "name": "LS_1", "component_type": "Logic solver (DU)" },
    { "name": "XV_1", "component_type": "Shutdown valve (DU)" } ] }
}
```

**Notes:**
- "DU" means dangerous undetected failure rate.
- The CCF mean is 1/(β·λ_DU) with β = 5 %.
- Read `pfd.pfd_avg` and map it to the SIL band.
- Then copy the model with `inspection_interval_hours: 4380` on each type and compare.
- **Expected outcome:**
  - With annual tests, PFD_avg ≈ 1.2×10⁻², i.e. **SIL 1**. The valve dominates: λ_DU·τ/2 ≈ 2.5×10⁻⁶ × 8760 / 2 ≈ 0.011.
  - With 6-monthly tests, PFD_avg ≈ 5.8×10⁻³, i.e. **SIL 2**.
  - Say that the valve is the driver, and note that architectural constraints (HFT) are a separate check.
