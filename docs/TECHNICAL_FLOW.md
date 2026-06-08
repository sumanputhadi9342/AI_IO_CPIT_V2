# Technical Flow Diagram — Master Data Agent (Internal Order)
## Architecture, Data Flow & Integration Layers

---

## 1. System Architecture Overview

```
┌──────────────────────────────────────────────────────────────────────────────┐
│                        PRESENTATION LAYER                                     │
│                                                                               │
│   ┌─────────────────────────────────────────────────────────────────────┐   │
│   │                 MDA-IO Web Application (HTML/CSS/JS)                │   │
│   │  ┌──────────┐  ┌──────────────────────┐  ┌──────────────────────┐  │   │
│   │  │ Sidebar  │  │   Chat Interface      │  │   Alert Banner       │  │   │
│   │  │ (Nav +   │  │   (Messages, QR,     │  │   (Budget/Expiry/    │  │   │
│   │  │  Audit)  │  │    Forms, Tables)    │  │    Activity/Gov)     │  │   │
│   │  └──────────┘  └──────────────────────┘  └──────────────────────┘  │   │
│   └─────────────────────────────────────────────────────────────────────┘   │
└──────────────────────────┬───────────────────────────────────────────────────┘
                           │  User Input / Agent Output
                           ▼
┌──────────────────────────────────────────────────────────────────────────────┐
│                        AGENT ORCHESTRATION LAYER                              │
│                                                                               │
│   ┌──────────────────┐   ┌──────────────────┐   ┌──────────────────────┐   │
│   │  Intent          │   │  Scenario        │   │  State Manager       │   │
│   │  Recognition     │──▶│  Router          │──▶│  (step, data,        │   │
│   │  Engine          │   │  (create/change/ │   │   pendingIO,         │   │
│   │                  │   │   close/query/   │   │   auditLog)          │   │
│   └──────────────────┘   │   advise)        │   └──────────────────────┘   │
│                          └──────────────────┘                               │
│                                   │                                          │
│   ┌──────────────────┐   ┌────────┴─────────┐   ┌──────────────────────┐   │
│   │  Validation      │   │  Recommendation  │   │  Alert Monitor       │   │
│   │  Engine          │   │  Engine          │   │  Engine              │   │
│   │  (8 checks)      │   │  (historical AI) │   │  (4 alert types)     │   │
│   └──────────────────┘   └──────────────────┘   └──────────────────────┘   │
│                                                                               │
│   ┌──────────────────┐   ┌──────────────────┐                               │
│   │  Approval        │   │  Audit Trail     │                               │
│   │  Workflow Engine │   │  Logger          │                               │
│   │  (threshold      │   │  (action/object/ │                               │
│   │   routing)       │   │   id/ts/user)    │                               │
│   └──────────────────┘   └──────────────────┘                               │
└──────────────────────────┬───────────────────────────────────────────────────┘
                           │  API Calls / RFC
                           ▼
┌──────────────────────────────────────────────────────────────────────────────┐
│                        SAP INTEGRATION LAYER                                  │
│                                                                               │
│   ┌──────────────────────────────────────────────────────────────────────┐  │
│   │                     SAP S/4HANA (System DE01)                        │  │
│   │                                                                      │  │
│   │  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────────────┐   │  │
│   │  │ KO01     │  │ KO02     │  │ KO04     │  │ CO Master Data   │   │  │
│   │  │ Create   │  │ Change   │  │ Close    │  │ Repository       │   │  │
│   │  │ IO       │  │ IO       │  │ IO       │  │                  │   │  │
│   │  └──────────┘  └──────────┘  └──────────┘  └──────────────────┘   │  │
│   │                                                                      │  │
│   │  ┌──────────────────────────────────────────────────────────────┐   │  │
│   │  │                   Core Tables                                 │   │  │
│   │  │  AUFK │ COBL │ CSKS │ CSKA │ AUFNR │ COBRA │ BKPF/BSEG     │   │  │
│   │  └──────────────────────────────────────────────────────────────┘   │  │
│   └──────────────────────────────────────────────────────────────────────┘  │
└──────────────────────────────────────────────────────────────────────────────┘
```

---

## 2. Detailed Request Processing Flow

```
USER INPUT
    │
    ▼
┌─────────────────────────────────────────────────────────────────┐
│  STEP 1: INPUT PROCESSING                                        │
│  • Read text from input bar or quick-reply button                │
│  • Strip whitespace, normalize                                   │
│  • Record in user message bubble                                 │
│  • Disable input to prevent double-submit                        │
│  • Show typing indicator (800–1400ms simulated delay)            │
└──────────────────────────┬──────────────────────────────────────┘
                           │
                           ▼
┌─────────────────────────────────────────────────────────────────┐
│  STEP 2: SCENARIO & STEP RESOLUTION                              │
│  • Look up current scenario in state.scenario                    │
│  • Look up current step index in state.step                      │
│  • Execute before() hook if defined (stores extracted data)      │
│  • Advance state.step to next step                               │
└──────────────────────────┬──────────────────────────────────────┘
                           │
                           ▼
┌─────────────────────────────────────────────────────────────────┐
│  STEP 3: RESPONSE GENERATION                                     │
│  • Call joule() function for the next step                       │
│  • joule() may reference state.data for dynamic content         │
│  • Returns response object with:                                 │
│    - text: string (HTML allowed)                                 │
│    - table?: { rows, header }                                    │
│    - steps?: [{ icon, text }]                                    │
│    - workflow?: [{ label, done }]                                │
│    - confirmBox?: { items, instruction }                         │
│    - successCard?: { ioId, description, budget, manager }        │
│    - quickReplies?: string[]                                     │
│    - manualForm?: boolean                                        │
│    - extraText?: string                                          │
└──────────────────────────┬──────────────────────────────────────┘
                           │
                           ▼
┌─────────────────────────────────────────────────────────────────┐
│  STEP 4: RENDER TO DOM                                           │
│  buildBubble(data) → HTML string                                 │
│  appendMessage('joule', html, isRaw=true)                        │
│  Auto-scroll to latest message                                   │
│  Re-enable input                                                 │
└──────────────────────────┬──────────────────────────────────────┘
                           │
                           ▼
┌─────────────────────────────────────────────────────────────────┐
│  STEP 5: WRITE ACTIONS (on confirm steps only)                   │
│  addAudit(action, object, id)                                    │
│  → In real implementation: POST to SAP OData / RFC               │
│  → Creates change document in S/4HANA                           │
│  → Updates Master Data Repository                                │
│  → Triggers connected system synchronization                     │
│  → Sends email notifications via SAP Business Workplace          │
└─────────────────────────────────────────────────────────────────┘
```

---

## 3. Validation Engine — Technical Detail

```
INPUT DATA
    │
    ▼
┌──────────────────────────────────────────────────────────────────┐
│  VALIDATION PIPELINE (runs in sequence)                           │
│                                                                   │
│  Check 1: Mandatory Field Validation                              │
│    • Verify description, company code, cost center,               │
│      manager, budget, valid-from, valid-to all present           │
│    • Result: PASS / FAIL with field list                          │
│                                                                   │
│  Check 2: Duplicate Internal Order Detection                      │
│    • Query AUFK table: WHERE KTEXT = :description                 │
│      AND BUKRS = :companyCode AND AUFART = :orderType            │
│    • Result: PASS (no duplicate) / WARN (similar found)           │
│                                                                   │
│  Check 3: Naming Convention Compliance                            │
│    • Apply regex against corporate naming policy rules            │
│    • Result: PASS / FAIL with suggestion                          │
│                                                                   │
│  Check 4: Cost Center Validity                                    │
│    • Query CSKS: WHERE KOSTL = :costCenter                        │
│      AND DATBI >= :validFrom AND DATAB <= :validTo               │
│      AND LOEVM != 'X'                                             │
│    • Result: PASS / FAIL                                          │
│                                                                   │
│  Check 5: Manager Assignment Validity                             │
│    • Query PA0001 (HR master): WHERE PERNR = :manager             │
│      AND STAT2 = '3' (active employee)                           │
│    • Result: PASS / FAIL / WARN (on leave)                        │
│                                                                   │
│  Check 6: Budget Threshold Verification                           │
│    • Compare against controlling area budget limits               │
│    • Flag if > EUR 250,000 for approval routing                   │
│    • Result: PASS / APPROVAL_REQUIRED                             │
│                                                                   │
│  Check 7: Corporate Controlling Policy (CP-007)                  │
│    • Verify order type allowed for cost center                    │
│    • Verify validity period within fiscal year boundaries         │
│    • Result: PASS / FAIL                                          │
│                                                                   │
│  Check 8: Financial Governance Rules                              │
│    • Verify user authorization object CO_AUFNR                    │
│    • Verify company code fiscal period is open                    │
│    • Result: PASS / FAIL                                          │
└──────────────────────────────────────────────────────────────────┘
    │
    ▼
ALL PASS → Proceed to Authorization Check
ANY FAIL → Return error list to user
```

---

## 4. State Management Data Model

```javascript
// Runtime state object
state = {
  scenario: 'create' | 'change' | 'close' | 'query' | 'advise',
  step: Number,           // current step index in scenario array
  data: {
    // dynamically populated per scenario
    manualEntry: Boolean,
    manager: String,      // e.g. "Anna Mueller"
    budget: String,       // e.g. "EUR 500,000"
    desc: String,
    costCenter: String,
    companyCode: String,
    validFrom: String,
    validTo: String,
    orderType: String,
    changeInput: String,  // raw input for change scenario
    queryInput: String,   // raw input for query scenario
    adviseInput: String,  // raw input for advise scenario
  },
  auditLog: [             // array of audit entries
    {
      action: 'CREATE' | 'CHANGE' | 'CLOSE',
      object: String,
      id: String,
      ts: String,         // time string
      user: String,
      source: String,
    }
  ],
  pendingIO: String | null,  // IO ID after creation
}
```

---

## 5. Scenario Step Schema

```javascript
// Each step in a scenario follows this structure:
{
  joule: () => ResponseObject,   // function returning response data
  match: RegExp,                 // (unused in current impl — for future NLU)
  before: (input: string) => void  // optional hook to extract/store data
}

// ResponseObject schema:
{
  text: String,                  // HTML string displayed as message
  table?: {
    rows: Array<Array<String>>,  // 2D array of cell values
    header?: Boolean,            // if true, first row is <th>
  },
  steps?: Array<{
    icon: String,                // emoji or icon
    text: String,
  }>,
  workflow?: Array<{
    label: String,
    done: Boolean,
  }>,
  confirmBox?: {
    items: Array<[String, String]>,  // [label, value] pairs
    instruction: String,
  },
  successCard?: {
    ioId: String,
    description: String,
    budget: String,
    manager: String,
  },
  quickReplies?: Array<String>,
  manualForm?: Boolean,
  extraText?: String,
}
```

---

## 6. Integration Architecture (Production Target)

```
┌──────────────────────────────────────────────────────────────────┐
│  MDA-IO Frontend (React / SAP Fiori)                             │
└──────────────────────────┬───────────────────────────────────────┘
                           │ HTTPS / WebSocket
                           ▼
┌──────────────────────────────────────────────────────────────────┐
│  API Gateway (SAP BTP / API Management)                          │
│  • Authentication: OAuth 2.0 / SAML                              │
│  • Rate limiting, logging, SSL termination                       │
└────────────┬─────────────────────────────────────────────────────┘
             │
    ┌────────┴──────────────────────────────────┐
    │                                           │
    ▼                                           ▼
┌──────────────────┐                 ┌────────────────────────┐
│  Agent Service   │                 │  Notification Service  │
│  (SAP BTP CF)    │                 │  (SAP Business         │
│                  │                 │   Workplace / Email)   │
│  • NLU Engine    │                 └────────────────────────┘
│  • Scenario Mgr  │
│  • Validation    │
│  • Approval Wf   │
│  • Audit Logger  │
└────────┬─────────┘
         │
    ┌────┴─────────────────────────────────┐
    │                                      │
    ▼                                      ▼
┌──────────────────────┐       ┌────────────────────────────┐
│  SAP S/4HANA         │       │  Master Data Repository    │
│  OData V4 APIs       │       │  (HANA DB)                 │
│                      │       │                            │
│  • KO01 (Create IO)  │       │  • IO snapshots            │
│  • KO02 (Change IO)  │       │  • Historical data         │
│  • KO04 (Close IO)   │       │  • Analytics dataset       │
│  • S_ALR_87012993    │       └────────────────────────────┘
│    (IO Report)       │
│  • BAPI_INTERNALORDER│
│    _CREATE           │
│  • BAPI_INTERNALORDER│
│    _CHANGE           │
└──────────────────────┘
```

---

## 7. Security & Authorization Flow

```
User Request
    │
    ▼
┌──────────────────────────────────────┐
│  AUTHENTICATION                       │
│  • SAP Identity Authentication       │
│  • SSO via SAML / OAuth 2.0          │
└──────────────────┬───────────────────┘
                   │
                   ▼
┌──────────────────────────────────────┐
│  AUTHORIZATION (SAP Role Check)      │
│  • Object: CO_AUFNR                  │
│  • Activity: 01 (Create)             │
│              02 (Change)             │
│              03 (Display)            │
│              05 (Lock/Close)         │
│  • Company Code: DE01                │
│  • Controlling Area: A100            │
└──────────────────┬───────────────────┘
                   │
          ┌────────┴────────┐
        Authorized       Unauthorized
          │                  │
          ▼                  ▼
    Continue            Return 403
    processing          + message to
                        user
```
