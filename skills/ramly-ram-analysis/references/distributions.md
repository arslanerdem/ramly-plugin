# Choosing failure and repair distributions

All times are in **hours**. Strings are `"<TYPE> <params…>"`.

| Type | Parameters | Typical use |
|---|---|---|
| `EXPONENTIAL` | mean (MTTF or MTTR) | Random failures (electronics, many process items in useful life); first-cut models |
| `WEIBULL` | scale η, shape β | Ageing/wear-out (β > 1), infant mortality (β < 1) |
| `LOGNORMAL` | mean and sd of **ln t** | Repair and restoration times (right-skewed) |
| `NORMAL` | mean, sd | Well-controlled task durations (planned replacement, PM) |
| `TRIANGULAR` | min, mode, max | Expert estimates of delays (logistics, mobilisation) |
| `UNIFORM` | min, max | Bounded unknowns |
| `CONSTANT` | value | Fixed durations (inspection, swap-in time) |

## From the data you usually have

**A failure rate λ (per hour) or an MTBF/MTTF:** `EXPONENTIAL <1/λ>`.
- 3.5 failures per 10⁶ h → mean = 10⁶/3.5 ≈ 285,714 → `EXPONENTIAL 285714`.
- 2 failures per year → mean = 8760/2 = 4380 → `EXPONENTIAL 4380`.

For repairable items in steady state, MTBF ≈ MTTF + MTTR. When MTTR ≪ MTTF,
use the MTBF as the mean.

**Weibull from MTTF and a known shape β:** η = MTTF / Γ(1 + 1/β).

| β | Γ(1+1/β) | η for MTTF 10,000 h |
|---|---|---|
| 0.8 | 1.133 | 8,826 |
| 1.0 | 1.000 | 10,000 |
| 1.5 | 0.903 | 11,077 |
| 2.0 | 0.886 | 11,284 |
| 3.0 | 0.893 | 11,198 |

**Shape guidance (only when the user has no fitted data, and label it as an assumption):**
- Pumps and compressors (mechanical wear): β ≈ 1.2–2.
- Bearings and seals: 1.5–3.
- Electronics: ≈ 1.

β > 1 is what makes preventive maintenance worthwhile.

**Lognormal repair from a median and error factor:** EF = 95th / 50th percentile.
- μ = ln(median), σ = ln(EF)/1.645.
- Median 8 h, EF 3 → μ = 2.079, σ = 0.668 → `LOGNORMAL 2.079 0.668`.
- Its mean = exp(μ + σ²/2), here ≈ 9.9 h.

**Lognormal from a mean m and standard deviation s (hours):**
- σ² = ln(1 + s²/m²), μ = ln(m) − σ²/2.
- Mean 24 h, sd 12 h → σ = 0.472, μ = 3.067 → `LOGNORMAL 3.067 0.472`.

**Expert range for a delay:** `TRIANGULAR <best> <likely> <worst>`, e.g. `TRIANGULAR 4 12 48`.

## Separating the parts of downtime

| Phase | Field |
|---|---|
| Waiting for crew, permits, access, transport | `logistics` on the type |
| Waiting for a crew that's busy elsewhere | `crew_groups` (modelled, don't add it by hand) |
| Waiting for a part | `spare_pool` + `replacement` + pool `replenishment` |
| Hands-on repair | `repair` |
| Restart or ramp-up of a block | `restart` on the block |

Double-counting is the most common error: if the "MTTR" you were given already
includes waiting for parts, don't also model spares.

## Sanity checks before running

- Component availability ≈ MTTF/(MTTF+MTDT) should look plausible (e.g. 0.95–0.9999).
- Very large MTTFs (> 10⁷ h) or tiny repair times (< 0.1 h) are usually unit errors (minutes vs hours, rate vs mean).
- Use `component_availability` / `calculate_system_availability` for a free cross-check.
