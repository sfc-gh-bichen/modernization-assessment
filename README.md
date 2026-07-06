# Modernization Assessment

A Cortex Code skill that produces detailed effort sizing for Snowflake migrations — P50/P70/P90 hour bands, a wave plan, risk register, and a tiered assessment report. It answers **"how long will this take and what does the target architecture look like?"**

Works with any legacy data warehouse or ETL tool. Supports two modes: **Mode A** (code-grounded, using actual DDL/proc inventories) and **Mode B** (questionnaire-only, for early-stage conversations without a code drop).

## What it does

- Profiles source objects: tables, views, stored procedures, ETL pipelines, BI reports
- Matches customer to one of 10 migration scenarios (S1–S10: lift-and-shift through M&A consolidation)
- Designs a target reference architecture (modeling pattern, replication stack, transformation stack, semantic layer, Iceberg posture, governance)
- Scores effort with P50/P70/P90 bands and explains what drives the spread
- Produces a 6-wave migration plan with go/no-go gates
- Outputs a tiered report (exec / architect / engineering) as a Google Doc or Word file

## When to use

Invoke this skill when you need:
- A scoped effort estimate for a Snowflake migration
- A wave plan with milestones
- A target architecture recommendation
- A customer-facing assessment report

Typically used **after** `migration-approach-advisor` picks the target strategy.

## How to use

In Cortex Code, type the trigger phrase or invoke directly:

```
/modernization-assessment
```

Or Cortex Code routes automatically on phrases like:
> *"modernization assessment"*, *"estimate migration effort"*, *"wave plan"*, *"P50/P70/P90"*, *"scope a Snowflake migration"*, *"migration assessment"*

### Workflow overview

1. **Intake** — Source platforms, sizes, driver, constraints, audience tier, Mode A or B
2. **Profile** — Run SnowConvert + `sql-complexity-analysis`; or walk Mode B questionnaire (19 questions)
3. **Pattern selection** — Match scenario, lock target stack, produce reference architecture diagram
4. **Scoring** — Run `loe_calculator.py` for P50/P70/P90 bands
5. **Wave plan** — 6-wave Gantt with go/no-go gates
6. **Report** — Fill audience-tiered template; output as Google Doc or docx
7. **Review** — Checklist sign-off; save final to workspace

### Example

**Prompt:**
> Assess the migration effort for Ariat's BigQuery CDP migration. They have 161 tables (~1 TB), 27 stored procedures (4 high-complexity), 13 views, and a Cloud Function (300 lines, 10 SP calls). Target: Snowflake with dbt + Dynamic Tables. Team has low Snowflake maturity. Audience: architect-level report.

**What the skill produces:**
```
workspace/
  intake.md
  profile/
    inventory.json        ← object counts + complexity from SnowConvert
    questionnaire.yaml    ← Mode B only
  scorecard.json          ← P50/P70/P90 hours + risk composite
  wave-plan.md            ← 6-wave Gantt (mermaid)
  report-final/
    assessment-report.md  ← architect-tier report
```

**Example scorecard output:**
```json
{
  "p50_hours": 1760,
  "p70_hours": 2800,
  "p90_hours": 4000,
  "calendar_weeks": { "p50": 11, "p90": 25 },
  "risk_composite": "Medium-High",
  "top_drivers": ["4 high-complexity UNNEST procs", "Cloud Function orchestration", "low team maturity"]
}
```

## Scripts

```bash
# Mode A — code-grounded scoring
uv run --project . python scripts/loe_calculator.py \
  --mode a \
  --input workspace/profile/inventory.json \
  --output workspace/scorecard.json

# Mode B — questionnaire scoring
uv run --project . python scripts/loe_calculator.py \
  --mode b \
  --input workspace/profile/questionnaire.yaml \
  --output workspace/scorecard.json

# Override team size
uv run --project . python scripts/loe_calculator.py \
  --mode a --input inventory.json --output scorecard.json --team-size 4
```

## Wave plan structure (default)

| Wave | Focus | Duration |
|------|-------|----------|
| W0 | Foundation: network, governance, CI/CD, observability | 6–8 wks |
| W1 | Tables + Bronze + replication retarget | 6–10 wks |
| W2 | Silver Dynamic Tables + incremental pipelines | 6–10 wks |
| W3 | Stored procs + ETL + Gold marts | 6–12 wks |
| W4 | Complex objects (dynamic SQL, Spark, SSIS script tasks) | 6–12 wks |
| W5 | Semantic layer + Cortex + BI repoint + cutover | 6–10 wks |

Go/no-go gates after W1 and W3.

## Reference files

| File | Purpose |
|------|---------|
| `references/scenarios.md` | S1–S10 migration scenario definitions |
| `references/loe-calibration.md` | Scoring context and calibration notes |
| `references/mode-b-questionnaire.md` | 19-question intake for no-code engagements |
| `references/scenario-pattern-matrix.csv` | Scenario → target stack mapping |
| `references/artifact-tool-matrix.csv` | Artifact type → recommended tool |
| `references/report-template-exec.md` | Exec-tier report template |
| `references/report-template-architect.md` | Architect-tier report template |
| `references/report-template-engineering.md` | Engineering-tier report template |
| `references/review-checklist.md` | Pre-delivery quality checklist |

## Related skills

| Skill | Purpose |
|-------|---------|
| `migration-approach-advisor` | Choose the migration strategy first (feeds into this skill) |
| `split-ddl` | Chunk monolithic DDL before SnowConvert |
| `sql-complexity-analysis` | Score stored procedure complexity |
| `spark-migration` | SMA assessment for PySpark workloads |
| `iceberg` | Iceberg posture design |
| `semantic-view` | Semantic layer design |
| `dbt-projects-on-snowflake` | dbt pipeline design |
| `dynamic-tables` | Dynamic Tables pipeline design |
| `docx` / `pptx` | Report document generation |
