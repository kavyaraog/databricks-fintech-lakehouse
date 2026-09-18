# Databricks Fintech Lakehouse Pipeline

Enterprise-grade Medallion Architecture pipeline built on **Azure Databricks** with **Unity Catalog** governance for fintech transaction processing and reconciliation.

## Architecture

Raw Data → Bronze (Delta) → Silver (DQ-validated) → Gold (Reconciled) → UC Governance


## Tech Stack

- **Platform:** Azure Databricks (Serverless compute)
- **Storage:** Delta Lake with Change Data Feed enabled
- **Governance:** Unity Catalog — column-level PII tagging, row-level security via dynamic views
- **Orchestration:** Databricks Notebooks (Git-backed)
- **Data Quality:** Deterministic per-record DQ flags with quarantine routing

## Pipeline Notebooks

| Notebook | Layer | Description |
|---|---|---|
| `01_bronze_ingestion` | Bronze | Auto Loader ingestion from ADLS Gen2 into raw Delta tables. Schema enforcement with `_rescued_data` capture. |
| `02_silver_transformation` | Silver | Type casting, normalization, 6 deterministic DQ flags per record, quarantine routing. 98.5% pass rate on 553 core banking records. |
| `03_gold_transactions` | Gold | LEFT JOIN reconciliation of core banking vs card authorization network. Composite key matching on `card_last_four + amount + transaction_date`. Partitioned by `transaction_date`. |
| `04_uc_governance` | Governance | PII column tagging via Unity Catalog system catalog, row-level security dynamic view, pipeline audit log. |

## Data Model

**Unity Catalog Namespace:** `fintech_lakehouse_dev.transactions`

| Table | Rows | Notes |
|---|---|---|
| `bronze_core_banking` | 553 | Raw strings, all columns varchar |
| `bronze_card_auth` | 270 | Raw strings, includes `_rescued_data` |
| `silver_core_banking` | 545 | DQ-passed, typed, 6 boolean flags |
| `silver_card_auth` | 270 | DQ-passed, typed |
| `gold_transactions` | 545 | Reconciled, partitioned by date |
| `gold_transactions_analyst_view` | — | Dynamic view with PII masking |
| `pipeline_audit_log` | — | Append-only run audit |

## Data Quality Framework

Six deterministic flags applied per record at the Silver layer:

- `dq_amount_invalid` — NULL or non-positive amount
- `dq_timestamp_invalid` — unparseable timestamp (ISO 8601 tolerant via `try_to_timestamp`)
- `dq_currency_invalid` — outside USD/EUR/GBP/CAD/MXN
- `dq_txn_type_invalid` — outside PURCHASE/TRANSFER/REFUND/WITHDRAWAL
- `dq_status_invalid` — outside APPROVED/PENDING/DECLINED
- `dq_card_format_invalid` — non-4-digit card suffix

## Reconciliation Results

| Status | Type | Count | Settled Amount |
|---|---|---|---|
| MISSING_AUTH | PURCHASE | 260 | $171,915.07 |
| NO_AUTH_EXPECTED | TRANSFER | 106 | $77,469.79 |
| NO_AUTH_EXPECTED | REFUND | 95 | $63,340.93 |
| NO_AUTH_EXPECTED | WITHDRAWAL | 84 | $48,294.01 |

## UC Governance

- **PII tagging:** `card_last_four` tagged `pii=true`, `sensitivity=confidential`, `data_class=payment_card` across silver and gold layers — tracked in `system.information_schema.column_tags`
- **Row-level security:** `gold_transactions_analyst_view` enforces group-based card masking via `is_account_group_member()`
- **Audit trail:** Full Delta table history via `DESCRIBE HISTORY` + append-only `pipeline_audit_log`

## Author

Kavya Gangadhara | Azure Databricks Engineer  
Certifications: Databricks Data Engineer Associate (in progress) | Gen AI Engineer Associate (Oct 2026)