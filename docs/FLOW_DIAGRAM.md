# Flow Diagram — Master Data Agent (Internal Order)
## End-to-End User Journey & System Flow

---

## 1. High-Level Application Flow

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    USER ENTRY POINT                                          │
│              MDA-IO Chat Interface (Browser)                                │
└───────────────────────────┬─────────────────────────────────────────────────┘
                            │
                            ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                    INTENT DETECTION                                          │
│   ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐   │
│   │ CREATE   │  │  CHANGE  │  │  CLOSE   │  │  QUERY   │  │  ADVISE  │   │
│   │   IO     │  │    IO    │  │    IO    │  │    IO    │  │    ME    │   │
│   └────┬─────┘  └────┬─────┘  └────┬─────┘  └────┬─────┘  └────┬─────┘   │
└────────│─────────────│─────────────│─────────────│─────────────│───────────┘
         │             │             │             │             │
         ▼             ▼             ▼             ▼             ▼
    [See §2]       [See §3]      [See §4]      [See §5]      [See §6]
```

---

## 2. Create Internal Order Flow

```
User: "Create a new Internal Order for AI Transformation Program"
                            │
                            ▼
              ┌─────────────────────────┐
              │  Agent acknowledges &   │
              │  presents options:      │
              │  • Suggest values (AI)  │
              │  • Enter manually       │
              └────────────┬────────────┘
                           │
           ┌───────────────┴───────────────┐
           │                               │
           ▼                               ▼
  ┌─────────────────┐             ┌─────────────────┐
  │  AI SUGGESTION  │             │  MANUAL FORM    │
  │  Mode           │             │  Mode           │
  │                 │             │                 │
  │ Agent queries   │             │ User fills in:  │
  │ historical IOs  │             │ • Description   │
  │ and org data:   │             │ • Company Code  │
  │ • Company Code  │             │ • Cost Center   │
  │ • Ctrl Area     │             │ • Manager       │
  │ • Order Type    │             │ • Budget        │
  │ • Cost Center   │             │ • Dates         │
  │ • Currency      │             │ • Order Type    │
  │ • Dates         │             └────────┬────────┘
  │ • Managers list │                      │
  └────────┬────────┘                      │
           └───────────────┬───────────────┘
                           │
                           ▼
              ┌─────────────────────────┐
              │  REVIEW SUMMARY         │
              │  Agent shows full IO    │
              │  details for review     │
              └────────────┬────────────┘
                           │
                           ▼
              ┌─────────────────────────┐
              │  BACKGROUND VALIDATION  │
              │  ✅ Mandatory fields     │
              │  ✅ Duplicate check      │
              │  ✅ Naming convention    │
              │  ✅ Cost Center validity │
              │  ✅ Manager validity     │
              │  ✅ Budget policy check  │
              │  ✅ Governance rules     │
              └────────────┬────────────┘
                           │
                    ┌──────┴──────┐
                    │             │
               All pass       Any fail
                    │             │
                    ▼             ▼
         ┌──────────────┐  ┌──────────────┐
         │ AUTHORIZATION│  │ Show errors, │
         │ CHECK        │  │ ask for      │
         │              │  │ corrections  │
         │ Budget >      │  └──────────────┘
         │ EUR 250K?    │
         └──────┬───────┘
                │
        ┌───────┴────────┐
        │                │
       Yes               No
        │                │
        ▼                ▼
┌───────────────┐  ┌──────────────────┐
│ APPROVAL      │  │ CONFIRMATION     │
│ WORKFLOW      │  │ GATE             │
│               │  │                  │
│ → Cost Center │  │ Show summary +   │
│   Manager     │  │ "Type Confirm    │
│ → Finance     │  │  Create"         │
│   Governance  │  └────────┬─────────┘
│ → Master Data │           │
│   Team        │     User confirms
└───────┬───────┘           │
        │                   │
   All approved             │
        │                   │
        ▼                   ▼
        └──────────┬────────┘
                   │
                   ▼
      ┌────────────────────────┐
      │  EXECUTE IN S/4HANA    │
      │  ✅ IO created          │
      │  ✅ Cost Center linked  │
      │  ✅ Manager assigned    │
      │  ✅ Budget posted       │
      │  ✅ Dates applied       │
      │  ✅ MDR updated         │
      │  ✅ Systems synced      │
      └────────────┬───────────┘
                   │
                   ▼
      ┌────────────────────────┐
      │  POST-CREATE           │
      │  • Audit log entry     │
      │  • Email notifications │
      │  • Monitoring activated│
      │    (budget 75%/90%,    │
      │     expiry -30d,       │
      │     activity 90d,      │
      │     governance checks) │
      └────────────────────────┘
```

---

## 3. Change Internal Order Flow

```
User: "Change IO-782145 budget to EUR 750,000"
                   │
                   ▼
      ┌────────────────────────┐
      │  RETRIEVE IO           │
      │  Current values from   │
      │  S/4HANA               │
      └────────────┬───────────┘
                   │
                   ▼
      ┌────────────────────────┐
      │  IMPACT ANALYSIS       │
      │  • Delta calculation   │
      │  • Policy threshold    │
      │  • Commitment check    │
      │  • Naming impact       │
      └────────────┬───────────┘
                   │
          ┌────────┴────────┐
     Approval              No approval
     required              needed
          │                     │
          ▼                     ▼
   [Approval Workflow]   [Confirmation Gate]
          │                     │
          └──────────┬──────────┘
                     ▼
         ┌─────────────────────┐
         │  EXECUTE CHANGE     │
         │  ✅ Budget updated   │
         │  ✅ Change doc       │
         │  ✅ MDR synced       │
         │  ✅ Systems notified │
         └─────────────────────┘
```

---

## 4. Close Internal Order Flow

```
User: "Close IO-782145"
                   │
                   ▼
      ┌────────────────────────┐
      │  PRE-CLOSE CHECKS      │
      │  ✅ Open POs           │
      │  ✅ Open commitments   │
      │  ✅ Open reservations  │
      │  ✅ Settlement status  │
      │  ⚠️ Remaining budget   │
      │     (to be released)   │
      └────────────┬───────────┘
                   │
          ┌────────┴────────┐
       All pass          Blockers
          │               found
          ▼                  │
  [Approval Workflow]       ▼
          │         [Show blockers,
          ▼          suggest fix]
  ┌────────────────┐
  │ CONFIRMATION   │
  │ GATE           │
  │ ⚠️ IRREVERSIBLE│
  │ "Confirm Close"│
  └───────┬────────┘
          │
          ▼
  ┌────────────────────────┐
  │  EXECUTE CLOSURE       │
  │  ✅ Status → Closed    │
  │  ✅ Budget released    │
  │  ✅ Postings locked    │
  │  ✅ MDR updated        │
  │  ✅ Audit doc created  │
  └────────────────────────┘
```

---

## 5. Query Internal Order Flow

```
User: "Show all active IOs for Cost Center Global Innovation"
                   │
                   ▼
      ┌────────────────────────┐
      │  PARSE QUERY PARAMS    │
      │  Filter by:            │
      │  • Cost Center         │
      │  • Status / Manager    │
      │  • Budget / Dates      │
      └────────────┬───────────┘
                   │
                   ▼
      ┌────────────────────────┐
      │  QUERY S/4HANA         │
      │  Return paginated      │
      │  result table          │
      └────────────┬───────────┘
                   │
                   ▼
      ┌────────────────────────┐
      │  RISK SURFACING        │
      │  Flag at-risk orders   │
      │  with recommendations  │
      └────────────┬───────────┘
                   │
                   ▼
      ┌────────────────────────┐
      │  DRILL-DOWN (optional) │
      │  Full detail view +    │
      │  inline action buttons │
      └────────────────────────┘
```

---

## 6. Advise / Portfolio Analysis Flow

```
User: "Which IOs should be closed this quarter?"
                   │
                   ▼
      ┌───────────────────────────────────────┐
      │  PORTFOLIO ANALYSIS ENGINE             │
      │                                        │
      │  Dimension 1: Posting Activity         │
      │  Dimension 2: Budget Utilization       │
      │  Dimension 3: Expiry Risk              │
      │  Dimension 4: Governance Compliance    │
      │  Dimension 5: Manager Validity         │
      └──────────────────┬────────────────────┘
                         │
                         ▼
      ┌────────────────────────────────────────┐
      │  PRIORITIZED RECOMMENDATIONS           │
      │  🔴 High: Close IO-755021, IO-742300   │
      │  🟠 Medium: Budget review IO-765881    │
      │  🟡 Low: Reassign manager IO-759044    │
      └──────────────────┬─────────────────────┘
                         │
                         ▼
      ┌────────────────────────────────────────┐
      │  BULK ACTION INITIATION                │
      │  User can act on 1 or all at once      │
      │  → Queues approval workflows           │
      │  → Creates audit entries               │
      └────────────────────────────────────────┘
```

---

## 7. Proactive Alert Flow

```
BACKGROUND MONITORING ENGINE (always running)
           │
    ┌──────┴──────────────────────────────┐
    │         │            │              │
    ▼         ▼            ▼              ▼
 Budget    Expiry      Activity      Governance
 Monitor   Monitor     Monitor       Monitor
    │         │            │              │
  IO > 75%  IO expiry   No postings   Manager
  or 90%    within 30d   > 90 days    invalid
    │         │            │              │
    └────────┬┴────────────┴──────────────┘
             │
             ▼
    ┌────────────────────┐
    │  ALERT TRIGGERED   │
    │  • Banner shown    │
    │  • Chat message    │
    │  • Action buttons  │
    └────────────────────┘
             │
             ▼
    ┌────────────────────┐
    │  USER ACTION       │
    │  (optional)        │
    │  • Extend IO       │
    │  • Close IO        │
    │  • Notify manager  │
    │  • Dismiss         │
    └────────────────────┘
```

---

## 8. Approval Workflow Routing Logic

```
Action Requested
       │
       ▼
Is Budget > EUR 250,000? ─── Yes ──▶ Route to Finance Governance Team
       │
       No
       │
       ▼
Is Budget Increase > 50%? ─── Yes ──▶ Route to Cost Center Manager
       │                               + Finance Governance Team
       No
       │
       ▼
Is Closure with remaining
budget > EUR 100,000? ──── Yes ──▶ Route to Cost Center Manager
       │                           + Finance Governance Team
       No
       │
       ▼
Standard route:
Cost Center Manager ──▶ Master Data Team
```
