# Validation Regression Detection Agent for CMS

Automated detection of performance regressions when reconstruction code changes. Uses Archi's document retrieval, reasoning, and tool orchestration to reduce manual validation work and accelerate code review.

## Problem

When CMS reconstruction code changes, analysts must:
1. Manually run reference samples through old and new code
2. Compare metrics (efficiency, resolution, fake-rate) across detector subsystems
3. Generate comparison reports and flag regressions
4. Document findings in tickets

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

## Example: Muon Reconstruction Code Change

### Setup

1. **Clone the repo and data:**
   ```bash
   git clone https://github.com/cms-sw/cmssw.git  # Real CMS code
   cd cmssw
   git checkout muon-reco-branch  # Example branch
   ```

2. **Create local knowledge base:**
   ```
   knowledge_base/
   ├── configs/
   │   ├── muon_reco.yaml              # Reconstruction params
   │   └── baseline_metrics.json        # Baseline perf by subsystem
   ├── reference_samples/
   │   ├── sample_metadata.csv          # List of validation samples
   │   └── sample_locations.txt         # EOS paths
   ├── git_history/
   │   └── past_validations.md          # Prior regression incidents
   └── tickets/
       └── validation_tickets.json      # JIRA export
   ```

3. **Example `baseline_metrics.json`:**
   ```json
   {
     "muon_chambers": {
       "efficiency": 0.9850,
       "efficiency_error": 0.0015,
       "fake_rate": 0.0032,
       "fake_rate_error": 0.0005,
       "resolution_pt": 0.045,
       "resolution_eta": 0.0012,
       "reference_date": "2025-05-15",
       "reference_run": "run365000"
     },
     "electron_id": {
       "efficiency": 0.9245,
       "efficiency_error": 0.0018,
       "fake_rate": 0.0078,
       "fake_rate_error": 0.0008
     }
   }
   ```

4. **Example `sample_metadata.csv`:**
   ```csv
   sample_name,physics_process,n_events,eos_path,detector_coverage,purpose
   Z_to_mumu_100k,Drell-Yan,100000,/eos/cms/store/samples/zmumu_2025_v1/,muon_chambers,efficiency
   Jpsi_to_mumu_50k,Quarkonium,50000,/eos/cms/store/samples/jpsi_2025_v1/,muon_chambers,mass_resolution
   WtoEnu_100k,W_boson,100000,/eos/cms/store/samples/wenu_2025_v1/,electron_id,efficiency
   ```

### Agent Configuration

Create `agent_config.yaml`:
```yaml
agent:
  name: "ValidationRegressionDetector"
  description: "Detect reconstruction code regressions via automated validation"
  
knowledge_base:
  sources:
    - name: "cmssw_git"
      type: "git_scraping"
      url: "https://github.com/cms-sw/cmssw.git"
      paths: ["Reconstruction/Muon/**", "Reconstruction/EGamma/**"]
      
    - name: "baseline_metrics"
      type: "manual_upload"
      path: "./knowledge_base/configs/baseline_metrics.json"
      update_interval: "weekly"
      
    - name: "reference_samples"
      type: "local_documents"
      path: "./knowledge_base/reference_samples/"
      
    - name: "validation_tickets"
      type: "jira"
      url: "https://cms.jira.cern.ch"
      project: "RECO"
      query: "labels=validation"
      
    - name: "reconstruction_config"
      type: "git_scraping"
      url: "https://github.com/cms-sw/cmssw.git"
      paths: ["Reconstruction/**/*.yaml", "Reconstruction/**/*.cff"]

tools:
  - name: "git_diff_analyzer"
    action: "Analyze code diff, identify affected subsystems"
    
  - name: "metrics_database_query"
    action: "Fetch baseline metrics for subsystem"
    
  - name: "grid_job_submitter"
    action: "Submit validation job to WLCG grid (sandbox: test queue only)"
    requires_approval: true
    
  - name: "metric_parser"
    action: "Extract performance metrics from validation output"
    
  - name: "report_generator"
    action: "Create comparison report with plots, recommendations"
    
  - name: "jira_escalator"
    action: "Create or update validation ticket (draft mode)"
    requires_approval: true

policy:
  sandbox_mode: true
  requires_human_approval:
    - grid_job_submission
    - ticket_creation
    - escalation
  regression_thresholds:
    safe: 0.02           # <2% deviation
    investigate: 0.05    # 2-5%
    reject: 1.0          # >5%
  auto_escalate_above: 0.05
  notify_on_change: ["reco-experts@cern.ch", "dqm-team@cern.ch"]

metrics:
  track:
    - code_change_detection_latency
    - validation_proposal_accuracy
    - regression_detection_recall
    - false_positive_rate
    - analyst_review_time_saved
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

## Getting Started

1. **Install Archi:**
   ```bash
   git clone https://github.com/archi-physics/archi.git
   cd archi
   pip install -e .
   ```

2. **Clone this example:**
   ```bash
   git clone https://github.com/[your-fork]/cms-validation-agent.git
   cd cms-validation-agent
   ```

3. **Set up knowledge base:**
   ```bash
   ./setup_knowledge_base.sh  # Downloads sample data, git repos
   ```

4. **Run agent on test data:**
   ```bash
   python agent.py --config agent_config.yaml --dry-run  # No grid submission
   python agent.py --config agent_config.yaml --watch cmssw.git  # Live mode
   ```

---

## Future Work

- Integrate detector geometry validation
- Add ML-based regression prediction (before running jobs)
- Distributed validation across multiple grid sites
- Real-time dashboards for validation status
