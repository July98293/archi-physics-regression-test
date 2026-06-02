# Validation Regression Detection Agent for CMS from Archi model

Scope: create an automated detection of performance regressions when reconstruction code changes using Archi's document retrieval, reasoning, and tool orchestration to reduce manual validation work and accelerate code review.

## Curent status

Starting from cloning archi repo, i add:

- cms_regression_validation_agent.py  
- cms_approval_workflow.py  
- src/archi/pipelines/agents/tools/cms_regression_tools.py 
- examples/agents/cms-regression-validation.md

Then dowload data from CMS Open Data Sample https://opendata.cern.ch - Record 545 — real CMS detector data from 2011 (LHC Run2011A, 7 TeV proton-proton collisions).

Three scripts, converts real CMS CSV data in NanoAOD-style ROOT file, one create the baseline metrics and te agent_pipeline.py is the automated evaluation on every git trigger, that recall human help only in certain condition:

Δ per metric	Agent verdict	Human needed?
≥ 0% (equal or better)	AUTO_APPROVE ✓	No
−2% to 0%	SAFE	No
−5% to −2%	NEEDS_HUMAN_REVIEW ⚠	Yes
< −5%	AUTO_REJECT ✗	Yes (override only)

The two results
Regression (def7890 — fails 10% of tight IDs):


tight_id_eff_10_20    0.9262 → 0.8329   -10.1%  ✗ CRITICAL_REGRESSION
tight_id_eff_20_50    0.7207 → 0.6474   -10.2%  ✗ CRITICAL_REGRESSION
→ AUTO_REJECT  ·  human shown prompt  ·  chose REJECT  →  ✗ BLOCKED

Improvement (abc1234 — recovers isolated muons):


tight_id_eff_10_20    0.9262 → 0.9306   +0.5%   ↑ IMPROVEMENT
tight_id_eff_20_50    0.7207 → 0.7339   +1.8%   ↑ IMPROVEMENT
→ AUTO_APPROVED  ·  no human involved  →  ✓ MERGED

The bigger vision is ofc much detailed and bigger, here it go:

## Problem

When CMS reconstruction code changes, analysts must manually run reference samples through old and new code, compare metrics and document findings in tickets

## Solution

**Validation Regression Detection Agent** automates:
- **Code change detection** : Monitors git commits, identifies reconstruction code changes
- **Baseline retrieval** : Fetches historical performance metrics for affected subsystems
- **Validation orchestration** : Proposes validation jobs with appropriate reference samples
- **Regression analysis** : Compares output metrics, flags deviations from baseline
- **Report generation** : Creates actionable reports with diagnostics, recommendations

**Built with Archi**, combining:
- Document retrieval (git diffs, reconstruction configs, past validation reports)
- LLM-powered reasoning (code impact analysis, anomaly classification)
- Tool integration (grid job submission, metric querying, report rendering)
- Human review checkpoints (no production action without approval)

---

## Architecture

```
Git Commit → [Code Change Detection] 
            ↓
        [Subsystem Impact Analysis] ← Reconstruction Config DB
            ↓
        [Baseline Metric Retrieval] ← Metrics Database
            ↓
        [Reference Sample Selection] ← Sample Metadata
            ↓
        [Validation Job Proposal] → [Human Review]
            ↓ (approved)
        [Grid Job Submission]
            ↓
        [Output Parsing & Metric Comparison]
            ↓
        [Regression Detection & Classification]
            ↓
        [Report Generation + Escalation]
```

### Data Sources (Archi Integration)

| Source | Type | Purpose | Archi Method |
|--------|------|---------|--------------|
| Reconstruction repo | Git | Code diffs, change scope | Git scraping |
| Baseline metrics | CSV/JSON + API | Historical performance | Manual upload + API client |
| Reference samples | File metadata | Validation datasets | Local document store |
| Validation reports | JIRA tickets | Past regression patterns | JIRA ingestion |
| Reconstruction config | YAML/JSON files | Affected subsystems, params | Git scraping |
| Grid logs | EOS/logs endpoint | Job diagnostics | REST API client |

### Core Workflows

**Workflow 1: Detect & Propose**
```
1. Poll git repo for new commits → Agent detects reconstruction code change
2. LLM analyzes diff: "Which subsystems affected?"
3. Query baseline metrics DB: fetch metrics for muon chambers, electron ID, etc.
4. Retrieve reference samples from metadata
5. Draft validation proposal: samples to run, estimated cost, risk level
6. Output: Validation proposal ticket (awaiting human approval)
```

**Workflow 2: Execute & Analyze** (post-approval)
```
1. Grid job submission (pre-approved by analyst)
2. Monitor job status, fetch output when complete
3. Parse output metrics, align with baseline
4. Compute deviations (efficiency Δ%, fake-rate Δ%, etc.)
5. Classify: "safe" (<2%), "investigate" (2-5%), "reject" (>5%)
6. Generate report with comparison plots, recommendations
7. Escalate if needed (email expert, create follow-up ticket)
```


---

## Metrics & Success Criteria

Track during pilot deployment:

| Metric | Target | Method |
|--------|--------|--------|
| Code change detection latency | <5 min | Log timestamps |
| Validation proposal accuracy | >95% | Compare agent selection vs expert selection |
| Regression detection recall | >90% | Retrospective: does agent catch known regressions? |
| False positive rate | <5% | Fraction of flags analyst rejects as noise |
| Analyst time saved per validation | >1 hour | Survey: manual vs agent-assisted time |
| Human review bottleneck | <10 min per proposal | Measure approval latency |

---


## Implementation Roadmap

This is  a quick mvp, a real timeline will be:

**Phase 1 (Weeks 1-4):** Prototype & Design
- Set up data sources (git, baseline DB, JIRA)
- Implement code change detection + subsystem impact analysis
- Build validation proposal workflow
- Deploy to sandbox; test on past commits

**Phase 2 (Weeks 5-8):** Execution & Analysis
- Integrate grid job submission (approval-gated)
- Implement metric parsing, regression detection
- Add report generation
- Pilot with muon reconstruction team

**Phase 3 (Weeks 9-12):** Refinement & Scale
- Collect metrics, refine thresholds
- Extend to additional subsystems (electron ID, tau reco, etc.)
- Integrate escalation workflows
- Deploy to production operations

---

## Using Archi's Core Functions

This agent combines Archi's primitives:

1. **Document Retrieval** — Fetch diffs, configs, past reports
   ```python
   from archi.retrieval import DocumentRetriever
   retriever = DocumentRetriever(knowledge_base)
   diffs = retriever.query("muon reconstruction code changes")
   ```

2. **LLM Reasoning** — Analyze impact, classify regressions
   ```python
   from archi.llm import ReasoningAgent
   agent = ReasoningAgent()
   impact = agent.analyze("Which subsystems affected by this diff?", diffs)
   ```

3. **Tool Orchestration** — Coordinate grid jobs, API calls, reporting
   ```python
   from archi.tools import ToolManager
   tools = ToolManager(["grid_submitter", "metric_parser", "report_gen"])
   result = tools.execute_workflow(validation_plan)
   ```

4. **Human Review** — Feedback loops, approval gates
   ```python
   from archi.approval import ApprovalWorkflow
   workflow = ApprovalWorkflow()
   workflow.request_approval(proposal, reviewer="reco-experts@cern.ch")
   ```
---

## Future Work

- Integrate detector geometry validation
- Add ML-based regression prediction (before running jobs)
- Distributed validation across multiple grid sites
- Real-time dashboards for validation status
