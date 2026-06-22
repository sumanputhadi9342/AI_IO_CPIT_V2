# Software Design Document (SDD)
## Master Data Agent — Internal Order (MDA-IO)

| Attribute | Value |
|---|---|
| Document Version | 1.0 |
| Date | June 2026 |
| Author | Suman Puthadi |
| System | SAP S/4HANA · Company Code DE01 · Controlling Area A100 |
| Platform | Single-page HTML/JS application — AI conversational agent |

---

## Table of Contents

1. [Introduction](#1-introduction)
2. [System Overview](#2-system-overview)
3. [Architecture](#3-architecture)
4. [Data Model](#4-data-model)
5. [Scenario Design — Create IO](#5-scenario-design--create-io)
6. [Scenario Design — Change IO](#6-scenario-design--change-io)
7. [Scenario Design — Close IO](#7-scenario-design--close-io)
8. [Scenario Design — Query IO](#8-scenario-design--query-io)
9. [Scenario Design — Advise Me](#9-scenario-design--advise-me)
10. [Scenario Design — Mass Change IO](#10-scenario-design--mass-change-io)
11. [Validation Framework](#11-validation-framework)
12. [Approval Workflow Engine](#12-approval-workflow-engine)
13. [Alert & Monitoring System](#13-alert--monitoring-system)
14. [UI Component Library](#14-ui-component-library)
15. [Mass Change Template — Upload/Download](#15-mass-change-template--uploaddownload)
16. [Audit Trail](#16-audit-trail)
17. [SAP Integration Layer](#17-sap-integration-layer)
18. [Security & Governance](#18-security--governance)
19. [Non-Functional Requirements](#19-non-functional-requirements)

---

## 1. Introduction

### 1.1 Purpose
This Software Design Document describes the design, architecture, and functional behaviour of the **Master Data Agent — Internal Order (MDA-IO)** application. It serves as the definitive technical reference for developers, architects, and SAP consultants maintaining or extending the system.

### 1.2 Scope
MDA-IO is an AI-powered conversational agent embedded in SAP S/4HANA for managing Internal Order (IO) master data. It supports the full lifecycle: creation, change, closure, query, portfolio advisory, and mass change — all governed by approval workflows and audit controls.

### 1.3 Definitions

| Term | Definition |
|---|---|
| IO | Internal Order (SAP object type KO — cost collecting object) |
| MDA-IO | Master Data Agent — Internal Order |
| Joule | SAP's AI assistant branding used as the UX paradigm |
| Scenario | A discrete conversational workflow (Create, Change, Close, Query, Advise, Mass Change) |
| Step | A single turn within a scenario — one agent response + one user input |
| Match Guard | A regex on each step that validates user input before advancing |
| Before-Hook | A function that extracts data from user input before advancing to the next step |
| Confirmation Gate | A hard-coded phrase check (e.g. "Confirm Create") required before write operations |
| BAPI | Business Application Programming Interface — SAP function module for system integration |

### 1.4 References

| Document | Location |
|---|---|
| Product Requirements Document | `docs/PRD.md` |
| Actions Reference | `docs/ACTIONS.md` |
| Technical Flow | `docs/TECHNICAL_FLOW.md` |
| API & Table Reference | `docs/API_TABLES.md` |
| Agentic Architecture Details | `docs/AGENTIC_DETAILS.md` |
| Prompt Examples | `docs/PROMPTS_EXAMPLES.md` |
| Business Story | `docs/BUSINESS_STORY.md` |
| Flow Diagrams | `docs/FLOW_DIAGRAM.md` |

---

## 2. System Overview

MDA-IO is a **single-page application (SPA)** delivered as a standalone HTML file (`index.html`). It requires no backend server — all logic is embedded in client-side JavaScript. In production, the agent interfaces with SAP S/4HANA via OData services and BAPIs (mocked in the current prototype with simulated responses).

### 2.1 Key Capabilities

| Capability | Description |
|---|---|
| Create IO | 10-step guided workflow with AI suggestions or manual form entry |
| Change IO | 7-step workflow with impact analysis and threshold-based approval routing |
| Close IO | 7-step workflow with pre-close checks and irreversibility safeguard |
| Query IO | Multi-dimensional search with drill-down and burn rate analysis |
| Advise Me | 8 specialised portfolio analysis topics with actionable recommendations |
| Mass Change IO | Bulk field update across multiple IOs via checkbox selection or CSV upload |
| Proactive Alerts | Budget, expiry, activity, and governance alerts with sidebar badges |
| Audit Trail | Every write operation logged with action, object, ID, timestamp, and user |

### 2.2 User Personas

| Persona | Role | Primary Use Cases |
|---|---|---|
| Suman Puthadi | Requestor / Master Data Administrator | Create, Change, Mass Change |
| Cost Center Manager | Approver | Approval simulation, Query |
| Finance Governance Team | Approver | Budget review, Governance alerts |
| Master Data Team | Final approver | Create, Change, Close sign-off |
| Portfolio Manager | Analyst | Advise Me, Query |

---

## 3. Architecture

### 3.1 Application Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                    index.html (SPA)                         │
│                                                             │
│  ┌──────────────┐  ┌──────────────────┐  ┌──────────────┐  │
│  │   HTML/CSS   │  │  JavaScript       │  │  Embedded    │  │
│  │   UI Shell   │  │  Scenario Engine  │  │  State Store │  │
│  └──────────────┘  └──────────────────┘  └──────────────┘  │
│                            │                                │
│              ┌─────────────┼─────────────┐                  │
│              ▼             ▼             ▼                  │
│       Scenario      buildBubble()   addAudit()             │
│       Definitions   Renderer        Logger                  │
└─────────────────────────────────────────────────────────────┘
                            │
                    (Production Integration)
                            │
              ┌─────────────┼──────────────────┐
              ▼             ▼                  ▼
       SAP S/4HANA    OData Services      BAPIs/RFCs
       (CO Module)   (/sap/opu/odata/…)  (BAPI_IO_*)
```

### 3.2 Component Breakdown

| Component | File Location | Responsibility |
|---|---|---|
| UI Shell | `index.html` lines 1–563 | Header, sidebar, toolbar, chat area, input bar |
| CSS Styles | `<style>` block lines 9–463 | All visual styling — SAP Joule design language |
| State Store | `const state` ~line 562 | Runtime state: scenario, step, data, auditLog, pendingIO |
| Scenario Registry | `const scenarios` ~line 571 | Named registry of all scenario step arrays |
| Scenario Functions | `createScenario()` … `massChangeScenario()` | Step definitions — joule(), match, before hooks |
| Render Engine | `buildBubble(data)` ~line 1595 | Converts data objects to HTML for chat bubbles |
| Message Engine | `processInput(text)` ~line 1724 | Match guard → before hook → advance step → render |
| Scenario Loader | `loadScenario(name)` ~line 1688 | Resets state and starts a named scenario |
| Alert Engine | `triggerAlert(type)` | Renders alert banners and alert bubbles |
| Audit Logger | `addAudit(action, object, id)` | Appends to `state.auditLog` and re-renders sidebar |
| Upload Handler | `handleMassChangeUpload(input)` | Parses CSV, pre-populates state, jumps to impact step |
| Download Handler | `downloadMassChangeTemplate()` | Generates and triggers CSV download |

### 3.3 Rendering Pipeline

```
User Input
    │
    ▼
processInput(text)
    │
    ├─► appendMessage('user', text)         — show user bubble
    │
    ├─► match guard check
    │       └─ fail → reprompt (same quick replies)
    │       └─ pass → continue
    │
    ├─► before(text) hook                   — extract data into state.data
    │
    ├─► state.step++
    │
    ├─► nextStep.joule()                    — generate response data object
    │
    └─► buildBubble(data) → appendMessage   — render agent bubble
```

---

## 4. Data Model

### 4.1 Runtime State Object

```javascript
const state = {
  scenario:  'create',   // Active scenario key
  step:      0,          // Current step index within scenario
  data:      {},         // Collected user data (see §4.2)
  auditLog:  [],         // Array of audit entries (see §4.3)
  pendingIO: null,       // IO ID of most recently created IO
};
```

### 4.2 state.data Fields by Scenario

| Field | Type | Set In | Used In | SAP Field |
|---|---|---|---|---|
| `manualEntry` | Boolean | Create step 1 | Create step 2 | — |
| `desc` | String | Create step 2/3 | Create steps 3–8 | AUFK-KTEXT |
| `companyCode` | String | Create step 2/3 | Create steps 3–8 | AUFK-BUKRS |
| `costCenter` | String | Create step 2/3 | Create steps 3–8 | AUFK-KOSTL |
| `manager` | String | Create step 2/3 | Create steps 3–8 | AUFK-VERANTWORTL |
| `budget` | String | Create step 2/3 | Create steps 3–8 | AUFK budget |
| `validFrom` | String | Create step 2/3 | Create steps 3–8 | AUFK-DATUV |
| `validTo` | String | Create step 2/3 | Create steps 3–8 | AUFK-DATUB |
| `orderType` | String | Create step 2/3 | Create steps 3–8 | AUFK-AUART |
| `postValidationChoice` | String | Create step 4 | Create step 5 | — |
| `changeInput` | String | Change step 1 | Change step 2 | — |
| `adviseChoice` | String | Advise step 0 | Advise steps 1–2 | — |
| `adviseAction` | String | Advise step 1 | Advise step 2 | — |
| `selectedIOs` | Array | Mass Change step 1 | Mass Change steps 2–8 | — |
| `changeField` | String | Mass Change step 2 | Mass Change steps 3–8 | — |
| `newValue` | String | Mass Change step 3 | Mass Change steps 4–8 | — |

### 4.3 Audit Log Entry Schema

```javascript
{
  action:  'CREATE' | 'CHANGE' | 'CLOSE' | 'MASS CHANGE',
  object:  'Internal Order',
  id:      'IO-XXXXXX',
  ts:      '14:32:01',           // toLocaleTimeString()
  user:    'Suman Puthadi',
  source:  'Master Data Agent- Internal Order',
}
```

### 4.4 Internal Order Master Data Fields

| SAP Field | Label | Type | Mandatory | Notes |
|---|---|---|---|---|
| AUFK-AUFNR | IO Number | Char 12 | Auto | Generated: IO-7XXXXX |
| AUFK-KTEXT | Order Description | Char 40 | Yes | Naming convention enforced |
| AUFK-AUART | Order Type | Char 4 | Yes | Project / Investment / Expense / Overhead |
| AUFK-BUKRS | Company Code | Char 4 | Yes | Default: DE01 |
| AUFK-KOKRS | Controlling Area | Char 4 | Derived | A100 |
| AUFK-KOSTV | Responsible Cost Center | Char 10 | Yes | Must be active |
| AUFK-VERANTWORTL | Order Manager | Char 20 | Yes | Must be active, not on leave |
| AUFK-WAERS | Currency | Char 5 | Derived | EUR |
| AUFK-DATUV | Valid From | Date | Yes | |
| AUFK-DATUB | Valid To | Date | Yes | |
| AUFK-ISTAT | System Status | Char | Derived | REL / CLSD |
| Budget Amount | Budget | Amount | Yes | EUR, checked vs EUR 250K threshold |

---

## 5. Scenario Design — Create IO

### 5.1 Flow Overview

| Step | Agent Message | Match Guard | Before Hook | Key Action |
|---|---|---|---|---|
| 0 | Greeting, offer to create | `/.+/` | — | Entry point |
| 1 | Ask: AI suggest or manual? | `/.+/` | Set `manualEntry` | |
| 2 | AI suggestions table OR manual form | `/.+/` | — | Render form or suggestion |
| 3 | Review summary table | `/.+/` | Extract manager & budget | |
| 4 | Run 8 validations | `/.+/` | Store `postValidationChoice` | Validation pipeline |
| 5 | Auth check + approval workflow | `/.+/` | — | Route to approvers |
| 6 | Simulate approvals | `/.+/` | — | All approvers → Approved |
| 7 | Confirmation gate | `/.+/` | — | Show confirm box |
| 8 | Execute creation | `/confirm create/i` | — | Generate IO ID, audit log |
| 9 | Post-creation monitoring | `/.+/` | — | Monitoring activation |

### 5.2 AI Suggestion Values

| Field | Suggested Value |
|---|---|
| Company Code | DE01 |
| Controlling Area | A100 |
| Order Type | Project Order |
| Responsible Cost Center | Global Innovation |
| Currency | EUR |
| Valid From | 01-Jul-2026 |
| Valid To | 31-Dec-2027 |

### 5.3 Manual Form Fields

| Field ID | Type | Mandatory |
|---|---|---|
| `mf_desc` | text | Yes |
| `mf_code` | text | Yes |
| `mf_cc` | text | Yes |
| `mf_mgr` | text | Yes |
| `mf_budget` | number | Yes |
| `mf_from` | date | No |
| `mf_to` | date | No |
| `mf_type` | select | No |

### 5.4 IO ID Generation

```javascript
const ioId = 'IO-' + (780000 + Math.floor(Math.random() * 9999));
```

Range: IO-780000 to IO-789999.

### 5.5 Approval Routing Logic

| Budget Amount | Approval Chain |
|---|---|
| ≤ EUR 250,000 | Cost Center Manager → Master Data Team |
| > EUR 250,000 | Cost Center Manager → Finance Governance Team → Master Data Team |

---

## 6. Scenario Design — Change IO

### 6.1 Flow Overview

| Step | Agent Message | Match Guard | Before Hook | Key Action |
|---|---|---|---|---|
| 0 | List changeable fields | `/.+/` | — | Entry point |
| 1 | Retrieve IO + impact analysis | `/.+/` | Store `changeInput` | Detect budget vs. manager change |
| 2 | Approval routing | `/.+/` | — | Route based on change type |
| 3 | Approvals received | `/.+/` | — | Mark all approved |
| 4 | Confirmation gate | `/.+/` | — | Show confirm box |
| 5 | Execute change | `/confirm change/i` | — | Update S/4HANA, audit log |
| 6 | Post-change monitoring | `/.+/` | — | Updated thresholds |

### 6.2 Change Type Detection

```javascript
const isBudget  = /budget|750|increase/i.test(state.data.changeInput);
const isManager = /manager|anna|michael|john/i.test(state.data.changeInput);
```

### 6.3 Approval Routing by Change Type

| Change Type | Approval Chain |
|---|---|
| Budget increase ≤ 50% | Cost Center Manager |
| Budget increase > 50% | Cost Center Manager → Finance Governance Team |
| Manager reassignment | Cost Center Manager |
| Validity date extension | Notification only (no approval) |
| Cost Center change | Finance Governance Team → Master Data Team |

---

## 7. Scenario Design — Close IO

### 7.1 Flow Overview

| Step | Agent Message | Match Guard | Before Hook | Key Action |
|---|---|---|---|---|
| 0 | List pre-close checks, request IO# | `/.+/` | — | Entry point |
| 1 | Run 7 pre-close checks | `/.+/` | — | Settlement, commitments, POs |
| 2 | Auth check + approval workflow | `/.+/` | — | |
| 3 | Approvals received | `/.+/` | — | All approved |
| 4 | Confirmation gate (irreversible) | `/.+/` | — | ⚠️ Warning shown |
| 5 | Execute closure | `/confirm close/i` | — | Status → CLSD, budget released |
| 6 | Post-close archive confirmation | `/.+/` | — | Monitoring deactivated |

### 7.2 Pre-Close Check List

| # | Check | Expected Result |
|---|---|---|
| 1 | Open purchase orders | None found |
| 2 | Open commitments | None found |
| 3 | Open reservations | None found |
| 4 | Settlement status | Fully settled |
| 5 | Actual costs | All posted and cleared |
| 6 | Remaining budget | Released to Cost Center |
| 7 | Governance sign-off | Not required if release < EUR 100K |

### 7.3 Closure Approval Chain

- Requester → Cost Center Manager → Master Data Team
- If remaining budget > EUR 100,000: + Finance Governance Team

---

## 8. Scenario Design — Query IO

### 8.1 Flow Overview

| Step | Agent Message | Match Guard | Before Hook | Key Action |
|---|---|---|---|---|
| 0 | Explain search dimensions | `/.+/` | — | Entry point |
| 1 | Show results table (5 IOs) | `/.+/` | — | Query S/4HANA |
| 2 | Drill-down detail | `/.+/` | Store `queryInput` | Detect IO number |
| 3 | Post-query action | `/.+/` | — | Notification, extension |

### 8.2 Query Result Table Columns

`IO Number | Description | Budget | Consumed (%) | Manager | Expiry | Status`

### 8.3 Status Indicators

| Symbol | Meaning |
|---|---|
| 🟢 Active | Normal utilization and activity |
| 🟠 Budget Risk | 75–89% consumed |
| 🔴 Critical | ≥ 90% consumed or expiring ≤ 13 days |

### 8.4 Drill-Down Fields

`Company Code | Controlling Area | Cost Center | Manager | Budget | Actual Postings (%) | Remaining | Valid From | Valid To | Days to Expiry | Status | Burn Rate`

---

## 9. Scenario Design — Advise Me

### 9.1 Flow Overview

| Step | Description |
|---|---|
| 0 | Display 9 analysis topic quick replies; store `adviseChoice` |
| 1 | Run analysis checks (5 checks specific to topic) + show recommendation table + action quick replies |
| 2 | Execute chosen action(s); store `adviseAction` |
| 3 | Wrap-up summary and next steps |

### 9.2 Analysis Topics

| Topic | Key Regex | Analysis Checks | Output Table Columns | Actions |
|---|---|---|---|---|
| Which IOs to close this quarter | `/close\|quarter/i` | Activity, budget, expiry, governance, manager | IO#, Description, Recommendation, Priority, Reason | Close, budget review, reassign manager |
| Where are the budget risks | `/budget risk/i` | Burn rate, commitments, threshold breach, extrapolation, notification status | IO#, Budget, Consumed, Remaining, Burn Rate, Depletion, Risk | Extension requests, manager notification |
| Predict cost overruns in 60 days | `/overrun\|predict\|60/i` | Burn rate model, depletion projection, variance, trend, governance threshold | IO#, Budget, Consumed, Burn Rate/Month, Depletion Date, Risk | Extension, close, notify |
| Find duplicate/redundant IOs | `/duplic\|redundant/i` | Name similarity, CC+type overlap, parallel spend, manager cross-ref, policy | IO Group, IO Numbers, Description Overlap, Combined Budget, Recommendation | Consolidate, flag for review |
| Manager workload distribution | `/workload\|manager/i` | Assignment scan, active IO count, budget responsibility, approval bottleneck, CP-007 | Manager, Active IOs, Budget Responsibility, Pending Approvals, Status | Reassign, assign, rebalance |
| Year-end cleanup | `/year.end\|cleanup/i` | FY close date, commitments, settlement rules, carry-forward, governance | IO#, Action Required, Deadline, Priority, Reason | Close, fix settlement, carry-forward |
| Missing settlement rules | `/settlem/i` | Rule completeness, receiver validity, allocation %, run history, GAAP | IO#, Rule, Receiver, Allocation %, Last Run, Status | Create rule, fix receiver, fix allocation |
| Budget release opportunities | `/release\|opportunit/i` | Utilization scan, commitments, activity, PM completion, governance threshold | IO#, Total Budget, Consumed, Releasable Budget, Last Activity, Recommendation | Release immediately, partial release |

---

## 10. Scenario Design — Mass Change IO

### 10.1 Flow Overview

| Step | Description | Key Mechanism |
|---|---|---|
| 0 | Greeting with 3 action buttons: Show IOs, Upload Template, Download Template | `actionButtons` renderer |
| 1 | IO selection table with checkboxes | `massChangeTable` renderer |
| 2 | Echo selected IOs, ask which field to change | `state.data.selectedIOs` array |
| 3 | Render field-specific form | `massChangeForm` renderer; `submitMassChangeForm()` |
| 4 | Impact analysis table (current → new value per IO) | Dynamic table rows from `selectedIOs` |
| 5 | Consolidated approval workflow | Single workflow for all IOs |
| 6 | All approvals received | Mark all 4 approvers as done |
| 7 | Confirmation gate | Type "Confirm Mass Change" |
| 8 | Execute changes + audit log per IO | `addAudit()` called per IO |
| 9 | Post-change monitoring | Summary + next steps |

### 10.2 Supported Mass Change Fields

| Field | Form Element | Form ID | state.data.changeField |
|---|---|---|---|
| Budget Amount | number input | `mcf_budget` | `'budget'` |
| Order Manager | select dropdown | `mcf_manager` | `'manager'` |
| Validity Dates | two date inputs | `mcf_from`, `mcf_to` | `'validity'` |
| Cost Center | text input | `mcf_cc` | `'costcenter'` |

### 10.3 IO Selection Table Columns

`Select (checkbox) | IO Number | Description | Budget | Consumed | Manager | Status`

### 10.4 Consolidated Approval Chain

All mass changes route through: Requester → Cost Center Manager → Finance Governance Team → Master Data Team (single workflow regardless of field or number of IOs).

### 10.5 Impact Analysis Table Columns

`IO Number | Description | Current Value | New Value | Approval Needed`

### 10.6 Approval Needed by Field

| Field Changed | Approval Required |
|---|---|
| Budget | Yes — Finance Governance |
| Order Manager | Yes — Cost Center Manager |
| Validity Dates | Notification only |
| Cost Center | Yes — Finance Governance |

---

## 11. Validation Framework

### 11.1 Create IO Validation Pipeline (8 Checks)

| # | Check | Pass Condition | SAP Object Checked |
|---|---|---|---|
| 1 | Mandatory field validation | All required fields non-empty | — |
| 2 | Duplicate IO check | No IO with same description + company code | AUFK |
| 3 | Naming convention compliance | Matches corporate naming policy | Custom Z-table |
| 4 | Cost Center validity | Cost Center active for specified date range | CSKS |
| 5 | Manager assignment validity | Employee active, not on extended leave | PA0000 / PA0001 |
| 6 | Budget threshold verification | Budget within policy limits | — |
| 7 | Corporate controlling policies | Compliant with CP-007 | Custom policy table |
| 8 | Financial governance rules | Fiscal period open; user authorized | T009B / Authorization |

### 11.2 Match Guard Pattern

Each scenario step carries a `match` regex. If the user's input does not match:

```javascript
if (currentStep.match && !currentStep.match.test(text)) {
  // Reprompt with same quick replies — do NOT advance step
  appendMessage('joule', buildBubble({ text: 'I\'m not sure I understood...', quickReplies: [...] }), true);
  return;
}
```

Only **confirmation gates** use strict phrase guards:

| Gate | Regex |
|---|---|
| Create | `/confirm create/i` |
| Change | `/confirm change/i` |
| Close | `/confirm close/i` |
| Mass Change | `/confirm mass change/i` |

All other steps use `/.+/` to accept any non-empty input.

---

## 12. Approval Workflow Engine

### 12.1 Workflow Node Schema

```javascript
{
  label: 'Cost Center Manager — Approved',
  done: true   // true = green dot; false = dashed pending dot
}
```

### 12.2 Workflow States

| State | Visual | Colour |
|---|---|---|
| Done | Filled circle | #48bb78 (green) with glow |
| Pending | Dashed circle | White with grey dashes |

### 12.3 Approval Simulation

The application simulates approval via the "Simulate Approval" quick reply. In production, approval notifications would be received via SAP Workflow, Business Workplace (SBWP), or SAP Business Network triggers.

### 12.4 Standard Approval Chain by Scenario

| Scenario | Approvers |
|---|---|
| Create (≤ EUR 250K) | Cost Center Manager → Master Data Team |
| Create (> EUR 250K) | Cost Center Manager → Finance Governance → Master Data Team |
| Change (budget > 50% increase) | Cost Center Manager → Finance Governance |
| Change (manager) | Cost Center Manager |
| Close | Cost Center Manager → Master Data Team |
| Close (> EUR 100K released) | + Finance Governance Team |
| Mass Change (any field) | Cost Center Manager → Finance Governance → Master Data Team |

---

## 13. Alert & Monitoring System

### 13.1 Alert Types

| Alert Type | Trigger Condition | Banner Class | Sidebar Badge |
|---|---|---|---|
| Budget Alert | IO consumed ≥ 75% or ≥ 90% | `alert-banner warning` | ⚠️ red count |
| Expiry Alert | IO expires within 30 days | `alert-banner info` | 📅 count |
| Activity Alert | No postings for ≥ 90 days (180 days = escalated) | `alert-banner warning` | 🔔 count |
| Governance Alert | Manager on leave / inactive CC / naming violation | `alert-banner warning` | 🛡️ count |

### 13.2 Alert Configuration Schema

```javascript
alertConfig[type] = {
  bannerClass: 'alert-banner warning',
  bannerHtml:  '⚠️ <b>Budget Alert:</b> ...',
  bubble: {
    text:         '...',
    table:        { rows: [...], header: true },
    extraText:    '...',
    quickReplies: ['...'],
  }
}
```

### 13.3 Budget Alert — Monitored IOs (Demo Data)

| IO Number | Description | Budget | Consumed | Risk |
|---|---|---|---|---|
| IO-765881 | Cloud Migration Phase 2 | EUR 1.2M | 89% | 🔴 Critical |
| IO-782145 | AI Transformation Program | EUR 500K | 85% | 🟠 High |

### 13.4 Expiry Alert — Monitored IOs (Demo Data)

| IO Number | Expiry Date | Days Left | Consumed |
|---|---|---|---|
| IO-748832 | 15-Jun-2026 | 13 days | 97% |
| IO-755021 | 30-Jun-2026 | 28 days | 12% |
| IO-738001 | 05-Jul-2026 | 33 days | 78% |

### 13.5 Monitoring Frequencies (Production)

| Alert Type | Check Frequency |
|---|---|
| Budget (75%) | Daily batch job |
| Budget (90%) | Real-time on posting |
| Expiry | Daily batch job |
| Activity | Weekly batch job |
| Governance | Real-time on HR/org change |

---

## 14. UI Component Library

All components are rendered by `buildBubble(data)`. Each is triggered by a specific key in the data object.

### 14.1 Component Reference

| Data Key | Component | Description |
|---|---|---|
| `data.text` | Paragraph | Plain text or HTML string |
| `data.table` | Data Table | Striped table with optional header row; horizontal scroll |
| `data.steps` | Step List | Icon + text rows with green left border; used for validation and execution |
| `data.workflow` | Workflow Chart | Vertical node chain with done/pending dots and connecting arrows |
| `data.confirmBox` | Confirm Box | Blue bordered box with key-value pairs and instruction label |
| `data.successCard` | Success Card | Green gradient card with IO ID, budget, manager, status |
| `data.quickReplies` | Quick Reply Buttons | Pill buttons; disabled after click |
| `data.manualForm` | Manual Entry Form | 2-column grid form; calls `submitManualForm()` |
| `data.massChangeTable` | Mass Change Selector | IO table with checkboxes, Select All, Confirm Selection, Download Template buttons |
| `data.massChangeForm` | Mass Change Form | Field-specific single-field form; calls `submitMassChangeForm()` |
| `data.actionButtons` | Action Buttons | Stacked primary/secondary buttons with custom `onclick` handlers |
| `data.extraText` | Extra Text | HTML paragraph rendered after the main component |

### 14.2 Message Bubble Architecture

```
.msg.joule                          .msg.user
  .avatar.joule (AI)                  .avatar.user (SP)
  .bubble                             .bubble
    [component HTML]                    [plain text]
```

### 14.3 CSS Design Tokens

| Token | Value | Usage |
|---|---|---|
| Primary Blue | `#0a6ed1` | Buttons, headers, links, active states |
| Dark Blue | `#054a8c` | Gradient ends, avatar fills |
| Success Green | `#48bb78` / `#276749` | Approved states, success chips |
| Warning Amber | `#f6ad55` | Budget/activity alert banners |
| Error Red | `#e53e3e` | Critical status, badges |
| Background | `#eef2f7` | Chat area background |
| Surface | `#ffffff` | Bubbles, sidebar, input bar |
| Border | `#e2e8f0` | All dividers and outlines |
| Font | Inter / Segoe UI / -apple-system | Body text |

---

## 15. Mass Change Template — Upload/Download

### 15.1 Template CSV Column Structure

| # | Column | Data Type | Description |
|---|---|---|---|
| 1 | IO Number | String | Internal Order ID (e.g. IO-782145) |
| 2 | Description | String | Order description |
| 3 | Budget (EUR) | Number | Current budget amount |
| 4 | Order Manager | String | Current responsible manager |
| 5 | Valid From | Date | Current valid from date |
| 6 | Valid To | Date | Current valid to date |
| 7 | Cost Center | String | Current responsible cost center |
| 8 | **New Budget (EUR)** | Number | Fill to change budget |
| 9 | **New Order Manager** | String | Fill to change manager |
| 10 | **New Valid From** | Date | Fill to change validity start |
| 11 | **New Valid To** | Date | Fill to change validity end |
| 12 | **New Cost Center** | String | Fill to change cost center |

Columns 1–7 are pre-filled. Columns 8–12 are blank — the user fills exactly one group.

### 15.2 Upload Processing Logic (`handleMassChangeUpload`)

```
1. FileReader reads CSV as text
2. Split on \r?\n, filter empty lines
3. Skip header row (index 0)
4. Map each row → { id, desc, budget, manager, validFrom, validTo, costCenter,
                    newBudget, newManager, newValidFrom, newValidTo, newCC }
5. Filter rows where id is non-empty
6. Detect which "New" column was filled (priority: budget > manager > validity > costcenter)
7. Validate: at least one row, at least one new value
8. Pre-populate: state.data.selectedIOs, state.data.changeField, state.data.newValue
9. Show change preview table in chat (IO#, Description, Field, Current Value, New Value)
10. Ask for confirmation: "Yes, proceed with changes" / "Cancel"
11. On confirm → advance to step 4 (impact analysis)
```

### 15.3 Error Handling

| Error Condition | Message Shown |
|---|---|
| File has < 2 lines | "The uploaded file appears to be empty or invalid" |
| No rows with IO Number | "No valid Internal Order rows found" |
| No "New" columns filled | "No new values found in the template. Please fill in one of the New columns" |

### 15.4 Download Logic (`downloadMassChangeTemplate`)

```javascript
1. Build rows array: [header, ...5 pre-filled IO rows]
2. CSV-encode: wrap each cell in double quotes, escape internal quotes
3. new Blob([csv], { type: 'text/csv' })
4. URL.createObjectURL(blob) → anchor click → URL.revokeObjectURL()
5. Filename: MassChange_IO_Template.csv
```

---

## 16. Audit Trail

### 16.1 Log Entry Creation

```javascript
function addAudit(action, object, id) {
  const now = new Date();
  state.auditLog.unshift({
    action, object, id,
    ts:     now.toLocaleTimeString(),
    user:   'Suman Puthadi',
    source: 'Master Data Agent- Internal Order',
  });
  renderAudit();
}
```

### 16.2 Sidebar Rendering

- Last **4 entries** displayed in sidebar audit panel
- Full log preserved in `state.auditLog` (in-memory)
- Format: `[ACTION] Internal Order | IO-XXXXXX · HH:MM:SS | By: Suman Puthadi`

### 16.3 When Audit Entries Are Created

| Trigger | Action Logged |
|---|---|
| Create IO step 8 | `CREATE` |
| Change IO step 5 | `CHANGE` |
| Close IO step 5 | `CLOSE` |
| Mass Change step 8 | `MASS CHANGE` (one entry per IO) |
| Advise — close actions | `CLOSE` (per closed IO) |
| Advise — change actions | `CHANGE` |

---

## 17. SAP Integration Layer

### 17.1 OData Services (Production Target)

| Service | Endpoint | Usage |
|---|---|---|
| Internal Order Read | `/sap/opu/odata/sap/CO_INTERNALORDER_SRV/InternalOrders` | Query, drill-down |
| Internal Order Create | `/sap/opu/odata/sap/CO_INTERNALORDER_SRV/InternalOrders` POST | Create IO |
| Internal Order Change | `/sap/opu/odata/sap/CO_INTERNALORDER_SRV/InternalOrders('AUFNR')` PATCH | Change IO |
| Cost Center Read | `/sap/opu/odata/sap/FCO_PI_CO_SRV/CostCenters` | Validate cost center |
| Budget Read | `/sap/opu/odata/sap/FCOM_BUDGETMANAGEMENT_SRV/Budgets` | Budget utilization |
| Employee Read | `/sap/opu/odata/sap/HCM_EMPLOYEE_SRV/Employees` | Validate manager |

### 17.2 BAPIs / Function Modules

| BAPI | Purpose |
|---|---|
| `BAPI_INTERNALORDER_CREATE` | Create Internal Order in CO module |
| `BAPI_INTERNALORDER_CHANGE` | Change Internal Order master data |
| `BAPI_INTERNALORDER_GETLIST` | Read list of Internal Orders |
| `BAPI_INTERNALORDER_GETINFO` | Read single Internal Order detail |
| `CO_OM_ORD_CLOSE` | Technical closure of Internal Order |
| `BAPI_COSTCENTER_GETINFO` | Validate Cost Center existence and validity |
| `BAPI_BUDGET_GETINFO` | Read budget and actual data |

### 17.3 SAP Tables Referenced

| Table | Description | Used For |
|---|---|---|
| AUFK | Order Master Data | Primary IO data |
| AUFNR | Order Number Range | IO ID generation |
| CSKS | Cost Center Master Record | Cost Center validation |
| CSKA | Cost Center Attributes | Activity type assignment |
| COEP | CO Document: Line Items (actual) | Actual postings, activity detection |
| BPGE | Budget: Overall Values | Budget availability |
| T009B | Fiscal Year Variants | Fiscal period validation |
| PA0000 | HR Actions (Infotype 0000) | Employee status validation |
| PA0001 | Organizational Assignment (Infotype 0001) | Manager validation |

---

## 18. Security & Governance

### 18.1 Authorization Checks

| Check | Timing | SAP Auth Object |
|---|---|---|
| IO creation authorization | Create step 5 | `K_ORDER — ACTVT 01` |
| IO change authorization | Change step 2 | `K_ORDER — ACTVT 02` |
| IO closure authorization | Close step 2 | `K_ORDER — ACTVT 12` |
| Budget release > EUR 100K | Close step 2 | `F_BKPF_BUK` |
| Mass change authorization | Mass Change step 5 | `K_ORDER — ACTVT 02` |

### 18.2 Governance Safeguards

| Safeguard | Description |
|---|---|
| Confirmation Gates | Write operations require exact phrase: "Confirm Create/Change/Close/Mass Change" |
| Validation Pipeline | 8 automated checks before any approval request |
| Approval Workflow | Threshold-based multi-level approval — no auto-execution |
| Irreversibility Warning | Close step 4 explicitly warns: "This action is irreversible" |
| Audit Trail | 100% coverage of all write operations with user, timestamp, and action |
| Duplicate Detection | Prevent duplicate IO creation by description + company code |
| CP-007 Compliance | Manager assignment policy enforced in validation and Advise Me analysis |

### 18.3 Corporate Policy References

| Policy | Description |
|---|---|
| CP-007 | Corporate Controlling Policy — manager assignment, max 8 IOs per manager |
| Budget Threshold | EUR 250,000 auto-approval limit; above requires Finance Governance sign-off |
| Governance Sign-off | Required for budget releases > EUR 100,000 |
| Naming Convention | IO descriptions must follow corporate naming policy (validated at step 4 check 3) |

---

## 19. Non-Functional Requirements

### 19.1 Performance

| Metric | Target |
|---|---|
| IO creation time | < 5 minutes end-to-end (vs. 45 min baseline) |
| Query response time | < 2 seconds for standard queries |
| Agent response delay | 800–1,400ms simulated typing indicator |
| Page load time | < 1 second (single HTML file, no external dependencies except Google Fonts) |

### 19.2 Reliability

| Requirement | Design Decision |
|---|---|
| No backend dependency | All logic client-side; no server failure risk |
| Match guard protection | Invalid input never advances state |
| Confirmation gate | Prevents accidental write operations |
| State persistence | In-memory within session; `state.auditLog` retains full session history |

### 19.3 Usability

| Requirement | Design Decision |
|---|---|
| Quick replies | Every step provides at least one quick reply to guide the user |
| No dead-end steps | All steps have `/.+/` match guard — user can always type something |
| Error messages | Validation failures shown inline with clear explanations |
| Responsive layout | Flexbox layout; tables with horizontal scroll on small screens |

### 19.4 Business KPIs (from PRD)

| Metric | Baseline | Target (6 months) |
|---|---|---|
| IO creation time | 45 min | < 5 min |
| Master data error rate | 12% | < 1% |
| Budget overrun detection lag | 30 days | Same day |
| Zombie IO closure rate | 15%/quarter | 90%/quarter |
| User adoption (% via agent) | 0% | 70% |
| Audit trail coverage | 40% | 100% |

---

*End of Software Design Document*

*Document maintained by: Suman Puthadi — Master Data Agent Team*
*Repository: https://github.com/sumanputhadi9342/AI_IO_CPIT_V2*
