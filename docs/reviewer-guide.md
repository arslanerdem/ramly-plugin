# Reviewer guide: Ramly connector / plugin

**What it does:** Ramly runs reliability, availability and maintainability (RAM)
studies. Through the MCP server an assistant builds reliability block diagrams and
runs Ramly's Monte Carlo or exact analytical (Markov) engine on the signed-in
user's Ramly account.

- **Server:** `https://mcp.ramly.io/mcp` (Streamable HTTP, OAuth 2.1 with PKCE S256, dynamic client registration and CIMD).
- **Test account:** provided in the submission form. It holds 9 models: 6 seeded examples plus 3 worked examples with completed results.

## Connect

1. Add a custom connector with the URL `https://mcp.ramly.io/mcp`.
2. On the Ramly consent page click **Continue**, sign in with the test account, then click **Allow**.

## Suggested prompts

1. "List my Ramly models." Calls `list_models`; 9 models are expected.
2. "Show the results of the HP trip SIF example. What SIL does it reach?" Calls `list_jobs` → `get_job`. Expect PFD_avg ≈ 1.2×10⁻², SIL 1, valve-dominated.
3. "Build a model of three pumps where two are needed (MTBF 12,000 h, repair 8 h) feeding one heat exchanger (MTBF 60,000 h, repair 48 h), and run it." Calls `create_model` → `run_simulation`, which uses one run credit and finishes in a few seconds.
4. "Would a second spare improve the cooling water example? Compare." Calls `get_model` → `create_model` (copy) → `run_simulation` → `compare_jobs`.
5. "What is the availability of two 1,000 h MTBF / 10 h MTTR units in parallel?" Calls `calculate_system_availability`: local, instant, no credit used.

## Tool safety

| Tool | readOnlyHint | destructiveHint | Notes |
|---|---|---|---|
| `get_account`, `list_models`, `get_model`, `get_job`, `list_jobs`, `compare_jobs` | true | false | Read the user's own data |
| `calculate_system_availability`, `component_availability`, `required_mtbf_for_target` | true | false | Local math, no network |
| `create_model` | false | false | Adds a model to the user's workspace |
| `update_model` | false | false | Replaces a model the user owns (earlier runs are kept) |
| `run_simulation` | false | false | Uses one monthly run credit on the user's plan |

No tool deletes data or acts outside the user's own Ramly account. Requests from a
browser Origin outside the allowlist are rejected.

## Data and privacy

- Access is scoped to the signed-in Ramly account. The grant holds a dedicated, revocable key; users disconnect under **Account → Connected AI apps** on ramly.io.
- Privacy policy: https://ramly.io/privacy · Terms: https://ramly.io/terms
- Support: https://github.com/arslanerdem/ramly-plugin/issues
