# KPI Extraction & Dependency Discovery Framework

## Purpose

This framework is the metadata discovery engine behind the Celonis → Databricks → Power BI migration.

Instead of manually reading hundreds of KPIs, nested PQL formulas, and Celonis Knowledge Models, the framework automatically:

1. Extracts KPI definitions from Celonis.
2. Identifies KPI-to-KPI dependencies.
3. Identifies SAP tables used by each KPI.
4. Identifies activity/event tables used by each KPI.
5. Generates dependency graphs.
6. Produces lineage artifacts that can later be used for:

   * PQL → SQL migration
   * Validation
   * Impact analysis
   * Documentation
   * Agentic migration workflows

The output is a machine-readable dependency graph. 

---

# Environment Variables Required

## 1. Celonis URL

```env
CELONIS_BASE_URL=
```

### Why Needed

Used to connect to the Celonis tenant.

Example:

```text
https://abi-prd-eu-12.celonis.cloud
```

Without this:

* Knowledge Models cannot be read
* KPI metadata cannot be extracted
* Data Pool metadata cannot be discovered

---

## 2. Celonis API Token

```env
CELONIS_API_TOKEN=
```

### Why Needed

Authentication against Celonis APIs.

Used for:

* Knowledge Model extraction
* KPI extraction
* Data Pool metadata discovery
* Data Job metadata discovery

---

## 3. Team / Space Identifier

```env
TEAM_ID=
```

or

```env
SPACE_ID=
```

### Why Needed

Allows framework to know which Celonis workspace to query.

---

## 4. Knowledge Model ID

```env
KNOWLEDGE_MODEL_ID=
```

Example:

```text
Touchless OTC
Perfect Purchase
```

### Why Needed

Root source for KPI extraction.

Without KM ID:

Framework cannot discover:

* KPI formulas
* KPI hierarchy
* KPI metadata

---

## 5. Data Model ID

```env
DATA_MODEL_ID=
```

### Why Needed

Used to retrieve:

* Tables
* Columns
* PK/FK relationships

This becomes the table dependency graph.

---

## 6. Output Path

```env
OUTPUT_DIR=
```

Example:

```text
outputs/
```

Used to store:

```text
kpi_dependents.json

kpi_graph.dot

table_graph.dot

pql_metadata.json
```

---

# High-Level Workflow

```text
Celonis KM
      |
      ▼
KPI Extraction
      |
      ▼
Dependency Discovery
      |
      ▼
Graph Construction
      |
      ▼
Lineage JSON
      |
      ▼
Migration Engine
```

---

# Core Algorithm Design

The framework is not doing SQL generation.

Its primary responsibility is lineage extraction.

---

# Phase 1 — KPI Discovery

Framework scans:

```text
Knowledge Model
```

and extracts:

```json
{
  "kpi_id":"",
  "kpi_name":"",
  "formula":""
}
```

for every KPI.

---

# Phase 2 — KPI Dependency Detection

Example

```text
Touchless Order %
```

depends on:

```text
Order Creation %
Credit Check %
Quotation %
Order Changes %
```

The framework parses formulas.

Whenever KPI A references KPI B:

```text
A -> B
```

an edge is created.

Example from your JSON:

```text
kpi_touchless_order_percentage
    |
    +--> kpi_so_automation_percentage
    |
    +--> kpi_credit_check_percentage
    |
    +--> kpi_ftr_so_automation_percentage
```



---

# Phase 3 — Recursive Traversal

Many KPIs are nested.

Example:

```text
Touchless Order %

    |
    +--> Order Creation %

            |
            +--> Order Creation Flag
```

Framework recursively traverses all descendants.

---

# Why BFS / DFS is Used

Dependency graph is a DAG.

Example:

```text
Touchless Order
     |
     +----> Order Creation
     |           |
     |           +----> Order Creation Flag
     |
     +----> Credit Check
                 |
                 +----> Credit Check Flag
```

DFS helps discover complete dependency chains.

Example:

```text
Touchless Order
    →
Order Creation
    →
Order Creation Flag
```

---

# Phase 4 — Path Generation

Framework generates complete paths.

Example:

```text
kpi_touchless_order_percentage
 ->
kpi_so_automation_percentage
 ->
kpi_so_automation_flag
```

This becomes:

```json
"dependency_paths":[]
```

inside the output JSON. 

---

# Phase 5 — SAP Table Discovery

Framework scans KPI formulas.

Detects references:

```text
VBAK
VBAP
LIKP
LIPS
VBRK
VBRP
```

Stores them as:

```json
"sap_tables_used":[]
```

Example:

```json
{
 "kpi_name":"Touchless Order",
 "sap_tables_used":["VBAK"]
}
```



---

# Phase 6 — Activity Table Discovery

Framework detects event-log usage.

Examples:

```text
_CEL_MERGED_ACTIVITIES

SHIPMENT_ACTIVITIES

INVOICE_ACTIVITIES

SO_ACTIVITIES
```

Stored as:

```json
"activity_tables_used":[]
```



This is critical because activity tables drive:

* Process Mining
* Throughput Time
* Touchless KPIs
* Automation KPIs

---

# Phase 7 — Dependency Graph Generation

Creates:

```text
Node = KPI

Edge = Dependency
```

Example:

```text
Touchless Order %

    |
    +----> Order Creation %

    |
    +----> Credit Check %

    |
    +----> Quotation %
```

Output:

```text
kpi_graph.dot
```

Can be visualized in:

* Graphviz
* Draw.io
* Neo4j
* NetworkX

---

# Why This Matters for Migration

Without this framework:

Engineer manually reads:

```text
1000+ KPIs
```

With framework:

Engineer immediately sees:

```text
Root KPI

↓

Dependencies

↓

Tables

↓

Activities

↓

Lineage
```

---

# How Agentic Migration Uses This

The dependency graph becomes the input to the migration engine.

```text
Touchless Order
```

↓

Framework discovers

```text
Order Creation
Credit Check
Quotation
```

↓

Agent gets only relevant subtree

↓

PQL parsed

↓

PQL → SQL generated

↓

Validation script generated

↓

Gold KPI table generated

This is why the dependency extraction framework is actually the most important asset. The SQL generation part is comparatively easy; the difficult part is discovering the complete KPI lineage, table lineage, and activity lineage automatically so that migration, validation, and impact analysis can be done systematically.
