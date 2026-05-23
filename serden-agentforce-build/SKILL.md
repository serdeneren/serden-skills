---
name: serden-agentforce-build
description: >-
  Hard-won gotchas from building a production Agentforce agent end-to-end: Agent Script
  authoring traps, Flow-backed action design, custom field visibility, and admin setup.
  Use when building or debugging an Agentforce agent, designing Flow backing logic for
  agent actions, wiring action I/O types, handling picklist safety, or making custom
  fields visible on record pages. Pairs with /developing-agentforce.
disable-model-invocation: true
---

# Agentforce Build Playbook

Lessons from building a sales and leasing agent end-to-end: Flow-backed actions, Lead/Event creation, and custom field setup.

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

Actions placed in `reasoning: actions:` are offered to the LLM as tools. The LLM decides whether and when to call them. Instructions like "call X immediately after Y" are hints, not enforcement.

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
`before_reasoning` runs at the START of each turn, before the LLM. If the customer provides their email on turn N, `before_reasoning` won't see it until turn N+1. The Lead is created on the first turn after email is captured, not the same turn.

### 1.4 Restricted and multi-select picklists crash Flow silently then loudly

Salesforce Flow throws `INVALID_OR_NULL_FOR_RESTRICTED_PICKLIST` at runtime if you pass a value that isn't in a restricted picklist. The agent passes LLM-generated text to Flow inputs mapped to restricted fields.

**Fix pattern — formula with keyword matching:**
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

Returning `""` for unrecognised values is always safer than passing the raw LLM string to a restricted picklist.

**Multi-select picklists** cannot be directly concatenated in Flow formula strings — use an `<assignments>` element to copy picklist field values into `String` variables first.

### 1.5 Validate picklist values in the org before wiring them in Flow

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

In this project `"Web"` did not exist, `"Website"` did. The Flow was silently assigning an invalid value.

### 1.6 Flow DateTime formulas don't have a `TIME()` function

```xml
<expression>DATETIMEVALUE({!requestedDateStr} &amp; " 09:00:00")</expression>
```

`TIME()` is a spreadsheet function, not a Salesforce Flow formula function.

### 1.7 `GenAiFunctionDefinition` duplication at publish time

If you change an action's `description` between topics or agent versions, the platform may attempt to create duplicate `GenAiFunctionDefinition` records. Symptoms: publish succeeds for one action but fails with `duplicate value found` for another.

**Fix:** Ensure `description` is **identical** across all topics that reference the same action. Deactivate then re-publish to clear state if the error persists.

### 1.8 `validate authoring-bundle` doesn't catch everything

The local validator does NOT catch:
- Type mismatches between action I/O and Flow variables (caught at `preview start`)
- Missing permissions on the Einstein Agent User (caught at `preview start`)
- `GenAiFunctionDefinition` duplication (caught at `publish`)
- Invalid picklist values (caught at Flow runtime)

Always run `sf agent preview start --use-live-actions` after validate before publishing.

### 1.9 Use session traces, not just preview output

Trace files at `.sfdx/agents/<bundle>/sessions/<id>/traces/<planId>.json` show:
- Which `before_reasoning` actions actually fired (`BeforeReasoningStep`)
- What variables held at the start of the turn (`NodeEntryStateStep`)
- What tools the LLM had available (`EnabledToolsStep`)
- What inputs and outputs a Flow received (`FunctionStep`)

The most common diagnosis error: looking only at the response text and inferring the action didn't fire, when actually it fired but returned `amount: 0` because the dollar sign was stripped from the message.

### 1.10 `filter_from_agent: True` hides output from the LLM — including for gate logic

`filter_from_agent: True` means the LLM **cannot see that value** — even though it is available for `set` assignments. Fatal for any output the LLM needs to reason about or branch on.

**Rule:** Use `filter_from_agent: True` only for outputs used exclusively in `available when` guards or `set` assignments in deterministic blocks. If the LLM needs to branch based on a value, set `filter_from_agent: False`.

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

When a gate topic succeeds and uses `after_reasoning` to transition to `topic_selector`, **both topics execute in the same user turn**. The hub fires against the gate's triggering message (e.g. `"1234"`), which matches no domain topic and routes to `off_topic`.

**Fix — two parts:**

1. Add a transition in `topic_selector` so numeric/code-like inputs route to the verification topic.

2. Add a conditional instruction in `topic_selector` for when verification just completed:

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
```

### 1.12 Variable type rules — quick reference

| Context | Allowed types |
|---------|--------------|
| Mutable variable | `string`, `number`, `boolean`, `object`, `date`, `id`, `list[T]` |
| Action I/O (Currency Flow var) | `object` + `complex_data_type_name: "lightning__currencyType"` |
| Action I/O (Number Flow var) | `number` |
| Action I/O (Date Flow var) | `date` |
| Action I/O (Apex Integer/Long) | `integer` / `long` (action params only — NOT mutable vars) |

---

## Part 2 — Debugging Flow errors

### 2.1 How to surface Flow errors from a running agent

**Via debug logs (most reliable):**
1. In Setup → Debug Logs, add the **Einstein Agent User** as a traced entity
2. Set log level: Workflow = FINEST, Apex = DEBUG
3. Re-run the failing utterance via `sf agent preview send` (not via the browser)
4. Open the log — search for `FLOW_ELEMENT_ERROR` or the Flow name

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

### 2.2 Common Flow error patterns and fixes

| Error message | Root cause | Fix |
|--------------|-----------|-----|
| `INVALID_OR_NULL_FOR_RESTRICTED_PICKLIST` | LLM passed free-text to a restricted picklist | Add a formula that maps keywords to valid API values; return `""` for unrecognised input |
| `REQUIRED_FIELD_MISSING: Required fields are missing: [LastName]` | Flow tried to create a record without a mandatory field | Add a formula fallback: `IF(ISBLANK({!lastName}), IF(ISBLANK({!firstName}), "Unknown", {!firstName}), {!lastName})` |
| `FIELD_FILTER_VALIDATION_EXCEPTION` | Filter logic references a null field | Add `assignNullValuesIfNoRecordsFound = true` on record lookup elements |
| `Compile error: Unknown function TIME` | Flow formula used `TIME()` which doesn't exist | Replace with `DATETIMEVALUE({!dateStr} & " 09:00:00")` |
| Multi-select picklist in formula string concat | Flow formulas can't directly concatenate picklist values | Use `<assignments>` to copy to a `String` variable first |

### 2.3 The fastest debug cycle for Flow errors

```
1. Reproduce via sf agent preview (--use-live-actions --authoring-bundle) →
   capture the planId for the failing turn
2. Read ActionExecutionStep / FunctionStep in the trace to see exact Flow inputs
3. Open Debug Logs → search FLOW_ELEMENT_ERROR → read element name and error
4. Open the Flow XML → find the element by name
5. Apply fix from §2.2 table
6. Deploy: sf project deploy start --json --metadata Flow:<FlowName>
7. Re-run in a new preview session — confirm trace shows correct outputs
```

Do NOT publish a new agent version just to fix a Flow bug — Flow changes take effect immediately on deploy without republishing.

---

## Part 3 — Salesforce Admin Setup: Making Custom Fields Visible

### 3.1 The three-layer visibility stack — all three must be correct

| Layer | What controls it | How to check |
|-------|-----------------|--------------|
| **Field exists in org** | `sf project deploy start` | `sf data query --query "SELECT BudgetMinUSD__c FROM Lead LIMIT 1"` |
| **Layout section contains the field** | `Layout` metadata | Retrieve layout and confirm field is in a `<layoutItems>` block |
| **Profile (or permission set) grants FLS** | `fieldPermissions` in `Profile` or `PermissionSet` | `sf data query --query "SELECT Field, PermissionsRead FROM FieldPermissions WHERE SobjectType = 'Lead' AND Parent.Name = 'Agent_Data_Admin'"` |

Newly created custom fields have **no FLS anywhere by default** — not automatically visible even to System Administrators.

### 3.2 Profile FLS is the floor — permission sets cannot override a missing profile grant

If a field has **no FLS entry at all** on the profile (not `false`, just absent), it defaults to hidden. Permission sets alone are insufficient.

**Diagnostic query to run before declaring FLS "done":**
```bash
sf data query --query "SELECT Field, PermissionsRead, PermissionsEdit \
  FROM FieldPermissions \
  WHERE SobjectType = 'Lead' \
  AND Parent.IsOwnedByProfile = true \
  AND Parent.Profile.Name = 'System Administrator' \
  AND Field IN ('Lead.BudgetMinUSD__c','Lead.BudgetMaxUSD__c',...)" \
  --target-org <alias>
```

Must return one row per field. If 0 rows — add `fieldPermissions` to the `Admin` profile and deploy.

### 3.3 SDO orgs have an unrecognised metadata type that breaks `sf project deploy start`

The `externalClientApplication` metadata type causes `TypeInferenceError` when deploying from the project root.

**Workaround — use a minimal temp project:**
```bash
mkdir -p /tmp/sf-tmp && cd /tmp/sf-tmp
cat > sfdx-project.json << 'EOF'
{"packageDirectories": [{"path": "force-app", "default": true}], "name": "tmp", "namespace": "", "sourceApiVersion": "62.0"}
EOF
mkdir -p force-app/main/default/layouts force-app/main/default/profiles

sf project retrieve start --metadata "Layout:Lead-SDO - Lead" --target-org <alias>
sf project retrieve start --metadata "Profile:Admin" --target-org <alias>
# Edit files, then:
sf project deploy start --metadata "Layout:Lead-SDO - Lead" --target-org <alias>
sf project deploy start --metadata "Profile:Admin" --target-org <alias>
```

Copy files back to the main project for source control after a successful deploy.

### 3.4 Retrieve layouts before editing — the `layouts/` folder may be empty

```bash
sf project retrieve start --metadata "Layout:<ObjectName>-<LayoutName>" --target-org <alias>

# List all layouts for an object to find the exact API name
sf org list metadata --metadata-type Layout --target-org <alias> --json \
  | python3 -c "import sys,json; [print(m['fullName']) for m in json.load(sys.stdin)['result'] if 'Lead' in m.get('fullName','')]"
```

### 3.5 Quick admin setup checklist for new custom fields

- [ ] Confirm fields exist: `sf data query --query "SELECT <Field1__c> FROM Lead LIMIT 1"`
- [ ] Retrieve the target layout and confirm the section exists in the XML
- [ ] Deploy the layout if the new section was added
- [ ] Run the FLS diagnostic query (§3.2) — expect 0 rows initially
- [ ] Add `fieldPermissions` (readable + editable) to the `Admin` profile for each field
- [ ] Deploy the updated profile
- [ ] Re-run the FLS diagnostic query — expect 1 row per field
- [ ] Check which FlexiPage the user sees — confirm it uses `force:detailPanel`
- [ ] Refresh the Salesforce record page (hard refresh: Cmd+Shift+R)

---

## Pre-publish diagnostic checklist

- [ ] `sf agent validate authoring-bundle` passes with zero errors
- [ ] `sf agent preview start --use-live-actions` starts without error (catches type mismatches)
- [ ] All Currency action I/O uses `object` + `complex_data_type_name: "lightning__currencyType"`
- [ ] All restricted picklist inputs in Flows have a formula guard returning `""` for invalid values
- [ ] All multi-select picklist fields use intermediate text variable assignments before formula use
- [ ] `before_reasoning` guards use `!=` for empty-string checks
- [ ] All standard picklist values confirmed active in the target org via describe
- [ ] Session trace `FunctionStep` shows correct inputs and outputs for each action
- [ ] All custom fields visible on the record page: layout section added, profile FLS deployed (§3.5)
