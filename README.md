# TwinShield: LLM-Supported Digital-Twin-Gated Zero-Touch Resilience for O-RAN Security

This repository contains the local prototype and reproducibility artifact for **TwinShield**, an LLM-supported digital-twin (DT)-gated zero-touch resilience framework for O-RAN security. The prototype evaluates bounded remediation decisions under coupled KPI poisoning and xApp misbehavior using a synthetic O-RAN-inspired environment, an analytical grey-box DT, heuristic controllers, and local LLM planners executed through Ollama.


---

## Repository purpose

The repository provides:

1. O-RAN-edge security simulator;
2. an analytical grey-box DT used for pre-actuation safety screening;
3. heuristic ungated and DT-gated controllers;
4. Ollama-backed LLM planners operating over a bounded remediation catalog;
5. JSON-schema validation for LLM action selection;
6. scripts for running experiments, saving logs, and generating figures/tables; and
7. The numerical parameter values used in the paper artifact.

The paper intentionally keeps the performance evaluation concise. This README records the implementation-specific numerical values so that readers can inspect, reproduce, or modify the artifact without overloading the manuscript.

---

## High-level architecture

TwinShield follows a bounded reasoning and safe actuation design:

1. **Telemetry layer:** produces service, resource, and security indicators.
2. **Risk estimation layer:** generates proxy scores for KPI poisoning and xApp misbehavior.
3. **Digital twin gate:** evaluates all candidate actions before execution.
4. **Planner:** selects one action using either a heuristic controller or a local LLM planner.
5. **Validation:** ensures that the selected LLM action is valid JSON and belongs to the supplied candidate set.
6. **Execution and logging:** applies the selected action in the synthetic environment and records state, action, DT scores, safety labels, and planner metadata.

In gated LLM mode, the LLM does **not** generate arbitrary network commands. It receives a DT-scored list of candidate actions and returns one structured choice from the approved catalog.

---

## Project structure

```text
codes/
|-- check_ollama.py
|-- plot_results.py
|-- requirements.txt
|-- run_experiment.py
|-- configs/
|   `-- example_synthetic.yaml
|-- data/
|   `-- trace_schema_example.csv
|-- results/
|-- runs/
|-- src/
|   `-- twinshield/
|       |-- __init__.py
|       |-- actions.py
|       |-- heuristics.py
|       |-- llm_planner.py
|       |-- metrics.py
|       |-- ollama_client.py
|       |-- state.py
|       |-- synthetic.py
|       |-- trace_io.py
|       `-- twin.py
```

## Main components

- `codes/run_experiment.py` runs the main experiments using either synthetic scenarios or CSV-based traces.
- `codes/plot_results.py` rebuilds plots from a saved run directory.
- `codes/configs/example_synthetic.yaml` provides an example experiment configuration.
- `codes/data/trace_schema_example.csv` provides an example input trace.
- `codes/src/twinshield/` contains the core source package.

## Code availability

The repository is being prepared for release. The full implementation, experiment scripts, and supporting materials will be uploaded after paper acceptance.


### Ollama setup for local LLM planning

The LLM planner is executed locally through Ollama. Install Ollama from the official distribution and pull the models used in the artifact.

Example commands:

```bash
ollama pull gemma:4b
ollama pull granite3.3
ollama pull llama3.2:1b
```

The exact local model tags may depend on the Ollama installation. If a model tag differs locally, update the model name in the experiment configuration file.

---

## Experimental configurations

The saved artifact uses eight controller/model configurations:

| Group | Configuration | Description |
|---|---|---|
| Heuristic | `heuristic_ungated` | Heuristic action selection without safe-only DT filtering. |
| Heuristic | `heuristic_gated` | Heuristic action selection with DT safety screening. |
| LLM | `llm_ungated + Gemma4` | Local Gemma planner without safe-only filtering. |
| LLM | `llm_gated + Gemma4` | Local Gemma planner with DT-gated candidate exposure. |
| LLM | `llm_ungated + Granite3.3` | Local Granite planner without safe-only filtering. |
| LLM | `llm_gated + Granite3.3` | Local Granite planner with DT-gated candidate exposure. |
| LLM | `llm_ungated + Llama3.2:1b` | Local Llama planner without safe-only filtering. |
| LLM | `llm_gated + Llama3.2:1b` | Local Llama planner with DT-gated candidate exposure. |

The evaluation size is:

| Parameter | Value |
|---|---:|
| Episodes per configuration | 80 |
| Decision epochs per episode | 16 |
| Total configurations | 8 |
| Total logged decisions | 10,240 |
| Attack conditions | KPI poisoning, xApp misbehavior, mixed attack |
| Action per decision step | One action from the fixed catalog |
| Compound actions | Not enabled in the current artifact |

---

## State representation

At each decision epoch `t`, the system state is represented as:

```text
s_t = [load, rho, xi, ue_risk, xapp_risk, throughput, latency, sla_score, stability]
```

where:

| Symbol / name | Meaning | Range |
|---|---|---|
| `load` / `ell_t` | Normalized network load | [0, 1] |
| `rho` / `rho_t` | Latent KPI poisoning severity | [0, 1] |
| `xi` / `xi_t` | Latent xApp compromise severity | [0, 1] |
| `ue_risk` / `r_UE` | Observable UE-risk proxy | [0, 1] |
| `xapp_risk` / `r_xApp` | Observable xApp-risk proxy | [0, 1] |
| `throughput` / `tau_t` | Normalized throughput | [0, 1] |
| `latency` / `lambda_t` | Normalized latency | [0, 1] |
| `sla_score` / `sigma_t` | SLA violation score | [0, 1] |
| `stability` / `kappa_t` | Control-loop stability | [0, 1] |

The latent variables `rho` and `xi` are internal simulator states. In a deployment, they would be estimated indirectly from telemetry and detector outputs.

---

## Threat scenarios and attack drift

The artifact considers three attack conditions. Each condition modifies the latent poisoning severity `rho` and xApp compromise severity `xi` through fixed drift terms.

| Attack condition | Drift in `rho` | Drift in `xi` | Interpretation |
|---|---:|---:|---|
| KPI poisoning | +0.06 | -0.03 | Poisoning pressure increases while xApp compromise pressure decays slightly. |
| xApp misbehavior | -0.02 | +0.07 | xApp compromise pressure increases while poisoning pressure decays slightly. |
| Mixed attack | +0.04 | +0.04 | Both latent threat components increase. |

The latent state update before bounding is:

```text
rho_next = rho + Delta_rho(attack_type) - Gamma_rho(action)
xi_next  = xi  + Delta_xi(attack_type)  - Gamma_xi(action)
```

Both values are then clipped to `[0, 1]`.

---

## Detector proxy generation

The current artifact uses synthetic monotone detector proxies rather than separately trained detectors.

The UE-risk proxy is:

```text
r_UE = clip(0.08 + 0.55 * rho + 0.08 * load + eps_UE, 0, 1)
```

The xApp-risk proxy is:

```text
r_xApp = clip(0.05 + 0.75 * xi + 0.05 * load + eps_x, 0, 1)
```

where `eps_UE` and `eps_x` are stochastic noise terms used by the simulator.

These proxies are intended to preserve the expected monotonic relationship between latent attack severity and observable detector evidence. They are not claimed to replace trained O-RAN security detectors.

---

## Remediation action catalog

TwinShield uses a fixed seven-action remediation catalog. The bounded action set prevents the planner from generating arbitrary or unsafe operations.

| Action | Cost `C(a)` | Reduction in `(rho, xi)` | Operational role |
|---|---:|---:|---|
| `noop` | 0.00 | (0.00, 0.00) | Hold state when intervention is not justified. |
| `reweight_ues` | 0.10 | (-0.32, 0.00) | Reduce suspicious UE influence during online adaptation. |
| `freeze_training` | 0.15 | (-0.24, 0.00) | Stop online model updates to prevent further poisoning. |
| `rollback_model` | 0.20 | (-0.44, -0.05) | Restore a trusted model or policy checkpoint. |
| `switch_shadow` | 0.25 | (0.00, -0.40) | Redirect control to a verified shadow policy. |
| `quarantine_xapp` | 0.40 | (-0.04, -0.64) | Isolate the suspected xApp from the control loop. |
| `migrate_xapp` | 0.35 | (0.00, -0.48) | Redeploy the xApp to a clean runtime environment. |

The `noop` action is always retained with zero cost and zero direct mitigation effect.

---

## 10. Security risk function

The post-action residual security risk is computed as:

```text
R = 0.55 * rho_hat + 0.65 * xi_hat
```

where:

| Weight | Value | Meaning |
|---|---:|---|
| `omega_rho` | 0.55 | Weight assigned to residual KPI poisoning severity. |
| `omega_xi` | 0.65 | Weight assigned to residual xApp compromise severity. |

The xApp-side weight is slightly higher because xApp compromise can directly affect closed-loop control actions.

---

## Digital twin transition model

We use an analytical grey-box DT rather than a learned twin. After applying attack drift and action mitigation, the DT predicts service outcomes using bounded surrogate equations.

### Residual risk

```text
R_hat = 0.55 * rho_hat + 0.65 * xi_hat
```

### Throughput prediction

```text
tau_hat = clip(0.89 - 0.35 * load - 0.46 * R_hat - 0.19 * C(action), 0, 1)
```

### Latency prediction

```text
lambda_hat = clip(0.15 + 0.54 * load + 0.55 * R_hat + 0.22 * C(action), 0, 1)
```

### SLA violation prediction

```text
sigma_hat = clip(0.10 + 0.56 * R_hat + 0.20 * load + 0.25 * C(action), 0, 1)
```

### Stability prediction

```text
kappa_hat = clip(0.92 - 0.35 * R_hat - 0.22 * C(action), 0, 1)
```

These equations encode the assumed relationship that higher load, higher residual security risk, and more aggressive actions can degrade throughput, increase latency/SLA violation, and reduce stability.

---

## Digital twin uncertainty model

The DT also assigns uncertainty margins to the predicted outcomes.

| Quantity | Formula |
|---|---|
| Throughput uncertainty | `U_tau = 0.03 + 0.04 * R_hat + 0.03 * C(action)` |
| SLA uncertainty | `U_sigma = 0.03 + 0.03 * R_hat + 0.02 * C(action)` |
| Stability uncertainty | `U_kappa = 0.03 + 0.02 * R_hat + 0.02 * C(action)` |

These uncertainty terms are used by the safety gate to apply conservative pre-actuation checks.

---

## Robust objective function

For each candidate action, the DT computes a one-step robust objective:

```text
J(action) = alpha * R_hat
          + beta  * sigma_hat
          + gamma * C(action)
          + delta * (1 - kappa_hat)
          + epsilon * U_total
```

The objective weights are:

| Weight | Value | Term |
|---|---:|---|
| `alpha` | 0.45 | Residual security risk |
| `beta` | 0.25 | SLA violation score |
| `gamma` | 0.15 | Actuation cost |
| `delta` | 0.10 | Instability penalty |
| `epsilon` | 0.05 | DT uncertainty penalty |

`U_total` can be implemented as a scalar aggregation of the DT uncertainty terms, such as the mean or weighted sum of `U_tau`, `U_sigma`, and `U_kappa`, depending on the code implementation.

---

## DT safety constraints

An action is DT-approved only if it satisfies all conservative safety constraints:

```text
tau_hat   - 1.96 * U_tau   >= 0.45
sigma_hat + 1.96 * U_sigma <= 0.40
kappa_hat - 1.96 * U_kappa >= 0.60
```

| Constraint | Threshold | Meaning |
|---|---:|---|
| Minimum conservative throughput | 0.45 | Avoid actions that are predicted to reduce throughput below the acceptable level. |
| Maximum conservative SLA violation | 0.40 | Avoid actions that are predicted to violate SLA tolerance. |
| Minimum conservative stability | 0.60 | Avoid actions that are predicted to destabilize the control loop. |
| Uncertainty multiplier | 1.96 | Conservative margin applied to DT uncertainty estimates. |

The gate therefore uses the DT as a safety authority, not only as a predictor.

---

## Candidate construction for LLM planning

The planner receives a bounded candidate list rather than the full uncontrolled action space.

###  Gated LLM mode

1. The DT evaluates all seven actions.
2. Actions satisfying all safety constraints are marked as approved.
3. The planner receives up to the top four DT-approved actions ranked by objective value.
4. If no safe action exists, the planner receives the top four ranked actions as a graceful fallback.
5. The selected action must belong to the provided candidate list.

### Ungated LLM mode

1. The DT still ranks actions for context.
2. The candidate list is not filtered to safe-only actions.
3. The LLM receives the top four ranked actions without conservative safety filtering.

### Heuristic modes

The heuristic controllers use the same bounded catalog and detector scores. The gated heuristic applies the DT safety screen, while the ungated heuristic does not. This makes the heuristic pair useful for isolating the effect of the DT gate independently of LLM behavior.

---

## LLM input and output contract

The LLM prompt includes:

1. the current telemetry snapshot;
2. UE-risk and xApp-risk scores;
3. attack context;
4. gate mode;
5. DT-scored candidate actions;
6. predicted throughput, SLA, stability, uncertainty, and objective score for each candidate; and
7. explicit instructions to choose exactly one action from the candidate list.

The LLM output is expected to follow a JSON schema similar to:

```json
{
  "selected_action": "rollback_model",
  "suspected_primary_cause": "kpi_poisoning",
  "confidence": 0.82,
  "rationale": "The UE-risk score is high and rollback_model has the best DT-approved safety score."
}
```

Recommended schema rules:

| Field | Type | Constraint |
|---|---|---|
| `selected_action` | string | Must be one of the provided candidate actions. |
| `suspected_primary_cause` | string | One of `kpi_poisoning`, `xapp_misbehavior`, `mixed`, or `uncertain`. |
| `confidence` | number | Should be in `[0, 1]`. |
| `rationale` | string | Short explanation of the decision. |

If parsing fails, the JSON schema is violated, or the model selects an out-of-set action, the controller falls back to the first candidate in the DT-ranked candidate list.

---

## Metrics

The artifact reports aggregate and attack-specific metrics:

| Metric | Meaning |
|---|---|
| Unsafe-action rate | Fraction of selected actions violating the DT safety criteria or unsafe labeling rule. |
| SLA violation score | Normalized SLA degradation score. Lower is better. |
| Normalized throughput | Predicted or realized normalized throughput. Higher is better. |
| Stability | Control-loop stability score. Higher is better. |
| Recovery steps | Number of steps needed to return below a risk or service-degradation threshold. |
| LLM latency | Time required for local LLM action selection. |
| JSON/schema validity | Fraction of LLM responses that satisfy the required JSON schema. |
| DT-approved actions | Number or fraction of catalog actions admitted by the DT gate. |
| DT-blocked actions | Number or fraction of catalog actions rejected by the DT gate. |
| Top-1 adherence | Fraction of times the LLM selects the top DT-ranked candidate. |

Confidence intervals are computed using 95% bootstrap intervals over episode-level metrics.

-
## License

For academic reproducibility, a permissive license such as MIT, BSD-3-Clause, or Apache-2.0 can be used if compatible with all dependencies and institutional requirements.

---

## Contact

For questions about the artifact, please open a GitHub issue or contact the corresponding author listed in the paper.
