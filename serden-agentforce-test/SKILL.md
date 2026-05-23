---
name: serden-agentforce-test
description: >-
  SE scenario testing playbook for Agentforce agents built from production experience.
  Covers demo script creation, sf agent preview workflow, two-phase validation, session
  trace analysis, Salesforce record verification, and test data cleanup.
  Use when writing a demo script, running SE scenario tests, interpreting agent preview
  output, reading session traces, or validating published agent behaviour.
  Pairs with /testing-agentforce and /observing-agentforce.
disable-model-invocation: true
---

# Agentforce Test Playbook

Lessons from scenario testing a production sales and leasing agent as a Solution Engineer.

---

## Part 1 — Test Spec and Demo Script

### 1.1 Locate or create the test spec first

The test spec may live anywhere: a Slack canvas, Confluence, a Google Doc, a README, or a YAML file. Before running any tests:

1. Ask the user where the spec lives or read the file if a path is provided
2. If **no spec exists**, create one — see §1.2 below
3. Extract from whatever source you find:
   - The 4–6 discrete scenarios (personas, intents, expected outcomes)
   - The happy-path conversation flow for each
   - The Salesforce records that should be created/updated and their expected field values

Structure each scenario as: **Persona → Utterances → Expected Salesforce state**. This maps directly to the `sf agent preview` workflow in Part 2.

### 1.2 Creating a demo script (when no spec exists)

Write a **presales demo script** that a Solution Engineer can run live during a customer presentation. Save it as a markdown file in the project.

#### The four scenario types for a sales/service agent

| # | Scenario type | Purpose |
|---|--------------|---------|
| 1 | **New visitor — qualify & search** | Show NLU, product/service search, qualification capture |
| 2 | **High-intent visitor — direct to action** | Show booking/order flow, record creation |
| 3 | **Power user — multi-criteria** | Show complex search, comparison, no action needed |
| 4 | **Sensitive data path** | Show identity verification gate, financial/account data |

#### Demo script template

```markdown
# [Agent Name] — SE Demo Script

## What this agent does
[One paragraph. Business problem, personas served, key capabilities.]

---

## Before you start
- Confirm agent is published and active: `sf agent preview start --json --api-name <Name>`
- Have Salesforce open in another tab for record verification
- Clean test data if needed (see §3.2)

---

## Scenario 1 — [Name] ([Persona name])
**What this demonstrates:** [One sentence]

**Utterances (send in order):**
1. "[opening message]"
2. "[follow-up message]"
3. "[closing message]"

**What to watch for:**
- Agent [specific AI behaviour to highlight]
- [Salesforce record created/updated — verify with sf data query]

**Expected outcome:** [record created / data returned / gate triggered]

---
[Repeat for scenarios 2–4]
```

#### SE instruction principles
- Every utterance is exact text — no paraphrasing
- Mark which turns are "AI moment" highlights
- Keep each scenario to 4–6 turns maximum
- Include the `sf data query` command to verify Salesforce state after each scenario

---

## Part 2 — Running Scenarios with `sf agent preview`

### 2.1 Use `sf agent preview` — not browser or curl

**Always use `sf agent preview` for scenario testing.** It is the right tool because:
- It tests the **same runtime** as the browser widget when using `--api-name` (published agent)
- It tests the **local working copy before publish** when using `--authoring-bundle --use-live-actions`
- It writes **session trace files** for every turn — the only way to see what actually happened inside the agent

**Do NOT use curl or browser testing to validate agent behaviour.** Both give you only the response text and are blind to topic routing, variable state, action inputs/outputs, and grounding failures.

**curl has one legitimate use:** smoke-testing that the Agent API connection is healthy. Nothing more.

### 2.2 Two-phase validation: authoring bundle first, published agent second

Never validate only one path. Always run both phases.

**Phase 1 — before publish (catches issues before they're locked in):**
```bash
sf agent preview start --json --use-live-actions --authoring-bundle <AgentName>
```
This tests the local `.agent` file with live Flows. Trace files are written. Fix anything that fails here before publishing.

**Phase 2 — after publish (same path as browser widget):**
```bash
sf agent preview start --json --api-name <AgentName>
```
This tests the published active version — identical to what the React widget calls. Any divergence between Phase 1 and Phase 2 means a publish-time transformation changed agent behaviour.

### 2.3 Which preview mode to use

| Situation | Command |
|---|---|
| Testing before publish (active development) | `sf agent preview start --json --use-live-actions --authoring-bundle <Name>` |
| Testing the published/active agent | `sf agent preview start --json --api-name <Name>` |
| Diagnosing production reports (reproduction step) | `sf agent preview start --json --api-name <Name>` |

### 2.4 Critical: `--use-live-actions` goes on `start` only — NOT on `send`

`--use-live-actions` is a valid flag **only** on `sf agent preview start`. Passing it on `send` returns `Error: Nonexistent flag: --use-live-actions`.

```bash
# CORRECT
sf agent preview start --json --use-live-actions --authoring-bundle MyAgent
sf agent preview send  --json --authoring-bundle MyAgent --session-id $S -u "message"

# WRONG — send rejects the flag
sf agent preview send --json --use-live-actions --authoring-bundle MyAgent --session-id $S -u "message"
```

### 2.5 Critical: `--api-name` sessions write empty trace files

When using `--api-name`, trace files contain **empty `{}`** — 2 bytes each. This is a platform limitation: the published runtime does not write local traces.

**Populated trace files are only written by `--authoring-bundle` sessions.**

Always run Phase 1 (`--authoring-bundle --use-live-actions`) to get traces before running Phase 2 (`--api-name`) for confirmation.

### 2.6 Full multi-turn scenario workflow

```bash
# 1. Start session
SESSION=$(sf agent preview start --json --api-name <AgentName> \
  | python3 -c "import sys,json; print(json.load(sys.stdin)['result']['sessionId'])")

# 2. Send each utterance, capturing planId
RESP=$(sf agent preview send --json --api-name <AgentName> \
  --session-id $SESSION -u "First utterance")
PLAN_ID=$(echo $RESP | python3 -c "
import sys,json,re
raw=sys.stdin.read()
clean=re.sub(r'[\x00-\x08\x0b\x0c\x0e-\x1f]','',raw)
d=json.loads(clean)
msgs=d.get('result',{}).get('messages',[])
print(msgs[-1].get('planId','') if msgs else '')
")

sf agent preview send --json --api-name <AgentName> \
  --session-id $SESSION -u "Second utterance"

# 3. End session — writes trace files
TRACES=$(sf agent preview end --json --api-name <AgentName> \
  --session-id $SESSION \
  | python3 -c "import sys,json; print(json.load(sys.stdin)['result']['tracesPath'])")

# 4. Read trace for a specific turn
TRACE="$TRACES/$PLAN_ID.json"
```

---

## Part 3 — Reading Traces and Verifying State

### 3.1 Reading traces after each scenario

```bash
# Topic routing — what topic was entered
python3 -c "
import json
trace = json.load(open('$TRACE'))
for step in trace.get('plan', []):
    if step['type'] == 'NodeEntryStateStep':
        print('Topic:', step['data'].get('agent_name'))
"

# Variable state at start of turn
python3 -c "
import json
trace = json.load(open('$TRACE'))
for step in trace.get('plan', []):
    if step['type'] == 'TransitionStep':
        state = step.get('data', {}).get('current_state', {})
        print({k: v for k, v in state.items() if not k.startswith('AgentScript')})
        break
"

# Action execution — inputs and outputs
python3 -c "
import json
trace = json.load(open('$TRACE'))
for step in trace.get('plan', []):
    if step['type'] == 'ActionExecutionStep':
        print('Action:', step['data'].get('action_name'))
        print('Inputs:', step['data'].get('inputs'))
        print('Outputs:', step['data'].get('outputs'))
"

# Final response text
python3 -c "
import json
trace = json.load(open('$TRACE'))
for step in trace.get('plan', []):
    if step['type'] == 'PlannerResponseStep':
        print(step.get('message',''))
"
```

### 3.2 Verify Salesforce state explicitly after each scenario

Don't trust the agent response text alone. Query the records that should have been created or updated:

```bash
sf data query --json \
  --query "SELECT Id, FirstName, Email, LeadSource FROM Lead WHERE Email = 'test@example.com'"

# Check linked Events if the agent books appointments
sf data query --json \
  --query "SELECT Id, Subject, ActivityDateTime, WhoId FROM Event WHERE WhoId = '<leadId>'"
```

### 3.3 Clean test data between runs

Stale records cause update paths to run when create paths are expected:

```bash
sf data query --json \
  --query "SELECT Id FROM Lead WHERE Email IN ('a@test.com','b@test.com')" \
  | python3 -c "
import sys,json,subprocess
ids=[r['Id'] for r in json.load(sys.stdin)['result']['records']]
for rid in ids:
    subprocess.run(['sf','data','delete','record','--json','--sobject','Lead','--record-id',rid])
"
```

### 3.4 Structure scenarios to trigger each code path

Each scenario must exercise a distinct topic branch. Always include:
- One happy-path per domain topic
- One multi-turn scenario that crosses a gate (e.g. OTP verification)
- One off-topic guardrail utterance

### 3.5 Use `/observing-agentforce` for production issues after go-live

Production failures surface in STDM session traces in Data Cloud. Use `/observing-agentforce` when a user reports behaviour not reproducible in preview.

---

## Scenario testing checklist

- [ ] Demo script exists with step-by-step utterances for all 4 scenarios
- [ ] Phase 1 (`--authoring-bundle --use-live-actions`) passes for all scenarios
- [ ] Session trace `FunctionStep` shows correct inputs and outputs for each action
- [ ] Salesforce records verified post-scenario via `sf data query`
- [ ] Test Lead records deleted between runs
- [ ] Phase 2 (`--api-name`) confirms behaviour matches Phase 1
- [ ] Off-topic guardrail utterance tested
