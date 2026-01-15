# EvidenceFirst — Transaction Alert Disposition Controls Testing Copilot (Snowflake)

A demo project that shows how to use **Snowflake Cortex (Search + Analyst + Agents)** to perform **controls testing** over a transaction monitoring alert population.

**Control intent (example):** *Transaction monitoring alerts must be reviewed and dispositioned within **48 hours**, with required documentation (notes, reason, evidence references) and supervisor approval for high-severity closures.*

---

## What you get

1. **Structured alert population table** (alerts + assignments + disposition metadata)
2. **Unstructured policies & procedures** stored in Snowflake and indexed with **Cortex Search**
3. **Cortex Analyst semantic model (`.yml`)** to enable reliable structured querying
4. **An AI Agent** that combines policy retrieval + population analysis to produce:
   - population health metrics
   - exception lists (SLA breaches, missing fields, missing approvals)
   - a workpaper-style testing summary

---

## Architecture

### Structured data (for testing & metrics)
- `TM_CONTROLS.DATA.TXN_ALERTS` — alert population with closure/disposition fields  
- `TM_CONTROLS.DATA.CONTROL_PARAMETERS` — control configuration (SLA = 48h, documentation requirements)

### Unstructured data (for policy grounding)
- `TM_CONTROLS.DATA.TM_POLICIES` — policy/procedure text stored as rows
- `TM_POLICY_SEARCH` (Cortex Search service) — indexes policy text for retrieval

### Cortex components
- **Cortex Search**: retrieves policy requirements and documentation standards
- **Cortex Analyst**: answers structured questions (counts, rates, exceptions) using the semantic model
- **Cortex Agents**: orchestrates both and produces a consolidated controls testing output

### UI (optional but recommended)
- **Streamlit in Snowflake** to demo the copilot with dropdowns (period, severity, analyst) and one-click outputs.

---

## Setup

### 1) Create role, database, schema, and warehouse

Run as `ACCOUNTADMIN` (or a role with equivalent privileges):

```sql
USE ROLE ACCOUNTADMIN;

-- Role
CREATE OR REPLACE ROLE tm_controls_role;
GRANT DATABASE ROLE SNOWFLAKE.CORTEX_USER TO ROLE tm_controls_role;

SET my_user = CURRENT_USER();
GRANT ROLE tm_controls_role TO USER IDENTIFIER($my_user);

-- Database + schema
CREATE OR REPLACE DATABASE tm_controls;
CREATE OR REPLACE SCHEMA tm_controls.data;

GRANT USAGE ON DATABASE tm_controls TO ROLE tm_controls_role;
GRANT USAGE ON SCHEMA tm_controls.data TO ROLE tm_controls_role;

-- Allow object creation in schema (tables, stages, cortex search services, etc.)
GRANT CREATE TABLE ON SCHEMA tm_controls.data TO ROLE tm_controls_role;
GRANT CREATE VIEW ON SCHEMA tm_controls.data TO ROLE tm_controls_role;
GRANT CREATE STAGE ON SCHEMA tm_controls.data TO ROLE tm_controls_role;
GRANT CREATE CORTEX SEARCH SERVICE ON SCHEMA tm_controls.data TO ROLE tm_controls_role;

-- Warehouse
CREATE OR REPLACE WAREHOUSE tm_controls_wh
  WAREHOUSE_SIZE='SMALL'
  AUTO_SUSPEND=3600
  AUTO_RESUME=TRUE
  INITIALLY_SUSPENDED=FALSE;

GRANT USAGE, OPERATE ON WAREHOUSE tm_controls_wh TO ROLE tm_controls_role;

-- (Optional) Agent schema pattern
CREATE DATABASE IF NOT EXISTS snowflake_intelligence;
CREATE SCHEMA IF NOT EXISTS snowflake_intelligence.agents;

GRANT USAGE ON DATABASE snowflake_intelligence TO ROLE tm_controls_role;
GRANT USAGE ON SCHEMA snowflake_intelligence.agents TO ROLE tm_controls_role;
GRANT CREATE AGENT ON SCHEMA snowflake_intelligence.agents TO ROLE tm_controls_role;

ALTER USER IDENTIFIER($my_user) SET DEFAULT_ROLE = tm_controls_role;
```

> **Common privilege pitfall:** If you created the schema/tables as `ACCOUNTADMIN` and then switched roles, your role may not have the `CREATE` privileges. The grants above fix that.

---

## Fixing the error: “Insufficient privileges to operate on schema 'DATA'”

This typically means your role can **USE** the schema but cannot **create/replace** objects inside it (often when creating the Cortex Search Service), or you’re attempting an operation that requires additional privileges.

Run the following as `ACCOUNTADMIN`:

```sql
USE ROLE ACCOUNTADMIN;

-- Ensure schema usage
GRANT USAGE ON DATABASE TM_CONTROLS TO ROLE TM_CONTROLS_ROLE;
GRANT USAGE ON SCHEMA TM_CONTROLS.DATA TO ROLE TM_CONTROLS_ROLE;

-- Ensure you can create objects in the schema
GRANT CREATE TABLE ON SCHEMA TM_CONTROLS.DATA TO ROLE TM_CONTROLS_ROLE;
GRANT CREATE VIEW ON SCHEMA TM_CONTROLS.DATA TO ROLE TM_CONTROLS_ROLE;
GRANT CREATE STAGE ON SCHEMA TM_CONTROLS.DATA TO ROLE TM_CONTROLS_ROLE;

-- Required for Cortex Search Services
GRANT CREATE CORTEX SEARCH SERVICE ON SCHEMA TM_CONTROLS.DATA TO ROLE TM_CONTROLS_ROLE;

-- If you already created objects and need to modify them (ALTER/REPLACE),
-- ensure ownership or grant OWNERSHIP (use carefully):
-- GRANT OWNERSHIP ON SCHEMA TM_CONTROLS.DATA TO ROLE TM_CONTROLS_ROLE COPY CURRENT GRANTS;
```

If the error happens specifically when using a warehouse, also confirm:

```sql
GRANT USAGE, OPERATE ON WAREHOUSE TM_CONTROLS_WH TO ROLE TM_CONTROLS_ROLE;
```

---

## 2) Create tables + load sample data

Switch to your role and warehouse:

```sql
USE ROLE tm_controls_role;
USE WAREHOUSE tm_controls_wh;
USE DATABASE tm_controls;
USE SCHEMA data;
```

Create the alert population:

```sql
CREATE OR REPLACE TABLE txn_alerts (
  alert_id STRING,
  alert_created_ts TIMESTAMP_NTZ,
  customer_id STRING,
  account_id STRING,
  scenario_name STRING,
  alert_severity STRING,
  alert_score NUMBER(10,2),
  amount_usd NUMBER(18,2),
  country STRING,
  channel STRING,
  status STRING,
  assigned_analyst STRING,
  assigned_ts TIMESTAMP_NTZ,
  due_ts TIMESTAMP_NTZ,
  closed_ts TIMESTAMP_NTZ,
  disposition STRING,
  disposition_notes STRING,
  disposition_reason STRING,
  evidence_ids STRING,
  supervisor_approval BOOLEAN,
  supervisor_id STRING,
  supervisor_approved_ts TIMESTAMP_NTZ,
  last_updated_ts TIMESTAMP_NTZ
);
```

Create control parameters:

```sql
CREATE OR REPLACE TABLE control_parameters (
  control_id STRING,
  control_name STRING,
  sla_hours NUMBER(10,2),
  requires_notes BOOLEAN,
  requires_reason BOOLEAN,
  requires_evidence BOOLEAN,
  requires_supervisor_for_high BOOLEAN,
  effective_start_date DATE,
  effective_end_date DATE
);
```

Load minimal sample data (add your own realistic cases):

```sql
INSERT INTO control_parameters VALUES
('TM-001','Alert Disposition Completeness + Timeliness',48, TRUE, TRUE, TRUE, TRUE, '2025-01-01', NULL);
```

---

## 3) Load policies & create Cortex Search service

Create policy table:

```sql
CREATE OR REPLACE TABLE tm_policies (
  doc_id STRING,
  doc_title STRING,
  doc_type STRING,
  effective_date DATE,
  policy_text TEXT
);
```

Insert example policy text:

```sql
INSERT INTO tm_policies VALUES
('P-001','Transaction Monitoring Alert Disposition Procedure','Procedure','2025-01-01',
'All transaction monitoring alerts must be reviewed and dispositioned within 48 hours of alert creation. Analysts must document disposition notes, a disposition reason, and attach supporting evidence references. High severity alerts require supervisor approval prior to closure.');
```

Create the search service:

```sql
CREATE OR REPLACE CORTEX SEARCH SERVICE tm_policy_search
  ON policy_text
  ATTRIBUTES doc_title, doc_type, effective_date
  WAREHOUSE = tm_controls_wh
  TARGET_LAG = '1 hour'
AS (
  SELECT doc_id, doc_title, doc_type, effective_date, policy_text
  FROM tm_policies
);

GRANT USAGE ON CORTEX SEARCH SERVICE tm_policy_search TO ROLE tm_controls_role;
```

---

## 4) Cortex Analyst semantic model (`.yml`)

Create a semantic model that maps common questions to the alert population fields:
- counts by status/severity/scenario
- timeliness (hours to close) and SLA breach flag (48 hours)
- documentation completeness flags (notes/reason/evidence)
- approval requirement checks for high severity closures

Place your file in a stage, for example:

```sql
CREATE OR REPLACE STAGE models DIRECTORY = (ENABLE = TRUE);
GRANT READ ON STAGE models TO ROLE tm_controls_role;
```

Upload your semantic model file (via Snowsight UI or `PUT`), then reference it in Cortex Analyst.

---

## 5) Agent behavior (recommended routing)

**Routing rules**
- Structured questions (counts, lists, exceptions) → **Cortex Analyst**
- Policy/procedure questions (requirements, evidence standards) → **Cortex Search**
- Combined assessments → Search first (policy basis), then Analyst (population results)

**Example prompts**
- “For December 2025, what % of alerts closed within 48 hours? Break down by severity and scenario.”
- “List all alerts that are CLOSED but missing disposition notes, disposition reason, or evidence IDs.”
- “Which HIGH severity alerts were closed without supervisor approval?”
- “Based on policy, summarize whether the control is operating effectively and draft a workpaper conclusion.”

---

## Output format (workpaper-friendly)

A good copilot output should include:

1. **Scope & period**
2. **Policy basis** (short excerpt + document title)
3. **Test approach** (population definition + sampling if used)
4. **Results summary** (metrics + pass/fail assessment)
5. **Exceptions** (table with alert_id, issue type, severity, analyst)
6. **Conclusion & recommended follow-ups**

---

## Demo tips (what hiring managers like)

- Show **traceability**: the policy excerpt + the SQL logic
- Show **reproducibility**: same question → same population definition
- Show **governance**: RBAC, least privilege, and logged steps
- Keep it realistic: focus on 3–4 exception types and a clean conclusion

---

## License / Disclaimer

This is a demonstration project with synthetic data. It is not financial, legal, or compliance advice, and should not be used for production decisions without appropriate governance, validation, and approvals.
