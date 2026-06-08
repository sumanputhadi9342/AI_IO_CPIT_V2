# Supported Actions — Master Data Agent (Internal Order)
## Complete Reference of All Agent Capabilities

---

## Action Summary

| Action | Trigger | SAP Equivalent | Approval Required | Confirmation Gate | Audit Logged |
|---|---|---|---|---|---|
| **Create IO** | User request | KO01 | Yes (if budget > EUR 250K) | ✅ "Confirm Create" | ✅ |
| **Change IO** | User request | KO02 | Yes (if budget increase > 50%) | ✅ "Confirm Change" | ✅ |
| **Close IO** | User request | KO04 | Yes (Cost Center Manager) | ✅ "Confirm Close" | ✅ |
| **Query IO** | User request | KO03 / S_ALR | No | No | No (read-only) |
| **Advise / Analyze** | User request | None (AI) | No | No | No (read-only) |
| **Budget Alert** | System trigger (≥75%/90%) | None | No | No | No |
| **Expiry Alert** | System trigger (≤30 days) | None | No | No | No |
| **Activity Alert** | System trigger (≥90d no posts) | None | No | No | No |
| **Governance Alert** | System trigger (invalid manager) | None | No | No | No |
| **Bulk Action** | User request (from Advise) | Multiple | Yes | ✅ Per action | ✅ |

---

## Action 1: Create Internal Order

### Description
Creates a new Internal Order in SAP S/4HANA with full governance, validation, and approval.

### Supported Input Patterns
```
"Create a new Internal Order for [description]"
"Set up an IO for [project name]"
"I need a new internal order"
"New IO for [cost center] — [description]"
```

### Data Collected
| Field | Required | Source |
|---|---|---|
| Order Description | Yes | User input / AI suggestion |
| Company Code | Yes | User input / AI suggestion (default: DE01) |
| Controlling Area | Yes | Derived from company code (default: A100) |
| Order Type | Yes | User selection / AI suggestion |
| Responsible Cost Center | Yes | User input / AI suggestion |
| Order Manager | Yes | User selection from AI-suggested list |
| Budget Amount (EUR) | Yes | User input / AI suggestion |
| Valid From | Yes | User input / AI suggestion |
| Valid To | Yes | User input / AI suggestion |
| Currency | Yes | Derived from company code (default: EUR) |

### Validation Checks (8 total)
1. Mandatory field completeness
2. Duplicate IO detection (same description + company code + order type)
3. Naming convention compliance (corporate naming policy)
4. Cost Center active and valid for the specified date range
5. Manager is an active SAP employee (no deletion flag, not on extended leave)
6. Budget within controlling area threshold
7. Corporate Controlling Policy CP-007 compliance
8. Financial governance rules (period open, user authorization)

### Approval Routing
| Condition | Approvers |
|---|---|
| Budget ≤ EUR 250,000 | Cost Center Manager → Master Data Team |
| Budget > EUR 250,000 | Cost Center Manager → Finance Governance Team → Master Data Team |

### Confirmation Phrase
> Type **"Confirm Create"** to execute

### Post-Creation Actions
- IO record created in S/4HANA (AUFK table)
- Budget posted to order
- Master Data Repository updated
- Connected systems synchronized
- Audit log entry created
- Email notifications sent to: Requester, Cost Center Manager, Finance Governance Team
- Monitoring activated (budget 75%/90%, expiry -30d, activity 90d, governance)

### S/4HANA Integration
- **BAPI:** `BAPI_INTERNALORDER_CREATE`
- **Transaction Equivalent:** KO01
- **Tables Written:** AUFK, BPGE/BPJA (budget)

---

## Action 2: Change Internal Order

### Description
Updates one or more fields of an existing Internal Order with impact analysis and governed approval.

### Supported Input Patterns
```
"Change IO-[number] budget to [amount]"
"Increase budget for IO-[number] by [amount]"
"Change manager for IO-[number] to [name]"
"Extend validity of IO-[number] by [period]"
"Update description of IO-[number]"
```

### Changeable Fields
| Field | Impact Analysis | Approval Trigger |
|---|---|---|
| Budget Amount | Δ% calculation, commitment check | Budget increase > 50% or > EUR 250K |
| Order Manager | Active employee check | Always requires Cost Center Manager approval |
| Valid From / Valid To | Fiscal period check | None (notification only) |
| Order Description | Naming convention check | None |
| Responsible Cost Center | Commitment re-assignment check | Finance Governance approval required |

### Validation Checks
1. IO exists and is in ACTIVE status (not already closed)
2. User is authorized to request changes (CO_AUFNR activity 02)
3. No open commitments that would be violated by the change
4. New values pass naming / policy rules
5. Manager validity (if changing manager)
6. Fiscal period open (if changing budget)

### Approval Routing
| Condition | Approvers |
|---|---|
| Budget increase ≤ 50% | Cost Center Manager |
| Budget increase > 50% | Cost Center Manager + Finance Governance Team |
| Manager change | Cost Center Manager |
| Validity date change | None |
| Cost Center change | Finance Governance Team + Master Data Team |

### Confirmation Phrase
> Type **"Confirm Change"** to execute

### Post-Change Actions
- IO record updated in S/4HANA
- Change document created (for audit trail)
- Master Data Repository synchronized
- Budget availability recalculated
- Connected systems notified
- Monitoring thresholds updated (e.g., budget alert threshold recalculated)
- Email notifications sent

### S/4HANA Integration
- **BAPI:** `BAPI_INTERNALORDER_CHANGE`
- **Transaction Equivalent:** KO02
- **Tables Updated:** AUFK, COSP/COSS (if budget change)

---

## Action 3: Close Internal Order

### Description
Permanently locks an Internal Order against further postings, releases remaining budget, and archives the order.

### Supported Input Patterns
```
"Close IO-[number]"
"Lock IO-[number]"
"Shut down IO-[number]"
"Archive and close [description] order"
```

### Pre-Close Checks (7 total)
| Check | Expected Result |
|---|---|
| Open Purchase Orders | None — must be cleared before close |
| Open Commitments | None — must be settled |
| Open Reservations | None — must be cancelled |
| Settlement Status | Fully settled |
| Actual Costs | All posted and cleared |
| Remaining Budget | Flagged — will be released to Cost Center |
| Governance Sign-off | Required if remaining budget > EUR 100,000 |

### Approval Routing
| Condition | Approvers |
|---|---|
| All cases | Requester → Cost Center Manager → Master Data Team |
| Remaining budget > EUR 100,000 | + Finance Governance Team |

### ⚠️ Irreversibility Warning
The agent explicitly warns: **"This action is irreversible."** User must type the exact phrase to proceed.

### Confirmation Phrase
> Type **"Confirm Close"** to execute

### Post-Close Actions
- IO status set to CLOSED (ISTAT = CLSD) in S/4HANA
- Remaining budget released to the responsible Cost Center
- IO locked against further financial postings
- Master Data Repository updated
- Change document created for audit trail
- Connected systems synchronized
- Email notifications sent to: Requester, Cost Center Manager, Finance Governance Team
- Monitoring deactivated for this order

### S/4HANA Integration
- **Transaction Equivalent:** KO04 (lock), with status change
- **Tables Updated:** AUFK (ISTAT field), BPGE/BPJA (budget release)

---

## Action 4: Query Internal Orders

### Description
Searches and retrieves Internal Orders from S/4HANA with filtering, sorting, and drill-down capabilities.

### Supported Input Patterns
```
"Show all active IOs for Cost Center [name]"
"Show IOs managed by [manager name]"
"Show IOs expiring this quarter"
"Show IOs with budget consumption above 80%"
"Find IO-[number]"
"List all closed IOs in [company code]"
```

### Supported Filter Dimensions
| Dimension | Values |
|---|---|
| Cost Center | Name or code |
| Manager / Person Responsible | Name or user ID |
| Order Status | ACTIVE, CLOSED, LOCKED |
| Order Type | Project Order, Investment Order, Expense Order, Overhead Order |
| Budget Utilization | Range (e.g., > 80%) |
| Expiry Date | Specific date, relative period ("this quarter", "next 30 days") |
| Company Code | DE01, etc. |
| IO Number | Direct lookup |

### Output Components
- **Summary Table:** IO Number, Description, Budget, % Consumed, Manager, Expiry, Status
- **Risk Indicators:** 🟢 Active / 🟠 Warning / 🔴 Critical / 🔴 Budget Risk
- **Drill-Down Detail:** Full IO fields + burn rate projection + days to depletion
- **Recommended Actions:** Inline action buttons from query results
- **Count Summary:** "5 Internal Orders found. 2 require attention."

### S/4HANA Integration
- **OData:** `GET /InternalOrders` with `$filter`, `$select`, `$orderby`
- **Transaction Equivalent:** KO03 (display) / S_ALR_87012993 (report)
- **Tables Read:** AUFK, COSP/COSS, BPGE/BPJA, CSKS

---

## Action 5: Advise Me (Portfolio Analysis)

### Description
Analyzes the full Internal Order portfolio across 5 dimensions and returns prioritized recommendations with actionable next steps.

### Supported Input Patterns
```
"Which IOs should be closed this quarter?"
"Where are the budget risks?"
"Show portfolio overview"
"What needs my attention?"
"Analyze all IOs for Global Innovation"
"Give me a health check of all active orders"
```

### Analysis Dimensions
| Dimension | What It Checks |
|---|---|
| 1. Posting Activity | Orders with no postings for 90+ days |
| 2. Budget Utilization | Orders consuming budget at risk pace |
| 3. Expiry Risk | Orders expiring within 30/60/90 days |
| 4. Governance Compliance | Policy violations (CP-007, naming, manager validity) |
| 5. Manager Assignment Validity | Active employee check for all assigned managers |

### Recommendation Priorities
| Priority | Label | Definition |
|---|---|---|
| 🔴 Critical | Immediate action | Order expiring in <14 days or >95% consumed |
| 🔴 High | Urgent | No activity 180d, or fully settled with open budget |
| 🟠 Medium | Review needed | Budget >85% consumed, projected depletion <60 days |
| 🟡 Low | Monitor | Governance item not yet blocking workflows |
| 🟢 None | On Track | Normal utilization, active manager, within validity |

### Recommendation Types
| Type | Description |
|---|---|
| 🔒 Close | Order meets all criteria for immediate closure |
| 🔒 Close or Extend | Borderline — needs decision from manager |
| ⚠️ Budget Review | Budget extension request recommended |
| 📋 Review Manager | Manager reassignment needed |
| ✅ On Track | No action required |

### Bulk Action Support
From the Advise view, users can:
- Act on one recommendation at a time
- Initiate all recommended actions simultaneously
- Agent queues all approval workflows in parallel
- Estimated budget release shown upfront

---

## Action 6: Budget Alerts (Proactive)

### Description
Automatically surfaced alert when one or more IOs reach the 75% or 90% budget consumption threshold.

### Trigger Conditions
| Alert Level | Threshold | Frequency |
|---|---|---|
| Warning | ≥ 75% budget consumed | Daily background check |
| Critical | ≥ 90% budget consumed | Daily background check |

### Alert Output
- Orange alert banner across top of interface
- Full alert message with table of affected orders
- Burn rate projection ("depleted in X days")
- Inline action buttons: Request Extension, Notify Manager, Dismiss

### Available Actions from Alert
- Initiate budget extension request (triggers Change IO workflow)
- Send notification to IO manager
- Dismiss alert (acknowledged state logged)

---

## Action 7: Expiry Alerts (Proactive)

### Description
Automatically surfaced alert when one or more IOs are approaching their Valid To date.

### Trigger Conditions
| Alert Level | Threshold | Frequency |
|---|---|---|
| Warning | ≤ 30 days to expiry | Daily background check |
| Critical | ≤ 7 days to expiry | Daily background check |

### Alert Output
- Blue info banner
- Table of expiring orders with days remaining
- Recommended actions (close vs. extend) per order

### Available Actions from Alert
- Close the expiring IO
- Extend validity (triggers Change IO workflow)
- Notify the responsible manager

---

## Action 8: Activity Alerts (Proactive)

### Description
Flags Internal Orders with no financial postings for an extended period — potential zombie orders.

### Trigger Conditions
| Threshold | Frequency |
|---|---|
| No postings for 90+ consecutive days | Weekly check |
| No postings for 180+ days | Immediate flag, escalated priority |

### Alert Output
- Warning banner
- Full detail: last posting date, remaining budget, manager contact
- Root cause hypothesis: project complete, abandoned, wrong cost object

### Available Actions from Alert
- Close the order
- Notify responsible manager
- Ignore / dismiss with comment

---

## Action 9: Governance Alerts (Proactive)

### Description
Detects and surfaces violations of Corporate Controlling Policy CP-007 and other governance rules.

### Monitored Governance Rules
| Rule | Trigger |
|---|---|
| CP-007: Valid Manager | Manager employee record inactive, terminated, or on extended leave |
| CP-007: Active Cost Center | Assigned Cost Center marked for deletion or deactivated |
| Naming Convention | IO description no longer matches corporate naming standards |
| Duplicate Detection | New order created with matching description to existing active order |

### Alert Output
- Warning banner
- Full explanation of violation and policy reference
- Impact statement (e.g., "Approval workflows are blocked")
- Suggested replacement / remediation with recommended action

### Available Actions from Alert
- Reassign manager directly from alert
- Escalate to Finance Governance Team
- Update Cost Center assignment

---

## Agent Non-Actions (Boundaries)

The following actions are intentionally **not supported** in v1.0:

| Action | Reason | Planned Version |
|---|---|---|
| IO Settlement / KO88 | Complexity requires dedicated settlement agent | v2.0 |
| Statistical Internal Orders | Different data model — separate scope | v1.1 |
| Mass creation (file upload) | UI/UX complexity — manual or API for bulk | v1.1 |
| Cross-company code transfers | Regulatory complexity | v2.0 |
| IO deletion | Destructive — not allowed per audit policy | Never (archive only) |
| Authorization role assignment | Separate identity governance domain | Out of scope |
