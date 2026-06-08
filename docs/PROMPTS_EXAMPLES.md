# Prompt Examples & Agent Output Reference
## Master Data Agent — Internal Order (MDA-IO)

---

## Overview

This document captures representative prompt/response pairs for all supported scenarios. Each example shows the user prompt, the agent's parsed intent, the action taken, and the formatted output.

---

## Scenario 1: Create Internal Order

### Example 1.1 — Create with AI Suggestions

**User Prompt:**
```
Create a new Internal Order for the AI Transformation Program
```

**Agent Intent Parsed:** `CREATE_IO` — description provided, suggestion mode offered

**Agent Response:**
> I can help create a new Internal Order.
>
> To proceed, I need:
> - Order Description
> - Responsible Cost Center
> - Order Manager
> - Company Code
> - Budget Amount
> - Validity Period
> - Investment or Expense Type
>
> Would you like me to suggest values based on similar Internal Orders?

**Quick Reply Options:** `Yes, suggest values` | `No, I will enter manually`

---

**User Prompt:**
```
Yes, suggest values
```

**Agent Intent Parsed:** `SUGGEST_MODE = true` — queries historical IO data and org master

**Agent Response:**

> Based on historical Internal Orders and organizational data, here are my recommendations:

| Field | Recommended Value |
|---|---|
| Company Code | DE01 |
| Controlling Area | A100 |
| Order Type | Project Order |
| Responsible Cost Center | Global Innovation |
| Currency | EUR |
| Valid From | 01-Jul-2026 |
| Valid To | 31-Dec-2027 |

> I found the following potential order managers:
> - **Anna Mueller**
> - Michael Weber
> - John Smith
>
> Please select the responsible manager and confirm the budget.

**Quick Reply Options:** `Assign Anna Mueller and budget EUR 500,000` | `Assign Michael Weber and budget EUR 300,000`

---

**User Prompt:**
```
Assign Anna Mueller and budget EUR 500,000
```

**Agent Intent Parsed:** `MANAGER = Anna Mueller`, `BUDGET = EUR 500,000` — proceeds to review

**Agent Response:**

> Please review the Internal Order details before I proceed:

| Field | Value |
|---|---|
| Order Name | AI Transformation Program |
| Order Type | Project Order |
| Company Code | DE01 |
| Controlling Area | A100 |
| Responsible Cost Center | Global Innovation |
| Order Manager | Anna Mueller |
| Budget | EUR 500,000 |
| Valid From | 01-Jul-2026 |
| Valid To | 31-Dec-2027 |

> Would you like to continue?

**Quick Reply Options:** `Yes, continue` | `Edit details`

---

**User Prompt:**
```
Yes, continue
```

**Agent Action:** Runs 8 background validation checks

**Agent Response:**

> Running background validations…
>
> ✅ Mandatory field validation — passed
> ✅ Duplicate Internal Order check — no duplicates found
> ✅ Naming convention compliance — passed
> ✅ Cost Center validity (Global Innovation) — active and valid
> ✅ Manager assignment (Anna Mueller) — valid and active
> ✅ Budget threshold verification — within policy limits
> ✅ Corporate controlling policies — compliant
> ✅ Financial governance rules — compliant
>
> ✔ All validations passed successfully

---

**Agent Response (Authorization):**

> Authorization check complete.
>
> ✅ You are authorized to request Internal Order creation.
>
> ⚠️ **Approval required:** The budget of EUR 500,000 exceeds the EUR 250,000 auto-approval threshold. The following approval workflow has been triggered:

| Step | Status |
|---|---|
| Requester (Suman Puthadi) | ✅ Done |
| Cost Center Manager | ⏳ Pending |
| Finance Governance Team | ⏳ Pending |
| Master Data Team | ⏳ Pending |

**Quick Reply Options:** `Simulate Approval`

---

**User Prompt:**
```
Simulate Approval
```

**Agent Response:**

> ✅ **All approvals received.**

| Step | Status |
|---|---|
| Requester (Suman Puthadi) | ✅ Done |
| Cost Center Manager | ✅ Approved |
| Finance Governance Team | ✅ Approved |
| Master Data Team | ✅ Approved |

---

**Agent Response (Confirmation Gate):**

> As a governance safeguard, I require explicit confirmation before creating the Internal Order.
>
> **Internal Order Name:** AI Transformation Program
> **Budget:** EUR 500,000
> **Manager:** Anna Mueller
> **Valid From:** 01-Jul-2026
> **Valid To:** 31-Dec-2027
>
> 👉 Type **"Confirm Create"** to proceed.

**Quick Reply Options:** `Confirm Create`

---

**User Prompt:**
```
Confirm Create
```

**Agent Action:** Writes to S/4HANA via BAPI_INTERNALORDER_CREATE, updates MDR, logs audit entry

**Agent Response:**

> Creating Internal Order…
>
> ✅ Internal Order created in S/4HANA
> ✅ Cost Center assigned (Global Innovation)
> ✅ Responsible Manager assigned (Anna Mueller)
> ✅ Budget posted (EUR 500,000)
> ✅ Validity dates applied (01-Jul-2026 → 31-Dec-2027)
> ✅ Master Data Repository updated
> ✅ Connected systems synchronized

> **✅ IO-782145**
> Description: AI Transformation Program
> Budget: EUR 500,000
> Manager: Anna Mueller
> Status: 🟢 Active

> 📧 Notifications sent to Requestor, Cost Center Manager, and Finance Governance Team.

---

### Example 1.2 — Create with Manual Entry

**User Prompt:**
```
No, I will enter manually
```

**Agent Response:**

> Please fill in the Internal Order details below:

*[Form displayed with fields: Order Description, Company Code, Responsible Cost Center, Order Manager, Budget Amount (EUR), Valid From, Valid To, Order Type (dropdown)]*

---

## Scenario 2: Change Internal Order

### Example 2.1 — Budget Increase

**User Prompt:**
```
Change IO-782145 budget to EUR 750,000
```

**Agent Intent Parsed:** `CHANGE_IO`, `IO_ID = IO-782145`, `FIELD = budget`, `NEW_VALUE = EUR 750,000`

**Agent Response:**

> Retrieving current values for **IO-782145**…
>
> Impact analysis for budget change:

| Field | Current Value | Proposed Value |
|---|---|---|
| Budget | EUR 500,000 | EUR 750,000 |
| Δ Budget | | +EUR 250,000 (+50%) |
| Approval Threshold | EUR 250,000 | — |
| Approval Required | | Yes — Finance Governance |

> ✅ No open commitments exceed new budget
> ✅ Naming convention unchanged — no impact
> ✅ Controlling area policy — compliant
> ⚠️ Budget increase >50% requires additional approval
>
> Shall I trigger the approval workflow?

**Quick Reply Options:** `Yes, trigger approval` | `Cancel`

---

### Example 2.2 — Manager Change

**User Prompt:**
```
Change manager for IO-782145
```

**Agent Intent Parsed:** `CHANGE_IO`, `IO_ID = IO-782145`, `FIELD = manager`

**Agent Response:**

> Retrieving current values for **IO-782145**…
>
> Current manager: **Anna Mueller**
>
> Please select the new manager:

**Quick Reply Options:** `Assign Michael Weber` | `Assign John Smith`

---

## Scenario 3: Close Internal Order

### Example 3.1 — Standard Close

**User Prompt:**
```
Close IO-782145
```

**Agent Intent Parsed:** `CLOSE_IO`, `IO_ID = IO-782145`

**Agent Response:**

> Running pre-close checks for **IO-782145 — AI Transformation Program**…
>
> ✅ Open purchase orders — none found
> ✅ Open commitments — none found
> ✅ Open reservations — none found
> ✅ Settlement status — fully settled
> ✅ All actual costs posted and cleared
> ⚠️ Remaining budget: EUR 45,000 — will be released to Cost Center
> ✅ Governance sign-off — not required (budget < EUR 100,000 released)
>
> All pre-close checks passed. ✔ Order is eligible for closure
>
> Shall I proceed to the approval step?

**Quick Reply Options:** `Yes, proceed to approval` | `Cancel`

---

## Scenario 4: Query Internal Orders

### Example 4.1 — Query by Cost Center

**User Prompt:**
```
Show all active IOs for Cost Center Global Innovation
```

**Agent Intent Parsed:** `QUERY_IO`, `FILTER = cost_center:Global Innovation, status:ACTIVE`

**Agent Response:**

> Querying S/4HANA… Here are the active Internal Orders for **Cost Center: Global Innovation**:

| IO Number | Description | Budget | Consumed | Manager | Expiry | Status |
|---|---|---|---|---|---|---|
| IO-782145 | AI Transformation Program | EUR 500K | 65% | Anna Mueller | 31-Dec-2027 | 🟢 Active |
| IO-771032 | Digital Twin Initiative | EUR 300K | 42% | Michael Weber | 30-Jun-2027 | 🟢 Active |
| IO-765881 | Cloud Migration Phase 2 | EUR 1.2M | 89% | John Smith | 31-Mar-2027 | 🔴 Budget Risk |
| IO-759044 | Sustainability Analytics | EUR 180K | 31% | Anna Mueller | 31-Dec-2026 | 🟢 Active |
| IO-748832 | Innovation Lab Ops | EUR 75K | 97% | Michael Weber | 15-Jun-2026 | 🔴 Critical |

> 5 Internal Orders found.
> ⚠️ **2 orders require attention:** IO-765881 (budget risk) and IO-748832 (critical — expires in 13 days).

**Quick Reply Options:** `Show details for IO-765881` | `Show details for IO-748832` | `Export all to Excel`

---

### Example 4.2 — Drill Down to At-Risk Order

**User Prompt:**
```
Show details for IO-765881
```

**Agent Response:**

> **IO-765881 — Cloud Migration Phase 2**

| Field | Value |
|---|---|
| Company Code | DE01 |
| Controlling Area | A100 |
| Cost Center | Global Innovation |
| Manager | John Smith |
| Budget | EUR 1,200,000 |
| Actual Postings | EUR 1,068,000 (89%) |
| Remaining Budget | EUR 132,000 |
| Current Burn Rate | EUR 48,000 / month |
| Projected Depletion | ~47 days |
| Valid To | 31-Mar-2027 |
| Status | 🔴 Budget Risk |

> ⚠️ At the current burn rate of EUR 48,000/month, budget will be fully consumed by mid-July 2026 — well before the order expiry in March 2027.
>
> **Recommended actions:**
> - Request a budget extension of EUR 300,000
> - Reduce scope to stay within current budget
> - Reassign some activities to another IO

**Quick Reply Options:** `Request budget extension for IO-765881` | `Notify John Smith` | `Create new IO for overflow`

---

## Scenario 5: Advise Me

### Example 5.1 — Portfolio Optimization

**User Prompt:**
```
Which IOs should be closed this quarter?
```

**Agent Action:** Runs 5-dimension portfolio analysis across 45 orders

**Agent Response:**

> Analyzing your Internal Orders portfolio… Running 5 analysis dimensions:
>
> ✅ Posting activity analysis — complete (45 orders scanned)
> ✅ Budget utilization modeling — complete
> ✅ Expiry date risk assessment — complete
> ✅ Governance compliance check — complete
> ✅ Manager assignment validity — complete
>
> Analysis complete. Here are my recommendations:

**Agent Response (Recommendations):**

> Portfolio recommendations for Q3 2026:

| IO Number | Description | Recommendation | Priority | Reason |
|---|---|---|---|---|
| IO-755021 | Legacy ERP Cleanup | 🔒 Close | 🔴 High | No postings 180d; expires Jul-26 |
| IO-742300 | Pilot BI Project | 🔒 Close | 🔴 High | Fully settled; EUR 0 remaining |
| IO-748832 | Innovation Lab Ops | 🔒 Close or Extend | 🔴 Critical | Expires in 13 days; 97% consumed |
| IO-765881 | Cloud Migration Ph.2 | ⚠️ Budget Review | 🟠 Medium | 89% consumed; 47d to depletion |
| IO-759044 | Sustainability Analytics | 📋 Review Manager | 🟡 Low | Manager on leave — reassign |
| IO-771032 | Digital Twin Initiative | ✅ On Track | 🟢 None | Normal utilization and activity |
| IO-782145 | AI Transformation | ✅ On Track | 🟢 None | Normal utilization and activity |

> **Summary:** 3 orders for immediate action, 1 budget review, 1 governance item.
> Estimated budget release from closures: **EUR 52,250**

**Quick Reply Options:** `Close IO-755021 and IO-742300` | `Request budget review for IO-765881` | `Reassign IO-759044 manager` | `Take all recommended actions`

---

## Scenario 6: Proactive Alerts

### Example 6.1 — Budget Alert (triggered automatically)

**Alert Banner:**
> ⚠️ **Budget Alert:** IO-765881 has consumed 89% of budget. IO-782145 has consumed 85% of budget.

**Agent Message:**
> ⚠️ **Proactive Budget Alert — 2 orders require attention**

| IO Number | Description | Budget | Consumed | Remaining | Risk |
|---|---|---|---|---|---|
| IO-765881 | Cloud Migration Phase 2 | EUR 1.2M | 89% | EUR 132K | 🔴 Critical |
| IO-782145 | AI Transformation Program | EUR 500K | 85% | EUR 75K | 🟠 High |

> At current burn rates:
> - IO-765881 will be depleted in **47 days**
> - IO-782145 will be depleted in **62 days**

**Quick Reply Options:** `Request extension for IO-765881` | `Request extension for IO-782145` | `Notify managers` | `Dismiss`

---

### Example 6.2 — Governance Alert

**Alert Banner:**
> 🛡️ **Governance Alert:** IO-759044 has an invalid manager assignment.

**Agent Message:**
> 🛡️ **Governance Alert — Invalid Manager Assignment**
>
> **IO-759044 — Sustainability Analytics**
>
> The responsible manager **Sarah Hoffmann** is no longer valid:
> - Employee record shows: on extended leave since 01-May-2026
> - No substitute has been designated
> - This violates Corporate Controlling Policy CP-007
>
> Impact: Approval workflows for this order are blocked until a valid manager is assigned.
>
> Suggested replacement: **Anna Mueller** (same Cost Center, available capacity)

**Quick Reply Options:** `Assign Anna Mueller as manager` | `Assign a different manager` | `Escalate to Finance Governance`

---

## Prompt Design Patterns

### Pattern 1: Entity + Action
```
[Action Verb] [IO Identifier/Name] [Optional Field] [Optional Value]

Examples:
"Create a new Internal Order for Digital Twin Initiative"
"Change IO-782145 budget to EUR 750,000"
"Close IO-748832"
"Show all IOs managed by Anna Mueller"
```

### Pattern 2: Question Form
```
"Which IOs should be closed this quarter?"
"Where are the budget risks?"
"What is the status of IO-782145?"
"Show me IOs expiring in June"
```

### Pattern 3: Quick Reply (Guided)
```
[Agent presents options as buttons]
User clicks → exact text sent as input
Agent processes deterministically
```

### Pattern 4: Confirmation Phrases
```
"Confirm Create"  → triggers CREATE in S/4HANA
"Confirm Change"  → triggers CHANGE in S/4HANA
"Confirm Close"   → triggers CLOSE in S/4HANA
```

---

## Agent Response Component Library

| Component | Used For | Visual |
|---|---|---|
| `data-table` | Structured data, query results, comparisons | Zebra-striped table with blue header |
| `steps` | Sequential process results, validations | Vertical list with icons |
| `workflow` | Approval chain status | Node graph with dots |
| `confirm-box` | Pre-action review and confirmation | Dashed blue border box |
| `success-card` | Post-action success summary | Green gradient card |
| `quick-replies` | Guided navigation options | Pill buttons |
| `alert-banner` | Proactive system alerts | Colored top banner |
| `chip` | Status badges inline | Colored pill inline |
| `typing` | Agent processing indicator | 3-dot animation |
