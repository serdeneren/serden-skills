---
name: serden-agentforce-build-and-test
description: >-
  Agentforce build-and-test playbook synthesised from production build experience.
  Covers hard-won gotchas in Agent Script, Salesforce Flow backing logic, SE demo script
  creation, and structured scenario testing. Trigger when: building or debugging an
  Agentforce agent on a Salesforce org, designing Flow backing logic for agent actions,
  creating a presales demo script, or running scenario tests as a solution engineer.
  Pairs with /developing-agentforce and /testing-agentforce skills.
disable-model-invocation: true
---

# Agentforce Build & Test Playbook

Lessons from building a sales and leasing agent end-to-end: a service agent with Flow-backed actions, Lead/Event creation, and structured SE scenario testing.

---

## Part 1 — Developing the Agent

### 1.1 Always use `/developing-agentforce` first

The skill's reference files contain the canonical Agent Script syntax, execution model, and CLI commands. Read them before touching a `.agent` file. The most expensive mistakes in this build came from NOT consulting them early enough.

### 1.2 Numeric action I/O is a trap

`number` is valid for **mutable variables** but **fails at preview/publish** for **action I/O** when the backing Flow variable is `Currency`. The platform validates the type mapping at preview start — not at `validate authoring-bundle` time, so CI will pass but the session won't start.

**Rule:** Any action input that maps to a Flow `Currency` variable must be declared as:
```agentscript
budgetMin: object
    complex_data_type_name: "lightning__currencyType"
```

Read `references/complex-data-types.md` in `/developing-agentforce` for the full decision tree before wiring any numeric action parameter.

### 1.3 LLM reasoning actions ≠ guaranteed execution

Actions placed in `reasoning: actions:` are offered to the LLM as tools. The LLM decides whether and when to call them. Instructions like "call X immediately after Y" are hints, not enforcement. This burned time: `save_lead` was consistently skipped even with explicit prompt wording.

**The fix — use `before_reasoning:` for mandatory, state-triggered logic:**
```agentscript
before_reasoning:
    if @variables.customer_email != "" and @variables.lead_id == "":
        run @actions.create_or_update_lead
            with firstName = @variables.customer_first_name
            with email = @variables.customer_email
            set @variables.lead_id = @outputs.leadId
```

This fires deterministically on every turn where the guard evaluates to true — no LLM involved. Read `references/architecture-patterns.md` → Post-Action Loop in `/developing-agentforce` for the full pattern.

**When does `before_reasoning` see the updated variable?**
`before_reasoning` runs at the START of each turn, before the LLM. If the customer provides their email on turn N (via a reasoning action like `capture_contact`), `before_reasoning` won't see it until turn N+1. Design for this: the Lead is created on the first turn after email is captured, not the same turn.

### 1.4 Restricted and multi-select picklists crash Flow silently then loudly

Salesforce Flow throws `INVALID_OR_NULL_FOR_RESTRICTED_PICKLIST` at runtime if you pass a value that isn't in a restricted picklist. The agent passes LLM-generated text (e.g. `"Multiple"`, `"several projects"`) to Flow inputs mapped to restricted fields. The Flow crashes and the Lead is never created.

**Fix pattern — `fmInterestedProducts` formula with keyword matching:**
```xml
<formulas>
    <name>fmInterestedProducts</name>
    <dataType>String</dataType>
    <expression>IF(
        CONTAINS(LOWER({!interestedProducts}), "product a"), "Product A",
        IF(
            CONTAINS(LOWER({!interestedProducts}), "product b"), "Product B",
            ""
        )
    )</expression>
</formulas>
```

Returning `""` (blank) for unrecognised values is always safer than passing the raw LLM string to a restricted picklist.

Similarly, **multi-select picklists** cannot be directly concatenated in Flow formula strings — they require intermediate text variable assignments before use in `fmSummary`-style formulas. Use an `<assignments>` element to copy picklist field values into `String` variables first.

### 1.5 Validate picklist values in the org before wiring them in Flow

Run `sf data query` or use the describe API to confirm active picklist values before hard-coding them:

```bash
SF_ACCESS_TOKEN=$(sf org display --json --target-org <alias> | python3 -c "import sys,json; print(json.load(sys.stdin)['result']['accessToken'])")
curl -s "$INSTANCE_URL/services/data/v66.0/sobjects/Lead/describe" \
  -H "Authorization: Bearer $SF_ACCESS_TOKEN" | python3 -c "
import sys,json
fields = json.load(sys.stdin)['fields']
for f in fields:
    if f['name'] == 'LeadSource':
        print([v['value'] for v in f['picklistValues'] if v['active']])
"
```

In this project, the org had customised `LeadSource` values — `"Web"` did not exist, `"Website"` did. The Flow was silently assigning an invalid value for months.

### 1.6 Flow DateTime formulas don't have a `TIME()` function

Use `DATETIMEVALUE()` with string concatenation instead:
```xml
<expression>DATETIMEVALUE({!requestedDateStr} &amp; " 09:00:00")</expression>
```
`TIME()` is a spreadsheet function, not a Salesforce Flow formula function. Attempting to use it produces a deployment error.

### 1.7 `GenAiFunctionDefinition` duplication at publish time

If you change an action's `description` between topics (or between agent versions), the platform may attempt to create duplicate `GenAiFunctionDefinition` records. Symptoms: publish succeeds for one action but fails with `duplicate value found` for another.

**Fix:** Ensure the `description` field on each `actions:` definition block is **identical** across all topics that reference the same action. Deactivate then re-publish to clear state if the error persists.

### 1.8 `validate authoring-bundle` doesn't catch everything

The local validator catches syntax and structural errors. It does NOT catch:
- Type mismatches between action I/O and Flow variables (caught at `preview start`)
- Missing permissions on the Einstein Agent User (caught at `preview start`)
- `GenAiFunctionDefinition` duplication (caught at `publish`)
- Invalid picklist values (caught at Flow runtime)

Always run `sf agent preview start --use-live-actions` after validate before publishing. For the full validation gap taxonomy and error-to-root-cause mapping, read `references/agent-validation-and-debugging.md` in `/developing-agentforce`.

### 1.9 Use session traces, not just preview output

`sf agent preview send` returns the agent's text response. That's insufficient for diagnosis. The trace files at `.sfdx/agents/<bundle>/sessions/<id>/traces/<planId>.json` show:
- Which `before_reasoning` actions actually fired (`BeforeReasoningStep`)
- What variables held at the start of the turn (`NodeEntryStateStep`)
- What tools the LLM had available (`EnabledToolsStep`)
- What inputs and outputs a Flow received (`FunctionStep`)

The most common diagnosis error: looking only at the response text and inferring the action didn't fire, when actually it fired but returned `amount: 0` because the dollar sign was stripped from the message.

For the full trace analysis reference — exact step types, jq commands, grounding check patterns — read `references/agent-validation-and-debugging.md` in `/developing-agentforce`. The trace snippet commands in §2.3 of this skill are a practical subset of that reference.

### 1.10 `filter_from_agent: True` hides output from the LLM — including for gate logic

`filter_from_agent: True` on an action output means the LLM **cannot see that value**. It is hidden from the LLM's context even though it is available for `set` assignments. This is correct for internal routing signals — but it is fatal for any output the LLM needs to reason about or act on.

**The bug this caused:** An OTP verification action had `isValid: boolean` with `filter_from_agent: True`. The LLM called the action and the `set` updated `@variables.is_verified` correctly. But the LLM could not see `isValid` — so its post-action instructions ("if isValid is True, say verified; if False, say wrong code") were unreachable. The LLM received no signal about the outcome and defaulted to assuming success regardless of what the flow returned.

**Rule:** Use `filter_from_agent: True` only for outputs that are **never** mentioned in instructions and are **only** used in `available when` guards or `set` assignments in deterministic blocks. If the LLM needs to branch based on a value — show the agent a success/failure message, choose different response text — set `filter_from_agent: False`.

```agentscript
# WRONG — LLM cannot see isValid, cannot respond correctly to failure
outputs:
    isValid: boolean
        filter_from_agent: True

# CORRECT — LLM can read isValid and compose the right response
outputs:
    isValid: boolean
        filter_from_agent: False
```

### 1.11 Same-turn routing hazard when a gate transitions back to `topic_selector`

When a gate topic (e.g. `otp_verification`) succeeds and uses `after_reasoning` to transition to `topic_selector`, **both topics execute in the same user turn**. The hub fires against the gate's triggering message — e.g. the string `"1234"` — not a fresh utterance. Since `"1234"` matches no domain topic, the hub routes it to `off_topic` or `ambiguous_question`, producing a confusing response.

**Fix — two parts:**

1. Add a `go_to_otp_verification` transition in `topic_selector` so numeric/code-like inputs are correctly routed to the verification topic rather than misclassified.

2. Add a conditional instruction in `topic_selector` that fires when verification just completed:

```agentscript
start_agent topic_selector:
    reasoning:
        instructions: ->
            if @variables.is_verified == True:
                | Your identity has been verified. How can I help you?

            if @variables.is_verified == False:
                | [normal routing instructions...]
        actions:
            go_to_otp_verification: @utils.transition to @topic.otp_verification
                description: "User is entering a verification code"
            [... other transitions ...]
```

The `is_verified == True` branch fires on the same turn the gate completes (because `is_verified` was just set by the gate's `set` assignment). The hub greets the user and asks what they need — the next user utterance is then routed normally.

This pattern applies any time a gate topic transitions to a router via `after_reasoning`. It is documented as an anti-pattern in `/developing-agentforce` `references/agent-script-core-language.md` Section 12.

### 1.12 Variable type rules — quick reference

| Context | Allowed types |
|---------|--------------|
| Mutable variable | `string`, `number`, `boolean`, `object`, `date`, `id`, `list[T]` |
| Action I/O (Currency Flow var) | `object` + `complex_data_type_name: "lightning__currencyType"` |
| Action I/O (Number Flow var) | `number` |
| Action I/O (Date Flow var) | `date` |
| Action I/O (Apex Integer/Long) | `integer` / `long` (action params only — NOT mutable vars) |

---

## Part 2 — Testing the Agent (SE approach)

### 2.1 Locate or create the test spec first

The test spec may live anywhere: a Slack canvas, a Confluence page, a Google Doc, a README, or a YAML file. Before running any tests:

1. Ask the user where the spec lives or read the file if a path is provided
2. If **no spec exists**, create one — see §2.2 below
3. Extract from whatever source you find:
   - The 4–6 discrete scenarios (personas, intents, expected outcomes)
   - The happy-path conversation flow for each
   - The Salesforce records that should be created/updated and their expected field values

Structure each scenario as: **Persona → Utterances → Expected Salesforce state**. This maps directly to the `sf agent preview` workflow in §2.3.

### 2.2 Creating a demo script (when no spec exists)

When there is no test spec, create one structured as a **presales demo script** that a Solution Engineer can run live during a customer presentation. Write it as a markdown file saved in the project.

#### Demo script structure

Each script must contain:
- A one-paragraph agent overview (what it does, what business problem it solves)
- 4 scenarios, each targeting a distinct agent capability
- Step-by-step utterances per scenario (exact text to send, no paraphrasing)
- Expected outcomes per scenario
- A "what to watch for" callout per scenario highlighting the AI behaviour

#### The four scenario types for a sales/service agent

Design scenarios to cover these four branches — adapt names and personas to the specific agent:

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
- Clean test data if needed (see §2.5)

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

### 2.3 Run scenarios with `sf agent preview` — not browser or curl

**Always use `sf agent preview` for scenario testing.** It is the right tool because:
- It tests the **same runtime** as the browser widget when using `--api-name` (published agent)
- It tests the **local working copy before publish** when using `--authoring-bundle --use-live-actions`
- It writes **session trace files** for every turn — the only way to see what actually happened inside the agent

**Do NOT use curl or browser testing to validate agent behaviour.** curl gives you only the response text. Browser testing gives you no programmatic access to traces. Both are blind to topic routing, variable state, action inputs/outputs, and grounding failures.

**curl has one legitimate use:** smoke-testing that the Agent API connection is healthy (correct token format, `sfap_api` scope, proxy reachability). Nothing more.

For exact `sf agent preview` command syntax, flags, and output format, read `references/salesforce-cli-for-agents.md` Section 9 in `/developing-agentforce`. For trace file locations and step-type reference, read `references/agent-validation-and-debugging.md` in the same skill.

#### Which preview mode to use

| Situation | Command |
|---|---|
| Testing before publish (active development) | `sf agent preview start --json --use-live-actions --authoring-bundle <Name>` |
| Testing the published/active agent (same path as browser widget) | `sf agent preview start --json --api-name <Name>` |
| Diagnosing production reports (reproduction step) | `sf agent preview start --json --api-name <Name>` |

#### Full multi-turn scenario workflow

```bash
# 1. Start session
SESSION=$(sf agent preview start --json --api-name <AgentName> \
  | python3 -c "import sys,json; print(json.load(sys.stdin)['result']['sessionId'])")

# 2. Send each utterance in sequence, capturing planId from each response
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

#### Reading traces after each scenario

After ending a session, read the trace for each `planId` to verify:

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

### 2.4 Verify Salesforce state explicitly after each scenario

Don't trust the agent response text alone. After each scenario, query the records that should have been created or updated:

```bash
sf data query --json \
  --query "SELECT Id, FirstName, Email, LeadSource FROM Lead WHERE Email = 'test@example.com'"

# Check linked Events if the agent books appointments
sf data query --json \
  --query "SELECT Id, Subject, ActivityDateTime, WhoId FROM Event WHERE WhoId = '<leadId>'"
```

### 2.5 Clean test data between runs

Stale records cause update paths to run when create paths are expected. Always delete test records before re-running scenarios:

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

### 2.6 Two-phase validation: authoring bundle first, published agent second

Never validate only one path. Always run both:

**Phase 1 — before publish (catches issues before they're locked in):**
```bash
sf agent preview start --json --use-live-actions --authoring-bundle <AgentName>
```
This tests the local `.agent` file with live Flows. Trace files are written. Fix anything that fails here before publishing. This is the validation step that `/developing-agentforce` requires before proceeding to publish — see the "Validate behavior" step in the Create an Agent workflow.

**Phase 2 — after publish (same path as browser widget):**
```bash
sf agent preview start --json --api-name <AgentName>
```
This tests the published active version — identical to what the React widget calls. Any divergence between Phase 1 and Phase 2 means a publish-time transformation changed agent behaviour. If Phase 2 diverges, use the Diagnose Behavioral Issues workflow in `/developing-agentforce` to isolate the root cause.

### 2.7 Structure scenario tests to trigger each code path

Each scenario must exercise a distinct topic branch. Always include:
- One happy-path per domain topic
- One multi-turn scenario that crosses a gate (e.g. OTP verification)
- One off-topic guardrail utterance

### 2.8 Use `/observing-agentforce` for production issues after go-live

Production failures surface in STDM session traces in Data Cloud. Use `/observing-agentforce` when a user reports behaviour not reproducible in preview.

---

## Part 3 — Debugging Flow errors

### 3.1 How to surface Flow errors from a running agent

When a Flow-backed action fails silently (agent says something generic, no Lead/Event created), check the flow error in Setup:

**In Salesforce Setup:**
1. Go to Setup → Flows → find the failing Flow → click **View Details**
2. Or: Setup → search **"Paused and Failed Flow Interviews"** → filter by Flow name

**Via debug logs (most reliable):**
1. In Setup → Debug Logs, add the **Einstein Agent User** (e.g. `my_agent@...ext`) as a traced entity
2. Set log level: Workflow = FINEST, Apex = DEBUG
3. Re-run the failing utterance via `sf agent preview send` (see §2.3 — not via the browser)
4. Open the log — search for `FLOW_ELEMENT_ERROR` or the Flow name
5. The error message will be on the same line as the element that failed

**Via anonymous Apex:**
```apex
List<FlowInterview> interviews = [
    SELECT Id, CurrentElement, InterviewLabel, Status
    FROM FlowInterview
    WHERE Status = 'Error'
    ORDER BY CreatedDate DESC
    LIMIT 10
];
for (FlowInterview fi : interviews) {
    System.debug(fi.InterviewLabel + ' | ' + fi.CurrentElement + ' | ' + fi.Status);
}
```

### 3.2 Common Flow error patterns and fixes

| Error message | Root cause | Fix |
|--------------|-----------|-----|
| `INVALID_OR_NULL_FOR_RESTRICTED_PICKLIST: [FieldName]: bad value for restricted picklist field: [value]` | LLM passed free-text to a restricted picklist input | Add a formula (`fmFieldName`) that maps keywords to valid picklist API values; return `""` for unrecognised input |
| `REQUIRED_FIELD_MISSING: Required fields are missing: [LastName]` | Flow tried to create a record without a mandatory field | Add a formula fallback: `IF(ISBLANK({!lastName}), IF(ISBLANK({!firstName}), "Unknown", {!firstName}), {!lastName})` |
| `FIELD_FILTER_VALIDATION_EXCEPTION` | Filter logic references a field that returned null | Add `assignNullValuesIfNoRecordsFound = true` on record lookup elements |
| `Error element [name] (FlowRecordCreate). ... bad value for restricted picklist` | Same as row 1 — affects create path | Add the formula guard on the `inputAssignments` for that field |
| `Compile error: Unknown function TIME` | Flow formula used `TIME()` which doesn't exist | Replace with `DATETIMEVALUE({!dateStr} & " 09:00:00")` |
| Multi-select picklist field used directly in formula string concat | Flow formulas can't directly concatenate picklist values | Use an `<assignments>` element to copy picklist value into a `String` variable first, then reference that variable in the formula |

### 3.3 The fastest debug cycle for Flow errors

```
1. Reproduce via sf agent preview (--use-live-actions --authoring-bundle) →
   note the turn where agent gives a generic response, capture the planId
2. Read the ActionExecutionStep / FunctionStep in the trace to see exact Flow inputs
3. Open Debug Logs → search FLOW_ELEMENT_ERROR → read the element name and error
4. Open the Flow XML in the project → find the element by name
5. Apply the fix from §3.2 table
6. Deploy: sf project deploy start --json --metadata Flow:<FlowName>
7. Re-run the exact same utterance in a new preview session
8. Confirm the trace shows correct outputs and no FLOW_ELEMENT_ERROR in debug logs
```

Do NOT publish a new agent version just to fix a Flow bug — Flow changes take effect immediately on deploy without republishing.

**Why `sf agent preview` beats the chat widget for Flow debugging:**
The trace `ActionExecutionStep` shows the exact inputs the agent sent to the Flow. The chat widget shows only the final response text — you have to guess what the agent passed. The trace eliminates that guesswork entirely.

---

## Part 4 — Salesforce Admin Setup: Making Custom Fields Visible

Lessons from getting custom Lead qualification fields (created for the agent) to actually appear on the Lead record page for the admin user.

### 4.1 The three-layer visibility stack — all three must be correct

A custom field created by code/CLI is invisible on a record page until **all three** of these are satisfied:

| Layer | What controls it | How to check |
|-------|-----------------|--------------|
| **Field exists in org** | `sf project deploy start` for the field metadata | `sf data query --query "SELECT BudgetMinUSD__c FROM Lead LIMIT 1"` — if it returns without error, the field exists |
| **Layout section contains the field** | `Layout` metadata deployed to the org | Retrieve the layout (`sf project retrieve start --metadata "Layout:..."`) and confirm the field is in a `<layoutItems>` block |
| **Profile (or permission set) grants FLS** | `fieldPermissions` in `Profile` or `PermissionSet` metadata | `sf data query --query "SELECT Field, PermissionsRead FROM FieldPermissions WHERE SobjectType = 'Lead' AND Parent.Name = 'Agent_Data_Admin'"` |

Newly created custom fields have **no FLS anywhere by default** — they are not automatically visible even to System Administrators. All three layers must be explicitly deployed.

### 4.2 Lightning Record Pages override classic layouts — check which one is active

In Lightning Experience, a Lightning Record Page (FlexiPage) replaces the classic page layout for display purposes. Editing the classic layout is **necessary but not sufficient** — if the page is rendered by a FlexiPage, the FlexiPage must also delegate field rendering to the layout.

**How to check which FlexiPages exist for an object:**
```bash
sf org list metadata --metadata-type FlexiPage --target-org <alias> --json \
  | python3 -c "import sys,json; [print(m['fullName']) for m in json.load(sys.stdin)['result'] if 'Lead' in m.get('fullName','')]"
```

On this SDO org, the Lead object had four active FlexiPages (`SDO_Sales_Lead_Default`, `SDO_Lead_Default`, `SDO_Sales_Lead_HVS`, `SDO_Pardot_Lead`). The admin user sees `SDO_Sales_Lead_Default` which uses the `force:detailPanel` component — this component **does** render classic layout sections and respects FLS. So the FlexiPage was not the blocker here, but always check it first before assuming the classic layout is what the user sees.

### 4.3 Profile FLS is the floor — permission sets cannot override a missing profile grant

**Root cause of this session's bug:** The `Agent_Data_Admin` permission set had correct FLS for all 9 fields, was assigned to the admin user, and was deployed to the org. Yet the fields were invisible. The System Administrator (Admin) profile had **zero FLS entries** for any of the 9 custom fields — they had never been granted at the profile level.

In Salesforce, FLS is additive: a permission set can grant access that the profile doesn't explicitly give. However, if a field has **no FLS entry at all** on the profile (not `false`, just absent), it defaults to hidden. The Admin profile in this SDO org returned 0 rows from:

```bash
sf data query --query "SELECT Field, PermissionsRead FROM FieldPermissions \
  WHERE SobjectType = 'Lead' \
  AND Parent.IsOwnedByProfile = true \
  AND Parent.Profile.Name = 'System Administrator' \
  AND Field IN ('Lead.BudgetMinUSD__c',...)" \
  --target-org <alias>
```

**Fix:** Add `fieldPermissions` entries to the `Admin` profile and deploy them.

**Diagnostic query to run before declaring FLS "done":**
```bash
# Check profile-level FLS — must return one row per field
sf data query --query "SELECT Field, PermissionsRead, PermissionsEdit \
  FROM FieldPermissions \
  WHERE SobjectType = 'Lead' \
  AND Parent.IsOwnedByProfile = true \
  AND Parent.Profile.Name = 'System Administrator' \
  AND Field IN ('Lead.BudgetMinUSD__c','Lead.BudgetMaxUSD__c','Lead.IntentLevel__c', \
    'Lead.InterestedProjects__c','Lead.InstalmentDurationMonths__c', \
    'Lead.PaymentMethod__c','Lead.PreferredBedrooms__c', \
    'Lead.PreferredPropertyType__c','Lead.QualificationSummary__c')" \
  --target-org <alias>

# Check permission set FLS — should already be present in Agent_Data_Admin
sf data query --query "SELECT Field, PermissionsRead FROM FieldPermissions \
  WHERE SobjectType = 'Lead' AND Parent.Name = 'Agent_Data_Admin'" \
  --target-org <alias>
```

Both queries must return 9 rows before the fields will be visible.

### 4.4 SDO orgs have an unrecognised metadata type that breaks `sf project deploy start`

This SDO org contains `force-app/main/default/externalClientApps/My_Agent_API.externalClientApplication-meta.xml`. The `sf project deploy start` and `sf project retrieve start` commands fail with `TypeInferenceError` when run from the project root because the CLI cannot resolve `.externalClientApplication` as a known metadata type.

**Workaround — use a minimal temp project:**
```bash
mkdir -p /tmp/sf-tmp && cd /tmp/sf-tmp
cat > sfdx-project.json << 'EOF'
{
  "packageDirectories": [{"path": "force-app", "default": true}],
  "name": "tmp", "namespace": "", "sourceApiVersion": "62.0"
}
EOF
mkdir -p force-app/main/default/layouts
mkdir -p force-app/main/default/profiles

# Retrieve into the temp project
sf project retrieve start --metadata "Layout:Lead-SDO - Lead" --target-org <alias>
sf project retrieve start --metadata "Profile:Admin" --target-org <alias>

# Edit the files, then deploy from the temp project
sf project deploy start --metadata "Layout:Lead-SDO - Lead" --target-org <alias>
sf project deploy start --metadata "Profile:Admin" --target-org <alias>
```

After a successful deploy from the temp project, copy the files back into the main project for source control:
```bash
cp /tmp/sf-tmp/force-app/main/default/layouts/*.layout-meta.xml \
   /path/to/project/force-app/main/default/layouts/
cp /tmp/sf-tmp/force-app/main/default/profiles/*.profile-meta.xml \
   /path/to/project/force-app/main/default/profiles/
```

### 4.5 Retrieve layouts from the org before editing — the `layouts/` folder may be empty

The project's `force-app/main/default/layouts/` folder can be **empty** even though layouts exist in the org. Always retrieve the live layout before editing so you don't overwrite org-side customisations:

```bash
sf project retrieve start --metadata "Layout:<ObjectName>-<LayoutName>" \
  --target-org <alias>
# e.g. "Layout:Lead-SDO - Lead"
```

List all layouts for an object to find the exact API name:
```bash
sf org list metadata --metadata-type Layout --target-org <alias> --json \
  | python3 -c "import sys,json; [print(m['fullName']) for m in json.load(sys.stdin)['result'] if 'Lead' in m.get('fullName','')]"
```

### 4.6 Add a new layout section — minimal correct XML

To add a two-column section to a classic layout, insert before the first non-data section (e.g. "Useful Links"):

```xml
<layoutSections>
    <customLabel>true</customLabel>
    <detailHeading>true</detailHeading>
    <editHeading>true</editHeading>
    <label>Agent Qualification</label>
    <layoutColumns>
        <layoutItems>
            <behavior>Edit</behavior>
            <field>IntentLevel__c</field>
        </layoutItems>
        <!-- more fields -->
    </layoutColumns>
    <layoutColumns>
        <layoutItems>
            <behavior>Edit</behavior>
            <field>QualificationSummary__c</field>
        </layoutItems>
        <!-- more fields -->
    </layoutColumns>
    <style>TwoColumnsLeftToRight</style>
</layoutSections>
```

Use `<behavior>Edit</behavior>` for editable fields and `<behavior>Readonly</behavior>` for formula or computed fields. `QualificationSummary__c` (written by the agent's Flow) should be `Edit` so an admin can also manually update it.

### 4.7 Quick admin setup checklist for new custom fields

After deploying custom fields to an SDO org for the first time:

- [ ] Confirm fields exist: `sf data query --query "SELECT <Field1__c>, <Field2__c> FROM Lead LIMIT 1"`
- [ ] Retrieve the target layout and confirm the section exists in the XML
- [ ] Deploy the layout if the new section was added
- [ ] Run the FLS diagnostic query (§4.3) for the relevant profile — expect 0 rows initially
- [ ] Add `fieldPermissions` (readable + editable) to the `Admin` profile for each field
- [ ] Deploy the updated profile
- [ ] Re-run the FLS diagnostic query — expect 1 row per field
- [ ] Check which FlexiPage the user sees (§4.2) — confirm it uses `force:detailPanel`
- [ ] Refresh the Salesforce record page in the browser (hard refresh: Cmd+Shift+R)

---

## Skills used in this build (reference)

| Skill | Used for |
|-------|---------|
| `/developing-agentforce` | Agent Script syntax, `before_reasoning` pattern, CLI commands, publish troubleshooting |
| `/testing-agentforce` | Mode A preview workflow, trace analysis, test case design |
| `sf-flow` / `generating-flow` | Would have caught `TIME()` error and picklist formula limitations earlier |
| `sf-metadata` / `generating-custom-field` | Would have surfaced `MultiselectPicklist` type early and prevented picklist crash |
| `sf-soql` / `querying-soql` | Should have been used to verify picklist values before wiring them in Flow |
| `/sf-permissions` + `/agentforce-slack-research` | Diagnosed profile FLS gap causing fields to be invisible on the Lead record page |

---

## Quick diagnostic checklist

Before declaring an agent ready to publish, verify each of these:

- [ ] `sf agent validate authoring-bundle` passes with zero errors
- [ ] `sf agent preview start --use-live-actions` starts without error (catches type mismatches)
- [ ] All Currency action I/O uses `object` + `complex_data_type_name: "lightning__currencyType"`
- [ ] All restricted picklist inputs in Flows have a formula guard returning `""` for invalid values
- [ ] All multi-select picklist fields use intermediate text variable assignments before formula use
- [ ] `before_reasoning` guards use `!=` for empty-string checks
- [ ] All standard picklist values (e.g. `LeadSource`) confirmed active in the target org via describe
- [ ] Session trace `FunctionStep` shows correct inputs and outputs for each action
- [ ] Salesforce records verified post-scenario via `sf data query`
- [ ] Test Lead records deleted between runs
- [ ] Demo script exists with step-by-step SE instructions for all 4 scenarios
- [ ] All custom fields visible on the Lead page: layout section added, profile FLS deployed (§4.7)
