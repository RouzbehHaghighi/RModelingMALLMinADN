# MA_LLM_Reliability_Visualization_Active_Distribution_Networks

This repository provides supplementary materials and reproducible resources for the paper:

**"Bayesian-Network-Informed Multi-Agent LLM Clustering for Reliability-Oriented Visualization of DER-Rich Active Distribution Networks"**

**Authors:**  
**Rouzbeh Haghighi**, Graduate Student Member IEEE, **Van-Hai Bui**, Senior Member IEEE, **Mengqi Wang**, Senior Member IEEE, **Wencong Su**, Senior Member IEEE

**Note:** This paper has been submitted to **Elsevier - Applied Energy**.

**Paper link:** (to be added)

## Description

This repository includes mathematical formulations, tables, and figures discussed in the paper. It aims to provide additional resources and data to support the research findings and facilitate further studies.

## Contents

### I. DER Generation Models

This study adopts standard steady-state WT and PV generation models to represent time-varying active-power injections from inverter-interfaced DERs. Environmental inputs are mapped to active-power outputs that serve as exogenous injections in the power-flow model and the subsequent VVC optimization.

$$
P_{\mathrm{WT}}(v)=
\begin{cases}
0, & v < v_{\mathrm{ci}} \text{ or } v \ge v_{\mathrm{co}}, \\
C_1 v^3 - C_2 P_{\mathrm{WT}}^{\mathrm{r}}, & v_{\mathrm{ci}} \le v \le v_{\mathrm{r}}, \\
P_{\mathrm{WT}}^{\mathrm{r}}, & v_{\mathrm{r}} \le v \le v_{\mathrm{co}}
\end{cases}
$$

$$
C_1=\frac{P_{\mathrm{WT}}^{\mathrm{r}}}{v_{\mathrm{r}}^{3}-v_{\mathrm{ci}}^{3}},
\qquad
C_2=\frac{v_{\mathrm{ci}}^{3}}{v_{\mathrm{r}}^{3}-v_{\mathrm{ci}}^{3}}
$$

where $P_{\mathrm{WT}}^{\mathrm{r}}$ is the rated WT active power (kW); $v$ is the wind speed (m/s); and $v_{\mathrm{ci}}$, $v_{\mathrm{r}}$, and $v_{\mathrm{co}}$ denote the cut-in, rated, and cut-out wind speeds, respectively.

For PV units, the cell temperature is approximated using the NOCT model, and the PV active power $P_{\mathrm{PV}}$ is computed using an irradiance- and temperature-corrected linear model:

$$
T_{\mathrm{c}} = T_{\mathrm{a}} + \frac{G_{\mathrm{irr}}(\mathrm{NOCT}-20)}{800}
$$

$$
P_{\mathrm{PV}}(G_{\mathrm{irr}},T_{\mathrm{c}})=
P_{\mathrm{PV}}^{\mathrm{r}}
\left(\frac{G_{\mathrm{irr}}}{G_{\mathrm{ref}}}\right)
\left[1+\eta_T(T_{\mathrm{c}}-T_{\mathrm{ref}})\right]
$$

where $P_{\mathrm{PV}}^{\mathrm{r}}$ is the rated PV active power (kW); $G_{\mathrm{irr}}$ is the solar irradiance (W/m²); $T_{\mathrm{a}}$ is the ambient temperature (°C); $\eta_T$ is the PV temperature coefficient (1/°C); $G_{\mathrm{ref}}$ is the reference irradiance; and $T_{\mathrm{ref}}$ is the reference cell temperature.


<img width="1000" height="250" alt="Git_Converter_Topology" src="https://github.com/user-attachments/assets/b2a3b9ac-4461-42a2-998e-503ffed38c4e" />


### II. Prompts templates

All LLM calls in this study use a two-message chat format:

- **System prompt** — fixed role, rules, and JSON output schema. It does not contain case-specific metrics.
- **User prompt** — filled at runtime with structured evidence (bus signals, matrices, partition metrics, or operator context). Agents must reason only from this evidence.

---

#### Phase 2 — MA-LLM reliability clustering (agents R, B, S, A)

Four specialist agents propose and refine a $K$-cluster partition of the IEEE 69-bus feeder. They share one **system** message; each agent receives a role-specific **user** prompt (initial proposal, then critic / refinement rounds).

**System prompt (shared by agents R, B, S, A)**

<pre style="background:#f0f0f0;padding:14px;border-radius:6px;overflow:auto;white-space:pre-wrap">
You are a power distribution system reliability-partitioning agent.

Your task is to partition only physical buses of the IEEE 69-bus feeder into K={num_partitions} reliability-oriented clusters. The partition must support reliability visualization, operator interpretation, and potential deployment under stressed or islanding-prone conditions.

You must reason using only the structured evidence provided in the user prompt.
Do not invent missing data. Do not change the number of clusters.
Do not write operator reports or planning recommendations — partitions only.

Return one valid JSON object only:
{
  "partitions": {
    "0": [bus indices],
    "1": [bus indices],
    ...
  },
  "reasoning_summary": {
    "main_basis": "&lt;short reason&gt;",
    "main_tradeoff": "&lt;short reason&gt;",
    "weakness_to_check": "&lt;short reason&gt;"
  }
}

Rules:
1. Use 0-based bus indices 0–{num_buses − 1}.
2. Every bus must appear exactly once.
3. Prefer contiguous or electrically meaningful clusters unless the evidence
   strongly supports a reliability-driven exception.
4. Do not output markdown, equations, or explanations outside the JSON object.
</pre>

**User prompt template — initial proposal (per agent)**

<pre style="background:#f0f0f0;padding:14px;border-radius:6px;overflow:auto;white-space:pre-wrap">
Agent {ROLE}: {ROLE_DESCRIPTION}.

System objective:
Propose K={num_partitions} clusters that {AGENT_OBJECTIVE}.

Engineering constraints:
- Use 0-based bus indices only.
- {AGENT_SPECIFIC_CONSTRAINTS}

Available metrics:
1. {PRIMARY_EVIDENCE_BLOCK}          # e.g. m_i table (R), A_BN (B), adjacency/d_elec (S), PhiC_i (A)
2. Number of buses: {num_buses}
3. {OPTIONAL_SECONDARY_TABLES}       # EENS_i, co-outage pairs, vvi_i, …

Engineering interpretation:
{SHORT_METRIC_INTERPRETATION}

Clustering preference:
1. …
2. …
3. …

Return the required JSON partition and a short reasoning_summary.
</pre>

| Agent | Role | Primary evidence in the user prompt |
|-------|------|-------------------------------------|
| **R** | Reliability-risk | Marginal outage-conditioned RI $m_i$, optional EENS$_i$ / criticality |
| **B** | BN dependency | Consensus BN adjacency $A_{\mathrm{BN}}$, optional co-outage pairs |
| **S** | Spatial-voltage / topology | Feeder adjacency, $d_{\mathrm{elec}}$, $A_{\mathrm{admit}}$, optional $vvi_i$ |
| **A** | Adequacy / islanding | Per-bus adequacy $\Phi^C_i$ |

**User prompt template — critic / refinement round (shared skeleton)**

<pre style="background:#f0f0f0;padding:14px;border-radius:6px;overflow:auto;white-space:pre-wrap">
Phase: critic feedback.

Agent {ROLE}: {ROLE_DESCRIPTION}.

You previously proposed a partition; the four agents' proposals were fused and the fused
partition was evaluated using deterministic reliability metrics. Revise YOUR proposal to
improve those metrics.

Your previous proposal:
{previous_partitions_json}

Fused-partition evaluation (metric summary):
{
  "Q_RI": …, "Q_BN": …, "PRCS": …, "J_intra": …, "J_sep": …,
  "VVI_g": […], "EENS_g": […], "SAI_g": […], "IRQI_g": […],
  "weak_clusters": […], …
}

Best partition observed so far (target to match or beat):
{best_partition_json}

Your agent-specific diagnosis:
{AGENT_CRITIC_GUIDANCE}
{optional_deployment_gate_block}   # if min IRQI_g fails the gate

Allowed revision:
- You may move at most {edit_budget} buses.
- Do not change the number of clusters ({num_partitions}).
- Focus on the weakest clusters first.

Return one valid JSON object only:
{
  "partitions": {"0": [bus indices], …},
  "reasoning_summary": {
    "moved_buses": […],
    "reason_for_moves": "&lt;short&gt;",
    "expected_metric_improvement": "&lt;short&gt;"
  }
}
</pre>

**Example — Agent R initial user prompt (abbreviated)**

<pre style="background:#f0f0f0;padding:14px;border-radius:6px;overflow:auto;white-space:pre-wrap">
Agent R: Reliability-risk specialist.

System objective:
Propose K=4 clusters that are internally homogeneous in reliability impact and expose high-risk buses clearly.

Engineering constraints:
- Use 0-based bus indices only.
- Do not hide high-risk buses inside low-risk zones unless needed for physical continuity.

Available metrics:
1. Marginal outage-conditioned reliability m_i = E[RI_comb | Bus_i is on outage]:
  Bus|m_i
  0|0.912345
  1|0.887210
  …
  68|0.654321
2. Approximate bus-level EENS_i (descending):
  Bus|EENS_i
  61|12.34
  …
4. Number of buses: 69; required clusters K=4

Clustering preference:
1. Group buses with similar m_i and EENS_i.
2. Avoid mixing very high-risk and very low-risk buses in the same cluster.
3. If a few buses dominate EENS_i, isolate them into an interpretable risk zone when feasible.
4. Preserve reasonable feeder continuity unless reliability evidence strongly suggests otherwise.

Return the required JSON partition and a short reasoning_summary.
</pre>

---

#### Phase 3 — Operator insight generation (orchestrator + specialists)

After an accepted MA-LLM partition is fixed, Stage 3 builds a deterministic metric context and calls LLMs to produce operator-facing JSON (zone ranking, reliability / voltage–islanding / planning briefs, and optional cross-zone action ranking). The partition is **never** changed in this phase.

**System prompt — Phase-3 orchestrator**

<pre style="background:#f0f0f0;padding:14px;border-radius:6px;overflow:auto;white-space:pre-wrap">
You are the Phase-2 orchestrator for operator insight generation on a partitioned IEEE 69-bus feeder.

Task:
Read the accepted partition and its complete metric suite. Rank the clusters from highest to lowest
operational concern, and explain each ranking from the metrics.

The fields f_R (reliability), f_VI (voltage/islanding), and f_P (planning) are pre-computed
deterministically from fixed KPI thresholds and provided in "specialist_flags".
Copy "specialist_flags" verbatim into "priority_zones_for_specialist". Do NOT add, remove, or
re-route clusters.

Do not change the partition. Do not invent missing values.
Base every recommendation on the provided metrics only.

Return one valid JSON object only (no markdown):
{
  "executive_summary": "&lt;2-3 sentences&gt;",
  "zone_ranking": [
    {
      "cluster": int,
      "risk_level": "HIGH|MEDIUM|LOW",
      "main_reason": "&lt;metric-grounded reason&gt;",
      "key_metrics": {"mC_g": …, "EENS_g": …, "VVI_g": …, "SAI_g": …, "IRQI_g": …}
    }
  ],
  "priority_zones_for_specialist": {
    "reliability": [cluster ids],
    "voltage_islanding": [cluster ids],
    "planning": [cluster ids]
  },
  "operator_alerts": [{"cluster": int, "alert": "&lt;short&gt;", "evidence": "&lt;metrics&gt;"}]
}
</pre>

**User prompt template — orchestrator**

<pre style="background:#f0f0f0;padding:14px;border-radius:6px;overflow:auto;white-space:pre-wrap">
{
  "phase2_context": {
    "der_tag": "DER30",
    "K": 4,
    "mC_g": […], "EENS_g": […], "VVI_g": […], "SAI_g": […], "IRQI_g": […],
    "specialist_flags": {
      "reliability": [0, 2],
      "voltage_islanding": [2],
      "planning": [0, 1]
    },
    "specialist_flags_per_cluster": {…},
    "candidate_actions": {…},
    "worst_season_by_cluster": {…},
    …
  }
}
</pre>

**System prompt — specialist example (Reliability, Agent R)**

<pre style="background:#f0f0f0;padding:14px;border-radius:6px;overflow:auto;white-space:pre-wrap">
You are a distribution reliability engineer.

Task:
Analyze every cluster provided in the user message (flagged_clusters). Explain each cluster's
reliability risk using mC_g, EENS_g, LSI_g, marginal critical buses, and worst-season behavior
when available. Choose "recommended_action" ONLY from that cluster's entry in
context.candidate_actions.per_cluster. If no candidate fits, return "monitor".
Do not invent actions outside the provided candidate set. Do not change the partition.

Return JSON only:
{
  "cluster_reliability_reports": [
    {
      "cluster": int,
      "risk_driver": "EENS|mC|critical_bus|seasonal_degradation",
      "interpretation": "&lt;operator-readable explanation&gt;",
      "recommended_action": "&lt;verbatim candidate action, or monitor&gt;",
      "candidate_signature_id": "&lt;signature_id or null&gt;",
      "evidence": {"mC_g": …, "EENS_g": …, "critical_buses": […]}
    }
  ],
  "proactive_schedule": […],
  "system_summary": {
    "main_reliability_bottleneck": "&lt;short&gt;",
    "highest_priority_cluster": int
  }
}
</pre>

Analogous system prompts are used for **voltage / islanding (VI)** and **planning (P)** specialists, and for an optional **cross-zone prioritization** pass that ranks a fixed, code-grounded action list without inventing new actions.

**User prompt template — specialist**

<pre style="background:#f0f0f0;padding:14px;border-radius:6px;overflow:auto;white-space:pre-wrap">
{
  "context": { … same phase2_context object as above … },
  "flagged_clusters": [0, 2],
  "role": "R"
}
</pre>

**Example — Phase-3 orchestrator user message (abbreviated)**

<pre style="background:#f0f0f0;padding:14px;border-radius:6px;overflow:auto;white-space:pre-wrap">
{
  "phase2_context": {
    "der_tag": "DER30",
    "K": 4,
    "mC_g": [0.42, 0.31, 0.55, 0.28],
    "EENS_g": [18.2, 9.1, 24.7, 7.4],
    "VVI_g": [0.08, 0.02, 0.15, 0.01],
    "SAI_g": [0.61, 0.78, 0.44, 0.82],
    "IRQI_g": [0.11, 0.19, 0.04, 0.21],
    "specialist_flags": {
      "reliability": [0, 2],
      "voltage_islanding": [2],
      "planning": [0, 2]
    },
    "candidate_actions": {
      "per_cluster": {
        "2": [
          {
            "signature_id": "R_EENS_high",
            "action": "Prioritize contingency crews and sectionalizing review on zone 2 critical buses",
            "horizon": "near_term",
            "anchor": "EENS_g=24.7",
            "trigger_metrics": ["EENS_g", "mC_g"]
          }
        ]
      }
    }
  }
}
</pre>

---

#### Phase 3 — Operator action ontology and functions

Stage 3 uses a **closed, versioned action ontology** (`OPERATOR_ACTION_ONTOLOGY`, currently **v1.1.0** in `S10_Operator_Insights.py`). LLM specialists **never invent actions**; they **select** from per-cluster `candidate_actions` emitted by deterministic Set-2 KPI signatures in Python, then write the metric-anchored rationale. Each ontology row carries:

| Field | Meaning |
|-------|---------|
| `signature_id` | Stable key (e.g. `converter_hardening`) |
| `horizon` | `planning` (long-term / capital) or `proactive` (stress-window / operational) |
| `owner` | Specialist that may cite it: `R`, `VI`, or `P` |
| `lever` | `capex` or `operational` |
| `cost` | `low` / `med` / `high` (used in cross-zone prioritization) |
| `action` | Fixed operator-facing wording |
| `trigger_metrics` | KPI names that justify the fire |
| `anchor` | Manuscript equation / definition tags for grounding audit |
| `signature` | Human-readable fire condition |

**Ontology catalog (v1.1.0)**

<pre style="background:#f0f0f0;padding:14px;border-radius:6px;overflow:auto;white-space:pre-wrap">
(A) Planning / capital
  converter_hardening          owner=P   | Gamma_g top-ranked zone hosting converters; C_c / double-jeopardy
  volt_var_support             owner=VI  | stress-conditioned CVSI_c &gt; 1+δ (or VVI breach with hosted converters)
  feeder_reinforcement         owner=R   | EENS_g high-percentile AND mC_g low-percentile in the same zone
  storage_der_adequacy         owner=P   | marginal IRQI gate fail with SAI_g the limiting factor (islanding value, not EENS cut)
  exposure_hardening_budget    owner=P   | most-negative DSI_g (peak converter-exposure concentration)

(B) Proactive / operational
  der_active_power_curtailment     owner=VI  | zone hosts stress-conditioned CVSI converter
  conservative_voltage_scheduling  owner=VI  | same CVSI / VVI voltage-channel trigger
  demand_response_load_transfer    owner=R   | EENS_g top zone (non-capital shortfall mitigation)
  prearm_islanding                 owner=VI  | worst-season IRQI(g,s) still passes the gate
  no_islanding_reliance            owner=VI  | annual IRQI passes but worst-season IRQI fails
  crew_prestaging                  owner=R   | EENS_g ranking (highest expected unserved energy)
  converter_inspection             owner=R   | system-level top-C_c converters (not zone-bound)
</pre>

**Deterministic specialist flags** (computed before any LLM call; orchestrator cannot override routing):

<pre style="background:#f0f0f0;padding:14px;border-radius:6px;overflow:auto;white-space:pre-wrap">
f_R(g)  :  mC_g ≤ pct(mC, INSIGHT_R_MC_PCT)  OR  EENS_g ≥ pct(EENS, INSIGHT_R_EENS_PCT)
f_VI(g) :  VVI_g ≥ INSIGHT_VI_TRIGGER_VVI    OR  (SAI applicable ∧ IRQI_g &lt; IRQI_GATE_THRESHOLD)
f_P(g)  :  |DSI_g| ≥ pct(|DSI|, INSIGHT_P_DSI_PCT)   (inactive when DSI unavailable)

Urgency (for action ranking, not LLM risk_level):
  IMMEDIATE  if channel_count ≥ 2 AND IRQI below gate
  HIGH       if channel_count ≥ 2
  MEDIUM     if channel_count = 1
  LOW        otherwise
</pre>

**Core Stage-3 functions** (`S10_Operator_Insights.py`)

<pre style="background:#f0f0f0;padding:14px;border-radius:6px;overflow:auto;white-space:pre-wrap">
build_phase2_context(row, kpis, *, seasonal_records, dj_converters)
  → Serialize accepted-partition cluster KPIs (mC_g, EENS_g, VVI_g, SAI_g, IRQI_g,
    DSI_g, RTR_g, Gamma_g, …) into the Phase-3 evidence bundle; attach
    worst_season_by_cluster, double_jeopardy_converters, and candidate_actions.

compute_specialist_flags(ctx, *, gating_enabled)
  → Deterministic f_R / f_VI / f_P per cluster + trigger margins.
    ROUTING: which clusters each specialist sees.
    RANKING: channel_count / irqi_below_gate / rank_margin for synthesise_actions.

compute_candidate_actions(ctx, seasonal_rows, dj_converters)
  → Evaluate OPERATOR_ACTION_ONTOLOGY signatures in Python.
    Returns {"per_cluster": {g: [records]}, "system": [...],
             "ontology_version", "ontology_hash"}.
    LLM may only SELECT from this closed list.

run_phase2_orchestrator(ctx)
  → LLM zone ranking; specialist_flags overwritten with deterministic flags.

run_phase2_specialist(role, ctx, cluster_ids, system_prompt)
  → Insight-R / VI / P: select candidate actions + write rationale (JSON).

run_phase2_prioritization(ctx, actions)
  → Optional hybrid pass: rank grounded actions across zones; may reorder/explain
    only — cannot invent, reword, or drop actions; urgency tiers re-imposed in code.

synthesise_actions(dashboard)
  → Flatten candidate_actions into the ranked operator action list
    (urgency from flags, tie-break by trigger margin; LLM supplies rationale only).

audit_numeric_grounding(insights_json, evidence_bundle)
  → Faithfulness check: classify numeric literals in orchestrator / specialists /
    prioritization / actions as GROUNDED vs UNGROUNDED against evidence numerics.
</pre>

**Stage-3 pipeline (functional order)**

<pre style="background:#f0f0f0;padding:14px;border-radius:6px;overflow:auto;white-space:pre-wrap">
accepted MA-LLM partition + Set-2 KPIs
        │
        ▼
build_phase2_context  ──►  compute_candidate_actions (ontology fires)
        │
        ▼
compute_specialist_flags  ──►  f_R / f_VI / f_P (routing + ranking)
        │
        ▼
run_phase2_orchestrator  ──►  zone ranking (flags enforced)
        │
        ▼
run_phase2_specialist (R / VI / P)  ──►  select ontology actions + rationale
        │
        ▼
synthesise_actions  ──►  ranked action backbone
        │
        ▼
run_phase2_prioritization (optional)  ──►  cross-zone order + portfolio note
        │
        ▼
audit_numeric_grounding  ──►  faithfulness / grounding rates
</pre>

