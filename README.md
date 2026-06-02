# Archi — CMS Physics Regression Testing Agent

**Archi** is an LLM-powered agent framework for operational intelligence, extended here with a purpose-built **CMS Reconstruction Regression Validation Agent** that automates performance testing for particle physics reconstruction code changes.

---

## New: CMS Regression Validation Agent

### Problem it solves

Every time a developer modifies CMS reconstruction code (muon ID, tracking, ECAL, jets), a human expert must manually:
- Run validation jobs on reference samples
- Compare output metrics to historical baselines
- Decide whether to approve or block the merge

This takes days and is a bottleneck for the ~1000 commits/year to CMS reconstruction software.

### What the agent does

The agent automates the full validation pipeline end-to-end:

```
Git commit detected
       │
       ▼  AUTOMATED
[1] git_diff_analyzer          — identify changed files + affected subsystem
       │
       ▼  AUTOMATED (LLM)
[2] Subsystem impact analysis  — code risk assessment via BaseReActAgent
       │
       ▼  AUTOMATED
[3] root_metric_extractor      — read NanoAOD ROOT file with uproot,
    metrics_db_query           — compute tight-ID efficiency per pT bin,
                                 compare new vs baseline
       │
       ▼  AUTOMATED
[4] regression_detector        — classify deviations:
                                 SAFE (<2%) / INVESTIGATE (2–5%) / ESCALATE (>5%)
       │
       ├─ All metrics ≥ 0%  → AUTO_APPROVE ✓  (no human needed)
       │
       ├─ Regression 2–5%  → NEEDS_HUMAN_REVIEW ★
       │                      ApprovalWorkflow sends proposal to reviewers
       │
       └─ Regression >5%   → AUTO_REJECT + jira_escalator ✗
                              JIRA ticket opened in CMS-RECO project
                              Notifies: cms-reco-conveners@cern.ch, muon-pog@cern.ch
                              Human can still override to approve
       │
       ▼
[5] report_generator           — save full JSON validation report
```

### Key design principles

- **Auto-approve without human when new code is better** — zero friction for improvements
- **Human gate only on regression** — reviewers are not interrupted for passing commits
- **Hard sandbox** — grid jobs submit to `test_queue` only; `cms_prod` is blocked
- **Escalation is automatic** — no human needs to remember to open a JIRA ticket

---

## Agent Architecture

### New files added to Archi

```
src/archi/pipelines/agents/
├── cms_regression_validation_agent.py   Agent class (extends BaseReActAgent)
├── cms_approval_workflow.py             Human-in-the-loop state machine
└── tools/
    └── cms_regression_tools.py          8 LangChain tool factories

examples/agents/
└── cms-regression-validation.md        AgentSpec (tool list + system prompt)
```

### `CMSRegressionValidationAgent`

Extends Archi's `BaseReActAgent` (LangGraph ReAct loop) with 11 tools:

| Tool | Source | What it does |
|------|--------|-------------|
| `root_metric_extractor` | `cms_regression_tools` | Read NanoAOD ROOT via uproot, compute tight-ID efficiency per pT bin; accepts `fail_pct` to simulate code changes |
| `git_diff_analyzer` | `cms_regression_tools` | Parse git diff → changed files → affected subsystem |
| `metrics_db_query` | `cms_regression_tools` | Fetch baseline or compare new vs baseline (`mode=fetch\|compare`) |
| `regression_detector` | `cms_regression_tools` | Classify deviations as SAFE / INVESTIGATE / ESCALATE |
| `report_generator` | `cms_regression_tools` | Save JSON validation report |
| `jira_escalator` | `cms_regression_tools` | Open CMS-RECO JIRA ticket |
| `grid_job_submitter` | `cms_regression_tools` | Submit jobs to test_queue (sandbox-only) |
| `metric_parser` | `cms_regression_tools` | Parse grid job output metrics |
| `search_local_files` | Archi standard | Search indexed documents |
| `search_metadata_index` | Archi standard | Query file metadata catalog |
| `fetch_catalog_document` | Archi standard | Fetch full document by hash |

### `ApprovalWorkflow`

File-based human-in-the-loop state machine:

```
create_proposal(proposal_id, commit, subsystem, samples, ...)
       ↓
  Proposal saved as JSON → reviewers@cern.ch notified
       ↓
request_approval(proposal_id, ...)
  • auto_approve=True  → approves after delay (demo/CI)
  • auto=True          → rejects immediately (CI no-human)
  • interactive        → waits for human to edit proposal file or respond to prompt
       ↓
  APPROVED | REJECTED | TIMED_OUT (48h)
       ↓
escalate(proposal_id, regressions)   ← only on ESCALATE_AND_BLOCK
  → Creates escalation JSON, logs JIRA ticket
```

---

## CMS Open Data Example

Demonstrates the full agent pipeline using **real CMS detector data** from the CERN Open Data Portal.

### Data

**Source:** [CERN Open Data Portal — Record 545](https://opendata.cern.ch/record/545)  
**Dataset:** Dimuon events from CMS Run2011A (7 TeV proton–proton collisions)

| File | Events | Physics process | pT range |
|------|--------|----------------|---------|
| `Zmumu.csv` | 10 000 | Z boson → μμ | ~20–80 GeV |
| `Jpsimumu.csv` | 20 000 | J/ψ meson → μμ | ~5–15 GeV |

The two samples together cover a wide muon pT range, which is essential: a calibration bug that only affects low-pT muons (J/ψ) would be missed by a Z-only validation.

### Baseline metrics (uproot + NanoAOD-style ROOT)

```python
import uproot, numpy as np, json

file    = uproot.open("data/data_baseline.root")
tree    = file["Events"]
muon_pt = tree["Muon_pt"].array(library="np")
muon_id = tree["Muon_tightId"].array(library="np")

pt_bins = [10, 20, 50, 100, 1000]
metrics = {}
for i, threshold in enumerate(pt_bins[:-1]):
    mask     = (muon_pt > threshold) & (muon_pt < pt_bins[i+1])
    id_tight = muon_id[mask].sum()
    total    = mask.sum()
    if total > 0:
        metrics[f"tight_id_eff_{threshold}_{pt_bins[i+1]}"] = float(id_tight / total)
```

Produces the baseline:

| Metric | Value | pT bin | Muons |
|--------|-------|--------|-------|
| `tight_id_eff_10_20` | 0.9262 | 10–20 GeV | 12 263 |
| `tight_id_eff_20_50` | 0.7207 | 20–50 GeV | 17 720 |
| `tight_id_eff_50_100` | 0.7129 | 50–100 GeV | 2 494 |
| `tight_id_eff_100_1000` | 0.6667 | 100–1000 GeV | 102 |

### Simulated code change

```python
# MuonIdProducer.cc bug: 10% of tight-ID selections randomly fail
np.random.seed(42)
fail_mask   = np.random.rand(len(muon_id)) < 0.10
muon_id_new = muon_id & ~fail_mask   # flip 10% of True → False
```

The agent detects this automatically:

```
tight_id_eff_10_20      0.9262 → 0.8329   −10.1%  ✗ CRITICAL_REGRESSION
tight_id_eff_20_50      0.7207 → 0.6474   −10.2%  ✗ CRITICAL_REGRESSION
tight_id_eff_50_100     0.7129 → 0.6480    −9.1%  ✗ CRITICAL_REGRESSION
tight_id_eff_100_1000   0.6667 → 0.5882   −11.8%  ✗ CRITICAL_REGRESSION

Agent verdict:  ESCALATE_AND_BLOCK
JIRA ticket:    CMS-RECO-2131  →  cms-reco-conveners@cern.ch, muon-pog@cern.ch
Final:          ✗  BLOCKED  (merge rejected)
```

And auto-approves improvements without human involvement:

```
tight_id_eff_10_20      0.9262 → 0.9429   +1.8%   ↑ improvement
tight_id_eff_20_50      0.7207 → 0.7336   +1.8%   ↑ improvement
...

Agent verdict:  SAFE_TO_MERGE
Final:          ✓  MERGED  (auto-approved, no human review)
```

---

## Setup

```bash
# 1. Clone
git clone https://github.com/July98293/archi-physics-regression-test
cd archi-physics-regression-test

# 2. Install dependencies
pip install uproot numpy awkward langchain langchain-core psycopg2-binary langchain-mcp-adapters

# 3. Build ROOT file from CMS Open Data CSVs
python examples/cms-open-data/make_root.py

# 4. Compute baseline
python examples/cms-open-data/compute_baseline.py
```

## Run the example

```bash
# Regression: 10% tight-ID failure → ESCALATE_AND_BLOCK → BLOCKED
python examples/cms-open-data/agent_pipeline.py --commit def7890 --auto

# Improvement: tighter isolation → SAFE_TO_MERGE → AUTO-APPROVED
python examples/cms-open-data/agent_pipeline.py --commit abc1234 --improvement --auto

# Interactive: prompts for human approval on regression
python examples/cms-open-data/agent_pipeline.py --commit def7890

# Human override: approve despite regression
python examples/cms-open-data/agent_pipeline.py --commit def7890 --auto-approve
```

---

## File structure

```
archi-physics-regression-test/
│
├── README.md                                    ← this file
│
├── src/archi/pipelines/agents/
│   ├── cms_regression_validation_agent.py       agent class (BaseReActAgent)
│   ├── cms_approval_workflow.py                 human-in-the-loop approval
│   └── tools/
│       └── cms_regression_tools.py              8 LangChain tool factories
│
├── examples/agents/
│   └── cms-regression-validation.md             AgentSpec (tool list + prompt)
│
└── examples/cms-open-data/
    ├── README.md                                example walkthrough
    ├── make_root.py                             CSV → NanoAOD ROOT (one-time)
    ├── compute_baseline.py                      compute baseline_metrics.json
    ├── agent_pipeline.py                        full agent pipeline (git trigger → report)
    ├── baseline_metrics.json                    reference baseline (committed)
    ├── agent_config.yaml                        thresholds + approval policy
    ├── sample_metadata.csv                      sample registry
    └── data/
        ├── Zmumu.csv                            real CMS Z→μμ events (CERN OD)
        ├── Jpsimumu.csv                         real CMS J/ψ→μμ events (CERN OD)
        └── data_baseline.root                   NanoAOD-style ROOT file (generated)
```

---

## Built on

- **[Archi](https://github.com/cern-it-efp/archi)** — LLM agent framework (LangGraph + LangChain)
- **[CERN Open Data Portal](https://opendata.cern.ch)** — real CMS Run2011A collision data
- **[uproot](https://github.com/scikit-hep/uproot5)** — pure-Python ROOT I/O
- **[CMS Software](https://cms-sw.github.io)** — reconstruction framework context
