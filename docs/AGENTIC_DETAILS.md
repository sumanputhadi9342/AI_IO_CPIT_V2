# Agentic Architecture & Requirements
## Master Data Agent — Internal Order (MDA-IO)

---

## 1. What Makes MDA-IO an AI Agent (Not Just a Chatbot)

A traditional chatbot follows a fixed script: if user says X, respond with Y. MDA-IO is an **AI agent** — it perceives its environment, reasons about context, plans multi-step actions, executes those actions in an external system (S/4HANA), and monitors outcomes over time.

The key distinguishing characteristics:

| Characteristic | Traditional Chatbot | MDA-IO Agent |
|---|---|---|
| **Perception** | Reads user text only | Reads user text + IO portfolio state + S/4HANA data |
| **Memory** | None (stateless) | Maintains session state, audit log, pending IOs |
| **Reasoning** | Pattern match → fixed response | Analyzes context, impact, risk, policy |
| **Planning** | N/A | Multi-step orchestration (validate → approve → confirm → execute) |
| **Action** | Generates text only | Writes to S/4HANA, triggers workflows, sends notifications |
| **Monitoring** | N/A | Proactively watches 2,400 IOs for budget/expiry/activity/governance conditions |
| **Adaptation** | None | Adjusts recommendation based on historical org data |

---

## 2. Agent Architecture

```
┌─────────────────────────────────────────────────────────────────────┐
│                    AGENT CORE                                        │
│                                                                     │
│  ┌────────────┐    ┌────────────┐    ┌────────────┐                │
│  │ PERCEIVE   │───▶│  REASON    │───▶│    ACT     │                │
│  │            │    │            │    │            │                │
│  │ • User NL  │    │ • Intent   │    │ • Execute  │                │
│  │ • IO state │    │ • Context  │    │   in SAP   │                │
│  │ • S/4HANA  │    │ • Risk     │    │ • Trigger  │                │
│  │   data     │    │ • Policy   │    │   workflow │                │
│  │ • Alert    │    │ • Plan     │    │ • Notify   │                │
│  │   triggers │    │   steps    │    │ • Log      │                │
│  └────────────┘    └────────────┘    └────────────┘                │
│                           │                                         │
│                    ┌──────▼──────┐                                  │
│                    │   MEMORY    │                                  │
│                    │             │                                  │
│                    │ • Session   │                                  │
│                    │   state     │                                  │
│                    │ • Audit log │                                  │
│                    │ • IO data   │                                  │
│                    │   cache     │                                  │
│                    └─────────────┘                                  │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 3. Agent Capabilities

### 3.1 Perception

The agent perceives inputs from three sources:

**User Input:**
- Free-text natural language via chat interface
- Structured input via quick-reply buttons (pre-defined options)
- Form submissions (manual IO entry)

**Environmental State (S/4HANA):**
- Real-time IO master data from AUFK table
- Budget consumption data from COSP/COSS + BPGE
- Manager validity from HR master data (PA0001)
- Cost Center status from CSKS
- Open commitments from EKKN/COBK

**System Triggers (Proactive):**
- Budget threshold breach (75% / 90%)
- Expiry countdown (30 days / 7 days)
- Activity inactivity (90 days / 180 days)
- Governance rule violation (CP-007)

### 3.2 Reasoning & Planning

The agent applies reasoning at multiple levels:

**Intent Classification:**
```
"Create" | "Change" | "Close" | "Query" | "Advise"
```
Matched via pattern recognition (regex in v1.0, LLM-based NLU in v2.0).

**Context Extraction (before() hooks):**
```javascript
// Example: Extract manager and budget from user input
before: (input) => {
  if (/anna/i.test(input))  state.data.manager = 'Anna Mueller';
  if (/michael/i.test(input)) state.data.manager = 'Michael Weber';
  const m = input.match(/(\d[\d,]*)/);
  if (m) state.data.budget = 'EUR ' + parseInt(m[1]).toLocaleString();
}
```

**Policy Reasoning:**
- Budget > EUR 250,000 → approval required
- Budget increase > 50% → escalated approval required
- Manager inactive → governance alert + block
- No postings > 90 days → activity alert
- Remaining budget > EUR 100,000 on close → governance approval

**Impact Analysis:**
- Delta calculation for budget changes
- Commitment-vs-new-budget comparison
- Burn rate projection (remaining budget ÷ monthly average)
- Days-to-depletion estimation

**Recommendation Engine (Advise Mode):**
- Multi-dimensional scoring across 5 axes
- Priority assignment based on urgency + risk combination
- Estimated financial impact calculation (budget release)

### 3.3 Memory

The agent maintains session-scoped state:

```javascript
state = {
  scenario: String,        // active workflow
  step: Number,            // position within workflow
  data: {                  // extracted and derived values
    manager, budget, desc, costCenter,
    companyCode, validFrom, validTo, orderType,
    changeInput, queryInput, adviseInput,
    manualEntry
  },
  auditLog: Array,         // all write actions this session
  pendingIO: String        // IO ID after creation
}
```

**Cross-session memory (production):**
- IO creation history per user
- Past recommendations and outcomes
- User preference profile (preferred company codes, cost centers)
- Organizational context (who manages which cost centers)

### 3.4 Action Execution

The agent executes actions with a mandatory pre-execution gate:

```
Plan Action
    │
    ▼
Validate (8 checks)
    │
    ▼
Seek Approval (if required)
    │
    ▼
Confirmation Gate (explicit phrase)
    │
    ▼
Execute in S/4HANA (BAPI / OData)
    │
    ▼
Log to Audit Trail
    │
    ▼
Post-Execute Actions (notify, sync, monitor)
```

**Execution safety principles:**
- No action is taken without explicit user confirmation
- Destructive actions (Close) carry an irreversibility warning
- All writes are logged before execution
- Failed execution rolls back and surfaces error to user

---

## 4. Multi-Step Orchestration

MDA-IO orchestrates complex multi-step processes that span multiple turns:

### Create IO — 10-Step Orchestration

| Turn | Agent Role | User Role |
|---|---|---|
| 1 | Greet, offer options | Express intent |
| 2 | Ask for preference | Suggest or manual |
| 3 | AI suggestions / form | Select manager & budget |
| 4 | Review summary | Confirm or edit |
| 5 | Run validations | (automatic) |
| 6 | Show auth + trigger approval | (automatic) |
| 7 | Simulate/await approval | Click simulate |
| 8 | Show confirmation gate | Type "Confirm Create" |
| 9 | Execute + show success | (automatic) |
| 10 | Activate monitoring | Next action |

### Approval Workflow Orchestration

```
Agent determines approval requirements
           │
           ▼
Agent builds approver list based on thresholds
           │
           ▼
Agent sends approval requests (in parallel via BTP workflow)
           │
    ┌──────┴──────┐
    │             │
 All         Any
 approve     reject
    │             │
    ▼             ▼
Proceed to   Notify requester,
confirm      ask to revise
```

### Bulk Action Orchestration (Advise Mode)

```
User: "Take all recommended actions"
           │
           ▼
Agent queues 5 actions in parallel:
  ├── IO-755021: Pre-close → approval workflow
  ├── IO-742300: Pre-close → approval workflow
  ├── IO-748832: Pre-close → approval workflow
  ├── IO-765881: Budget review → Finance Governance notification
  └── IO-759044: Manager reassignment → change workflow
           │
           ▼
Agent reports status for each action
Expected completion: 2–3 business days
```

---

## 5. Proactive Monitoring Agent

The monitoring agent runs continuously as a background process, independent of user interaction:

```
MONITORING LOOP (runs every 24 hours)
           │
           ▼
Fetch all ACTIVE Internal Orders (AUFK WHERE ISTAT = 'REL')
           │
           ▼
For each IO:
  ├── Check 1: budget_consumed / budget_total ≥ 0.75 ?
  │   → Trigger BUDGET_ALERT
  │
  ├── Check 2: (valid_to - today) ≤ 30 days ?
  │   → Trigger EXPIRY_ALERT
  │
  ├── Check 3: (today - last_posting_date) ≥ 90 days ?
  │   → Trigger ACTIVITY_ALERT
  │
  └── Check 4: manager employee status ≠ ACTIVE ?
      → Trigger GOVERNANCE_ALERT
           │
           ▼
For each triggered alert:
  ├── Check: Already alerted in last 7 days?
  │   → Skip (prevent alert flooding)
  │
  └── Surface in UI:
      ├── Banner across top of interface
      ├── Sidebar badge count increment
      └── Chat message with detail + action buttons
```

### Alert Deduplication
- Each alert type per IO is limited to once per 7-day window
- Budget alerts fire at 75% threshold once, then again at 90%
- Expiry alerts fire at 30 days, then at 7 days
- Activity alerts fire at 90 days, then at 180 days with escalated priority

---

## 6. Safety & Governance Design

MDA-IO is designed around four safety principles:

### Principle 1: Human-in-the-Loop for All Write Actions
No data is written to S/4HANA without explicit user confirmation. The agent:
1. Shows exactly what it plans to do (confirmation box)
2. Requires the user to type a specific confirmation phrase
3. Warns when actions are irreversible

### Principle 2: Validation Before Action
Every create/change/close passes through the validation engine before an approval workflow is triggered. Invalid data never reaches the approval stage.

### Principle 3: Authorization Enforcement
Every action checks:
- User's SAP authorization object (CO_AUFNR with appropriate activity)
- Company code and controlling area scope
- Budget threshold limits for auto-approval vs. escalation

### Principle 4: Complete Audit Trail
Every agent-initiated write action creates:
- An audit log entry in the MDR (timestamp, user, action, before/after values)
- A change document in S/4HANA (standard SAP audit mechanism)
- An approval record linking the action to all approvers and timestamps

---

## 7. Agentic Design Patterns Used

### Pattern 1: Guided Autonomy
The agent is autonomous in information gathering, analysis, and orchestration — but delegates decisions to the human at critical junctures (manager selection, budget confirmation, close confirmation).

### Pattern 2: Progressive Disclosure
Information is revealed step by step, preventing cognitive overload. The agent shows what is needed at each step, not everything at once.

### Pattern 3: Fail-Safe Defaults
- If validation fails → stop and explain, never submit partial data
- If approval is rejected → return to user with rejection reason, offer to revise
- If execution fails → surface error, preserve state, allow retry

### Pattern 4: Proactive Monitoring (Push vs. Pull)
Instead of requiring users to log in and check dashboards, the agent pushes alerts to the user when conditions are met. This shifts the paradigm from reactive (check reports) to proactive (agent notifies).

### Pattern 5: Contextual Recommendations
Recommendations are generated in the context of the user's organization — using actual IO history, manager workloads, cost center hierarchies, and budget consumption patterns. Not generic advice.

### Pattern 6: Deterministic Confirmation Gates
Even in an agentic system, certain actions require exact phrase confirmation. This prevents accidental execution and ensures the user has read and understood the proposed action.

---

## 8. Production LLM Integration (v2.0 Target)

In the current v1.0 implementation, intent and context extraction are rule-based (regex). In v2.0, the agent will integrate a production LLM:

### LLM Role
| Function | v1.0 | v2.0 (LLM) |
|---|---|---|
| Intent classification | Regex matching | LLM classification with confidence score |
| Entity extraction | Regex (budget, manager name) | LLM NER (any field, any format) |
| Recommendation generation | Hard-coded rules | LLM analysis of full portfolio context |
| Natural language output | Template-based | LLM-generated prose responses |
| Ambiguity resolution | Ignored / default | LLM asks clarifying question |
| Multi-language support | English only | All SAP supported languages |

### Recommended Model Configuration (v2.0)
```
Provider: Anthropic
Model: claude-sonnet-4-6 (claude-sonnet-latest)
Role: Agent core — intent, extraction, recommendation generation
Context: IO master data summary, user profile, org hierarchy
Max tokens: 4,096 (response) / 32,768 (context window)
Temperature: 0.1 (deterministic for governance actions)
```

### System Prompt Design (v2.0)
```
You are the Master Data Agent for SAP Internal Orders at [Company Name].
You help authorized users create, change, close, and analyze Internal Orders
in SAP S/4HANA (Company Code DE01, Controlling Area A100).

Your core responsibilities:
1. Extract user intent and relevant entities from natural language
2. Guide users through multi-step governance workflows
3. Validate all data against SAP master data and corporate policies
4. Never execute a write action without explicit user confirmation
5. Maintain complete audit trail for all actions

You must NEVER:
- Create, change, or close an Internal Order without "Confirm [Action]" from user
- Suggest values without checking current organizational master data
- Bypass validation checks under any circumstances
- Take bulk actions without per-action confirmation

Current user: {user_name} ({sap_user_id})
Authorization scope: {company_codes} / {controlling_areas}
Active IOs in portfolio: {io_count}
```

---

## 9. Agent Metrics & Observability

| Metric | Definition | Target |
|---|---|---|
| Intent accuracy | % of user inputs correctly classified | > 97% |
| Validation pass rate | % of IO submissions passing all 8 checks on first attempt | > 90% |
| Approval cycle time | Average time from submission to all approvals | < 4 hours |
| Alert response rate | % of proactive alerts actioned within 48 hours | > 75% |
| Confirmation abandonment | % of users who reach confirmation gate but don't confirm | < 10% |
| End-to-end task completion | % of initiated create/change/close flows fully completed | > 85% |
| False positive alerts | Alerts triggered that required no action | < 5% |
| Audit coverage | % of write actions with complete audit record | 100% |
