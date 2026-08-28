# data-quality-sla-monitor
Contract-driven data quality SLA monitor built on Databricks and PySpark. Quality rules stored as data, evaluated by one engine, tracked in an append-only history table, and surfaced in a live monitoring dashboard. Proven on real sample data by catching four injected defects.

# Data Quality SLA Monitor (Databricks)

A contract driven data quality monitor built on Azure Databricks and PySpark, applied to real
shared data from the Databricks `samples` catalog. Quality rules are stored as data, evaluated
by a single extensible engine, and written to an append only history table that powers a live
AI/BI dashboard.

It answers the question every data consumer has each morning: **is this data good enough to
use, and if not, what exactly is wrong with it?**

## Why this exists

Most data quality checks are scattered across pipelines as one off assertions that are hard to
find, hard to change, and silent when they pass. This project takes a different approach:

1. **The contract is data, not code.** Every rule lives as a row in a `sla_contract` table (the
   check, the column, the threshold, the severity). Adding or tightening a rule is a data edit,
   not a code change.
2. **One engine runs every check.** A single dispatch function handles each check type, so a new
   kind of rule is a few lines, not a new script.
3. **Results are history, not a snapshot.** Every run is appended to a `sla_results` table, which
   is what makes trends, pass rate over time, and recurring problem detection possible.

## What it checks

Eight rules across three data quality dimensions, on `samples.bakehouse.sales_transactions`:

| Dimension | Rule | What it catches |
|-----------|------|-----------------|
| Completeness | row count, null rates | missing rows, missing values on key columns |
| Uniqueness | duplicate keys | a transaction ID that is not actually unique |
| Validity | non positive values, reconciliation | a negative quantity, or `totalPrice` that does not equal `quantity * unitPrice` |

Rules that would corrupt every downstream financial number (null transaction IDs, duplicate
keys, broken price reconciliation) are marked **SEV1**.

## Proof it actually catches problems

A monitor that always shows green proves nothing. To demonstrate real detection, I built a
deliberately corrupted copy of the data (never touching the shared source) and pointed the same
engine at it. It caught **four breaches**, all correctly:

| Rule | Dimension | Result |
|------|-----------|--------|
| `null_txn` | completeness | BREACH (SEV1) |
| `dup_txn` | uniqueness | BREACH (SEV1) |
| `total_recon` | validity | BREACH (SEV1) |
| `bad_qty` | validity | BREACH (SEV2) |

The detail worth noting: a single injected defect (nulling out around 1% of transaction IDs)
surfaced across **two dimensions at once**. The nulls tripped the completeness check, and because
the nulled IDs collapsed into a shared null value they also tripped the uniqueness check. One
upstream issue, two failing rules. That is exactly how quality problems cascade in real systems.

Everything the monitor did not break stayed green. No false positives.

## Architecture

```
samples.bakehouse.sales_transactions   (real shared source, read only)
                |
                v
        evaluation engine  <---  sla_contract   (rules stored as data)
                |
                v
          sla_results        (append only run history)
                |
                v
        AI/BI dashboard      (KPIs, on call status board, trend)
```

The monitor is read only over the source and writes only its own contract and results tables, so
the same logic is safe to point at production data.

## Dashboard

The published dashboard reads from `sla_results` and shows:

- **Pass rate %** as the headline KPI
- **Breach count** and **SEV1 count**
- **Status board**: every rule, its threshold, the real observed value, and its status, with
  breaches highlighted in red
- **Pass rate over time**, which fills in as the monitor runs repeatedly

![dashboard](docs/dashboard.png)

## How to run

1. Import `sla_monitor.py` into Databricks (Workspace, Import, File). The `# COMMAND` markers
   split it into cells.
2. It reads from the shared `samples` catalog and writes to `workspace.sla_monitor`. Change the
   `CATALOG` variable if your writable catalog is named differently.
3. Run the cells in order. The steps are: verify access, profile the data, declare the contract,
   evaluate, persist history, and prove detection on a corrupted copy.

**Requirements:** a Unity Catalog enabled Databricks workspace (Free Edition works). No external
libraries.

## Built with

Databricks, PySpark, Delta Lake, Unity Catalog, Databricks AI/BI dashboards, medallion
architecture.

## Notes on the data

The `samples` data is static, so time based freshness is not meaningful here and the monitor
focuses on completeness, uniqueness, and validity, which are real and testable on this data. The
same engine gains freshness SLAs the moment it is pointed at a table that actually refreshes.
