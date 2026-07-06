---
name: modernization-assessment
description: "**[REQUIRED]** Use for ALL Snowflake legacy data warehouse modernization assessment tasks. Drives end-to-end scoping, sizing, and report generation for customer migrations from legacy DW (SQL Server, Redshift, Synapse, BigQuery, Oracle, Teradata, Netezza, DB2, Hive, Vertica) and ETL tools (SSIS, Informatica, DataStage, Talend, Pentaho, BTEQ, PySpark, Qlik Replicate, GoldenGate) to Snowflake. Produces effort bands (P50/P70/P90), wave plans, risk registers, and tiered assessment reports. Triggers: modernization assessment, migration assessment, legacy DW to Snowflake, scope a Snowflake migration, size a migration, wave plan, LOE migration, Snowflake migration risk, run assessment kit, estimate migration effort, SnowConvert assessment, migration scoring, Mode B questionnaire."
---

# Modernization Assessment Skill

## Setup

1. **Load** `references/scenarios.md` for scenario matching
2. **Load** `references/loe-calibration.md` for scoring context
3. Verify customer workspace directory exists or create: `<cwd>/<customer_name>/`

## Workflow

### Step 1: Intake (Phase 0)

**Goal:** Capture scope and constraints.

**Ask** user for (use `ask_user_question` for missing fields):
- Source platforms (DB + ETL, with versions and sizes)
- Primary driver: cost / performance / AI / consolidation / regulatory
- Constraints: deadlines, license renewals, parallel-run, regulatory windows
- Audience tier: exec / architect / engineering
- Mode: A (code available) or B (questionnaire only)
- Code drop path (Mode A) or readiness for questionnaire (Mode B)

**Save** intake to `<workspace>/intake.md`.

**⚠️ MANDATORY STOPPING POINT**: Confirm intake is complete before proceeding.

### Step 2: Profile (Phase 1)

**Mode A (code-grounded):**
1. Invoke `split-ddl` skill on monolithic DDL exports
2. Run SnowConvert assessment. Capture: object counts by type, conversion status, EWI severity, dependency waves, dynamic SQL inventory
3. For PySpark/Spark: invoke `spark-migration` skill (SMA)
4. For stored procs: invoke `sql-complexity-analysis` skill
5. Save outputs to `<workspace>/profile/`

**Mode B (questionnaire):**
1. **Load** `references/mode-b-questionnaire.md`
2. Walk user through all 19 questions interactively
3. Save answers to `<workspace>/profile/questionnaire.yaml`

**Output:** Inventory summary (one-page).

### Step 3: Pattern Selection (Phase 2-3)

1. **Load** `references/scenarios.md` — match customer to S1-S10
2. **Load** `references/scenario-pattern-matrix.csv` — lock target stack
3. **Load** `references/artifact-tool-matrix.csv` — map artifacts to tools

For each area, recommend:
- Modeling pattern (Inmon/Kimball/DV2/OBT) + medallion layering
- Replication stack (Openflow/Qlik/GoldenGate)
- Transformation stack (dbt+DT/Snowpark/Tasks)
- Semantic layer (Semantic View + Cortex Analyst)
- Iceberg posture (none/managed/external/linked)
- Governance overlay + CI/CD posture

Produce **Target Reference Architecture** as mermaid diagram.

**⚠️ MANDATORY STOPPING POINT**: Get user approval of target stack before scoring.

### Step 4: Scoring (Phase 4)

**Mode A:**
```bash
uv run --project <SKILL_DIR> python <SKILL_DIR>/scripts/loe_calculator.py \
  --mode a --input <workspace>/profile/inventory.json --output <workspace>/scorecard.json
```

Or build interactively: map SnowConvert outputs to `ObjectInventory` entries, EWI counts, dynamic SQL classifications, and multipliers.

**Mode B:**
```bash
uv run --project <SKILL_DIR> python <SKILL_DIR>/scripts/loe_calculator.py \
  --mode b --input <workspace>/profile/questionnaire.yaml --output <workspace>/scorecard.json
```

**Output:** `scorecard.json` with P50/P70/P90 hours, calendar weeks, risk composite.

Present scorecard to user. Comment on what drives the P90-P50 spread.

### Step 5: Wave Plan (Phase 5)

Default 6-wave structure:
- **W0** Foundation (network, governance, CI/CD, observability)
- **W1** Tables + Bronze + replication retarget
- **W2** Silver Dynamic Tables + incremental pipelines
- **W3** Stored procs + ETL + Gold marts
- **W4** Complex objects (dynamic SQL, Spark, SSIS script tasks)
- **W5** Semantic layer + Cortex + BI repoint + cutover

Rules:
- Each wave 6-12 weeks; rebalance if needed
- Go/no-go gates at end of W1 and W3
- Quick wins first (tables + simple views in W1)

Produce mermaid Gantt diagram.

### Step 6: Author Report (Phase 6)

**Load** the appropriate template based on audience tier:
- Exec: `references/report-template-exec.md`
- Architect: `references/report-template-architect.md`
- Engineering: `references/report-template-engineering.md`

Fill template with:
- Intake fields, scenario fit, target architecture diagram
- Scorecard (P50/P70/P90), risk register, wave Gantt
- Citations to KB sections for engineering tier

Output via `docx` skill or Google Doc (`mcp_google-worksp_create_document`).

**⚠️ MANDATORY STOPPING POINT**: Present report draft before delivery.

### Step 7: Review (Phase 7)

1. **Load** `references/review-checklist.md`
2. Run through checklist; flag any gaps
3. Present gaps to user for resolution
4. After acceptance, save final to `<workspace>/report-final/`

## Tools

### Script: loe_calculator.py

**Description**: LOE/Risk scoring engine. Supports Mode A (code-grounded inventories) and Mode B (questionnaire bins).

**Usage:**
```bash
uv run --project <SKILL_DIR> python <SKILL_DIR>/scripts/loe_calculator.py \
  --mode [a|b] --input <input_file> --output <output_file>
```

**Arguments:**
- `--mode`: `a` (inventory JSON) or `b` (questionnaire YAML)
- `--input`: Path to inventory/questionnaire file
- `--output`: Path for scorecard JSON output
- `--team-size`: Override team size (default: from input)

### Integrated Skills

| Phase | Skill | Purpose |
|-------|-------|---------|
| Profile | `split-ddl` | Chunk monolithic DDL exports |
| Profile | `sql-complexity-analysis` | Score proc complexity |
| Profile | `spark-migration` | Run SMA on PySpark |
| Pattern | `iceberg` | Iceberg posture design |
| Pattern | `semantic-view` | Semantic layer design |
| Transform | `dbt-projects-on-snowflake` | dbt pipeline design |
| Transform | `dynamic-tables` | DT pipeline design |
| Governance | `data-governance` | Masking/RAP/classification |
| Deploy | `dcm` | Object lifecycle IaC |
| Report | `docx` or `pptx` | Document generation |

## Stopping Points

- ✋ After Step 1 (intake confirmation)
- ✋ After Step 3 (target stack approval)
- ✋ After Step 6 (report draft review)

## Troubleshooting

**No code drop yet:** Use Mode B. Document LOE as preliminary; re-baseline when code arrives.

**SnowConvert doesn't cover source dialect:** Note as risk in tooling-fit dimension; budget manual conversion. Cite artifact-tool-matrix row.

**Multiple legacy stacks:** Apply S10 (M&A consolidation). Produce per-source inventories + unified target architecture.

**Databricks coexistence:** Apply S7. Use `iceberg` skill for catalog-linked database.

**AI/Cortex only (no migration):** Apply S8. Skip replication/transformation; focus on Semantic Views + Cortex.

## Best Practices

- Always run SnowConvert first when code is available
- Produce P50/P70/P90 bands, never single-point estimates
- Default new transforms to dbt + Dynamic Tables + DCM
- Default replication to Openflow when source is in catalog; else retain/retarget Qlik
- Default semantic layer to Snowflake Semantic Views + Cortex Analyst
- Apply governance at table-create time in Silver, not retrofitted
- Keep one workspace per customer; never overwrite kit references
