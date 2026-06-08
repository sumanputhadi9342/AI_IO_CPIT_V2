# API & Database Tables Reference
## Master Data Agent — Internal Order (MDA-IO)

---

## 1. SAP OData APIs (Production Integration)

### 1.1 Internal Order Management APIs

| API Endpoint | Method | Description | SAP Transaction Equivalent |
|---|---|---|---|
| `/sap/opu/odata/sap/CO_INTERNALORDER_SRV/InternalOrders` | GET | Retrieve list of Internal Orders with filters | S_ALR_87012993 |
| `/sap/opu/odata/sap/CO_INTERNALORDER_SRV/InternalOrders('{OrderId}')` | GET | Get single IO details | KO03 |
| `/sap/opu/odata/sap/CO_INTERNALORDER_SRV/InternalOrders` | POST | Create new Internal Order | KO01 |
| `/sap/opu/odata/sap/CO_INTERNALORDER_SRV/InternalOrders('{OrderId}')` | PATCH | Update IO fields (budget, manager, dates) | KO02 |
| `/sap/opu/odata/sap/CO_INTERNALORDER_SRV/InternalOrders('{OrderId}')/Close` | POST | Close/Lock Internal Order | KO04 |
| `/sap/opu/odata/sap/CO_INTERNALORDER_SRV/BudgetDetails('{OrderId}')` | GET | Get budget utilization and commitments | OKKS |

### 1.2 BAPI Reference (RFC-based integration alternative)

| BAPI Name | Purpose | Key Parameters |
|---|---|---|
| `BAPI_INTERNALORDER_CREATE` | Create Internal Order | ORDERTYPE, KTEXT, KOSTL, BUKRS, WAERS, DATUV, DATUB, VERANTWORTL |
| `BAPI_INTERNALORDER_CHANGE` | Change Internal Order fields | ORDERID, ORDERCHANGEIN (structure with changed fields) |
| `BAPI_INTERNALORDER_GETDETAIL` | Retrieve full IO details | ORDERID |
| `BAPI_INTERNALORDER_GETLIST` | Query IO list with filters | OBJECTLIST (multiple IOs) |
| `CO_OBJECT_CHECK_USAGE` | Pre-close usage check | OBJNR (object number) |

### 1.3 Request/Response Schemas

#### Create IO Request (POST)
```json
{
  "OrderType": "01",
  "Description": "AI Transformation Program",
  "CompanyCode": "DE01",
  "ControllingArea": "A100",
  "CostCenter": "GINNO",
  "ResponsiblePerson": "ANNAM",
  "Currency": "EUR",
  "ValidFrom": "2026-07-01",
  "ValidTo": "2027-12-31",
  "BudgetAmount": 500000.00,
  "ProfitCenter": "PC_INNOVATION"
}
```

#### Create IO Response (201 Created)
```json
{
  "OrderId": "IO-782145",
  "OrderNumber": "0000782145",
  "Description": "AI Transformation Program",
  "Status": "ACTIVE",
  "CreatedBy": "SPUTHADI",
  "CreatedAt": "2026-07-01T08:30:00Z",
  "ChangeDocument": "CD-2026-07-001234"
}
```

#### Change IO Request (PATCH)
```json
{
  "BudgetAmount": 750000.00,
  "ChangeReason": "Project scope expansion approved by Finance Governance",
  "ApprovalReference": "APPR-2026-0892"
}
```

#### Query IO List Request (GET with OData filters)
```
GET /InternalOrders?
  $filter=CostCenter eq 'GINNO' and Status eq 'ACTIVE'
  &$select=OrderId,Description,Budget,BudgetConsumed,Manager,ValidTo,Status
  &$orderby=ValidTo asc
  &$top=50
```

---

## 2. Core SAP Database Tables

### 2.1 AUFK — Order Master Data

| Field | Type | Description | Example |
|---|---|---|---|
| `MANDT` | CLNT(3) | Client | 100 |
| `AUFNR` | CHAR(12) | Order Number (Internal Order ID) | 0000782145 |
| `AUART` | CHAR(4) | Order Type | OI01 (Project Order) |
| `WERKS` | CHAR(4) | Plant | DE01 |
| `BUKRS` | CHAR(4) | Company Code | DE01 |
| `KOKRS` | CHAR(4) | Controlling Area | A100 |
| `KOSTL` | CHAR(10) | Cost Center | GINNO |
| `KTEXT` | CHAR(40) | Short Description | AI Transformation Program |
| `LTEXT` | CHAR(40) | Long Description | — |
| `VERANTWORTL` | CHAR(20) | Person Responsible (Manager) | ANNAM |
| `WAERS` | CUKY(5) | Currency | EUR |
| `DATUV` | DATS(8) | Valid From | 20260701 |
| `DATUB` | DATS(8) | Valid To | 20271231 |
| `ERDAT` | DATS(8) | Creation Date | 20260701 |
| `AEDAT` | DATS(8) | Last Changed Date | 20260715 |
| `ERNAM` | CHAR(12) | Created By | SPUTHADI |
| `AENAM` | CHAR(12) | Changed By | SPUTHADI |
| `SCOPE` | CHAR(1) | Order Category | 01 (Internal Order) |
| `OBJNR` | CHAR(22) | Object Number | OR0000782145 |
| `ISTAT` | CHAR(5) | System Status | REL (Released), CLSD (Closed) |
| `LOEKZ` | CHAR(1) | Deletion Flag | (blank) |

### 2.2 CSKS — Cost Center Master Data

| Field | Type | Description | Example |
|---|---|---|---|
| `MANDT` | CLNT(3) | Client | 100 |
| `KOKRS` | CHAR(4) | Controlling Area | A100 |
| `KOSTL` | CHAR(10) | Cost Center | GINNO |
| `DATBI` | DATS(8) | Valid To | 99991231 |
| `DATAB` | DATS(8) | Valid From | 20200101 |
| `MCTRT` | CHAR(2) | Record Type | — |
| `LOEVM` | CHAR(1) | Deletion Flag | (blank = active) |
| `VERAK` | CHAR(20) | Person Responsible | ANNAM |
| `BUKRS` | CHAR(4) | Company Code | DE01 |
| `KHINR` | CHAR(12) | Cost Center Group | INNOVATION |

### 2.3 CSKA — Cost Element Master Data

| Field | Type | Description | Example |
|---|---|---|---|
| `MANDT` | CLNT(3) | Client | 100 |
| `KOKRS` | CHAR(4) | Controlling Area | A100 |
| `KSTAR` | CHAR(10) | Cost Element | 0000430000 |
| `DATBI` | DATS(8) | Valid To | 99991231 |
| `DATAB` | DATS(8) | Valid From | 20200101 |
| `KATYP` | CHAR(1) | Cost Element Category | 1 (Primary), 3 (Allocated Activity) |

### 2.4 COBRA — Settlement Rule (Order)

| Field | Type | Description | Example |
|---|---|---|---|
| `MANDT` | CLNT(3) | Client | 100 |
| `AUFNR` | CHAR(12) | Order Number | 0000782145 |
| `OBJNR` | CHAR(22) | Object Number | OR0000782145 |
| `BUREG` | CHAR(2) | Distribution Rule | 001 |
| `EMPKO` | CHAR(10) | Settlement Receiver Cost Center | GINNO |
| `PROZS` | DEC(5,2) | Settlement Percentage | 100.00 |
| `ABTYP` | CHAR(3) | Settlement Type | FUL (Full) |

### 2.5 COSP / COSS — CO Object Cost Totals

| Field | Type | Description | Example |
|---|---|---|---|
| `MANDT` | CLNT(3) | Client | 100 |
| `OBJNR` | CHAR(22) | Object Number | OR0000782145 |
| `GJAHR` | NUMC(4) | Fiscal Year | 2026 |
| `WRTTP` | CHAR(2) | Value Type | 04 (Actuals), 01 (Budget) |
| `VERSN` | CHAR(3) | Version | 0 |
| `KSTAR` | CHAR(10) | Cost Element | 0000430000 |
| `WKG001` — `WKG016` | CURR(15,2) | Period values (Jan–Dec) | 48000.00 |
| `WKG000` | CURR(15,2) | Annual total | 576000.00 |

### 2.6 AUFNR — Budget Table (via BPGE / BPJA)

| Field | Type | Description | Example |
|---|---|---|---|
| `MANDT` | CLNT(3) | Client | 100 |
| `OBJNR` | CHAR(22) | Object Number | OR0000782145 |
| `LEDNR` | CHAR(2) | Ledger | 01 |
| `GJAHR` | NUMC(4) | Fiscal Year | 2026 |
| `WRTTP` | CHAR(2) | Value Type | 41 (Budget Original) |
| `WTGES` | CURR(15,2) | Total Budget Amount | 500000.00 |

---

## 3. Agent Application Data Store (Master Data Repository)

### 3.1 io_master — Internal Order Snapshot Table

| Column | Type | Description | Example |
|---|---|---|---|
| `io_id` | VARCHAR(12) | Internal Order Number | IO-782145 |
| `aufnr` | VARCHAR(12) | SAP AUFNR | 0000782145 |
| `description` | VARCHAR(100) | Order Description | AI Transformation Program |
| `order_type` | VARCHAR(20) | Order Type | Project Order |
| `company_code` | CHAR(4) | Company Code | DE01 |
| `controlling_area` | CHAR(4) | Controlling Area | A100 |
| `cost_center` | VARCHAR(10) | Cost Center | GINNO |
| `manager` | VARCHAR(50) | Responsible Manager | Anna Mueller |
| `manager_user_id` | VARCHAR(12) | SAP User ID | ANNAM |
| `currency` | CHAR(3) | Currency | EUR |
| `budget_original` | DECIMAL(15,2) | Original Budget | 500000.00 |
| `budget_current` | DECIMAL(15,2) | Current Budget (after changes) | 750000.00 |
| `budget_consumed` | DECIMAL(15,2) | Actual Postings to Date | 487500.00 |
| `budget_pct` | DECIMAL(5,2) | Budget % Consumed | 65.00 |
| `valid_from` | DATE | Valid From | 2026-07-01 |
| `valid_to` | DATE | Valid To | 2027-12-31 |
| `status` | VARCHAR(10) | Status | ACTIVE / CLOSED |
| `last_posting_date` | DATE | Last Financial Posting | 2026-06-01 |
| `created_by` | VARCHAR(50) | Created By (agent user) | Suman Puthadi |
| `created_at` | TIMESTAMP | Creation Timestamp | 2026-07-01T08:30:00Z |
| `updated_at` | TIMESTAMP | Last Updated | 2026-07-15T14:22:00Z |

### 3.2 io_audit_log — Audit Trail Table

| Column | Type | Description | Example |
|---|---|---|---|
| `log_id` | UUID | Unique Log ID | uuid-v4 |
| `action` | VARCHAR(10) | Action Type | CREATE / CHANGE / CLOSE / QUERY |
| `io_id` | VARCHAR(12) | Internal Order ID | IO-782145 |
| `object_type` | VARCHAR(30) | Object Description | Internal Order |
| `old_value` | JSONB | Previous Values (for change) | `{"budget": 500000}` |
| `new_value` | JSONB | New Values | `{"budget": 750000}` |
| `user_id` | VARCHAR(50) | Initiating User | Suman Puthadi |
| `user_sap_id` | VARCHAR(12) | SAP User ID | SPUTHADI |
| `source` | VARCHAR(50) | Source System | Master Data Agent - IO |
| `approval_ref` | VARCHAR(50) | Approval Reference | APPR-2026-0892 |
| `timestamp` | TIMESTAMP | Action Timestamp | 2026-07-01T08:30:00Z |
| `ip_address` | VARCHAR(45) | Client IP | 10.20.30.40 |
| `session_id` | VARCHAR(100) | Browser Session | sess_abc123 |

### 3.3 io_alerts — Alert Configuration & History Table

| Column | Type | Description | Example |
|---|---|---|---|
| `alert_id` | UUID | Unique Alert ID | uuid-v4 |
| `io_id` | VARCHAR(12) | Internal Order ID | IO-782145 |
| `alert_type` | VARCHAR(20) | Type | BUDGET / EXPIRY / ACTIVITY / GOVERNANCE |
| `threshold_pct` | DECIMAL(5,2) | Trigger Threshold | 75.00 |
| `threshold_days` | INT | Days Threshold | 30 |
| `triggered_at` | TIMESTAMP | When Alert Fired | 2026-06-15T06:00:00Z |
| `acknowledged_by` | VARCHAR(50) | User who dismissed | Suman Puthadi |
| `acknowledged_at` | TIMESTAMP | Dismissal Time | 2026-06-15T09:45:00Z |
| `action_taken` | VARCHAR(100) | Result Action | Budget extension requested |
| `status` | VARCHAR(10) | OPEN / ACKNOWLEDGED / RESOLVED | RESOLVED |

### 3.4 io_approvals — Approval Workflow Table

| Column | Type | Description | Example |
|---|---|---|---|
| `approval_id` | UUID | Unique Approval ID | uuid-v4 |
| `io_id` | VARCHAR(12) | Internal Order ID | IO-782145 |
| `action_type` | VARCHAR(10) | Action being approved | CREATE / CHANGE / CLOSE |
| `approver_role` | VARCHAR(50) | Role of Approver | Finance Governance Team |
| `approver_user` | VARCHAR(50) | Approver Name | Karl Fischer |
| `approver_sap_id` | VARCHAR(12) | SAP User ID | KFISCH |
| `status` | VARCHAR(10) | PENDING / APPROVED / REJECTED | APPROVED |
| `requested_at` | TIMESTAMP | When sent for approval | 2026-07-01T08:35:00Z |
| `actioned_at` | TIMESTAMP | When approved/rejected | 2026-07-01T10:15:00Z |
| `comments` | TEXT | Approver comments | Approved per Q3 budget allocation |
| `sequence` | INT | Approval chain order | 2 |

---

## 4. Alert Trigger Thresholds (Configuration)

| Alert Type | Trigger Condition | Frequency | Recipients |
|---|---|---|---|
| Budget Alert (Warning) | Consumed ≥ 75% of budget | Daily check | IO Manager, Cost Center Manager |
| Budget Alert (Critical) | Consumed ≥ 90% of budget | Daily check | IO Manager, Cost Center Mgr, Finance Governance |
| Expiry Alert (Warning) | Valid-to within 30 days | Daily check | IO Manager, Cost Center Manager |
| Expiry Alert (Critical) | Valid-to within 7 days | Daily check | IO Manager, Cost Center Mgr, Master Data Team |
| Activity Alert | No postings for 90 consecutive days | Weekly check | IO Manager, Controlling Team |
| Governance Alert | Manager employee record inactive/on-leave | Real-time | IO Manager, Finance Governance, Master Data Team |
