---
name: serden-agentforce-build
description: >-
  Hard-won gotchas from building production Agentforce agents end-to-end: Agent Script
  authoring traps, Flow-backed action design, custom field visibility, admin setup, and
  service agents on Financial Services Cloud (FSC) over the email channel (invocable Apex
  actions, Knowledge grounding, PCC hand-off, email deliverability). Use when building or
  debugging an Agentforce agent, designing Flow/Apex backing logic for agent actions,
  wiring action I/O types, handling picklist safety, making custom fields visible, or
  diagnosing agent-user permission/email/flow failures. Pairs with /developing-agentforce.
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

### 3.2 FLS is the union of profile + assigned permission sets — grant it in either

Effective FLS for a user is the **union** of their profile and every assigned permission set; the more permissive grant wins. So a field with **no FLS entry at all** on the profile (not `false`, just absent) defaults to hidden — *unless* an assigned permission set grants it. Two valid ways to make a field visible to a user:

- **Permission set** (additive, reversible, no profile edit) — add `fieldPermissions` to a permission set and assign it to the user. This works even when the profile has no entry. Fastest fix, and the only practical option for SDO orgs where editing the profile is painful (§3.3).
- **Profile** — add `fieldPermissions` to the user's profile and deploy.

> Earlier versions of this playbook claimed "permission sets cannot override a missing profile grant" — that is **wrong**. Permission sets are additive and absolutely can grant FLS the profile lacks. The real-world failure (see §4.5) is granting FLS only to the **agent user's** permission set and forgetting the **human** who views the records.

**Diagnostic query to run before declaring FLS "done"** (checks the profile only; also check assigned permission sets):
```bash
sf data query --query "SELECT Field, PermissionsRead, PermissionsEdit \
  FROM FieldPermissions \
  WHERE SobjectType = 'Lead' \
  AND Parent.IsOwnedByProfile = true \
  AND Parent.Profile.Name = 'System Administrator' \
  AND Field IN ('Lead.BudgetMinUSD__c','Lead.BudgetMaxUSD__c',...)" \
  --target-org <alias>
```

Must return one row per field for the profile path. If 0 rows, the user can still see the field if an **assigned permission set** grants it — drop the `Parent.IsOwnedByProfile = true` filter to check permission sets too. If neither grants it, either add `fieldPermissions` to the profile and deploy, or add them to a permission set and assign it to the user.

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

## Part 4 — Service agents on FSC over the email channel (invocable Apex)

Lessons from building a **service agent** (`AgentforceServiceAgent`) on a
Financial Services Cloud trial org: invocable Apex actions, Knowledge grounding, a
Case + custom-object hand-off, and demo-grade automation.

### 4.1 The CLI generates the `subagent` grammar, not `topic`

`sf agent generate authoring-bundle` (current CLI) scaffolds **`start_agent <router>` +
`subagent X:` blocks with `@utils.transition to @subagent.X`** — NOT the `topic` grammar
shown in much of `/developing-agentforce`. Mixing the two breaks compilation.

**Rule:** Open the generated `.agent` file FIRST and match whatever grammar it emitted
(`subagent` vs `topic`). The `reasoning:` / `actions:` internals are the same; only the
block keyword and `@subagent.`/`@topic.` reference prefix differ.

### 4.2 Reference an action output as a variable → must capture it first

`{!@variables.X}` in instructions only works for declared **variables**. An action
**output** (e.g. `accountSummary`) is not a variable. Capture it with `set` into a
mutable variable, then template that variable:
```agentscript
set @variables.account_overview = @outputs.accountSummary   # in the action's set block
# ...later...
| Their accounts: {!@variables.account_overview}
```
Symptom if you skip it: `CompilationError: 'accountSummary' is not defined in variables`.

### 4.3 Prefer preformatted `string` outputs over typed object/list outputs

Returning a single human-readable, displayable `string` (e.g. a formatted balance/txn
list) from invocable Apex grounds reliably and **avoids `complex_data_type_name` mapping
entirely**. Pattern: `summary: string (is_displayable: True)` + `isSuccess: boolean
(filter_from_agent: True)`. Tell the LLM to "write the exact values from the summary
field, do not round/paraphrase, and do NOT use the show_command tool."

### 4.4 Test as the AGENT USER, not as admin — they behave differently

Anonymous Apex (`sf apex run`) executes as the **admin**, which masks permission/context
gaps. The agent runs as the **Einstein Agent User**. Always validate behavior with
`sf agent preview start --use-live-actions` (runs as the agent user). Things that pass as
admin but fail/differ as the agent user: object/field CRUD, Knowledge access, and SOSL
relevance.

### 4.5 Grant the agent user explicit CRUD + FLS on every object/field the actions touch

Invocable Apex reads ran in system mode (queries returned data without object grants),
but for objects the agent **writes** (Case, custom hand-off object) and their **custom
fields**, add to the custom `{Agent}_Access` permission set:
- `<classAccesses>` for every Apex class (required to invoke).
- `<objectPermissions>` (create/read/edit) for objects the actions insert/update.
- `<fieldPermissions>` for every custom field written.
- `<objectPermissions allowRead>` for `Knowledge__kav` if doing Knowledge search.

Also note: a SOQL "**No such column 'X__c'**" error on a field that *does* exist is an
**FLS** symptom for the running user — confirm existence via Tooling API
`FieldDefinition`, then grant FLS (see §3.2).

**The FLS in this section is for the agent user's *runtime* only — it does NOT make the
fields visible to humans.** Any **human** who reviews these records in the Salesforce UI
(the demo presenter, the admin, a service rep) needs FLS **separately**, via their profile
**or an assigned permission set**, AND the custom fields must be on the **page layout**
(§3.1, §3.5). Classic miss on a service-agent build: the `{Agent}_Access` permission set
gives the agent user read/write all along, so the agent works perfectly — but the human
demoing it opens the Case and the custom fields are invisible (no profile/permset FLS) or
absent (not on the layout). A System Administrator profile does **not** get FLS on
metadata-deployed custom fields automatically (§3.1). Quickest fix: assign the same
`{Agent}_Access` permission set (or a small view-only one) to the human user, and add the
fields to the layout.

### 4.6 A record-triggered Flow that sends email rolls back the triggering DML

If invocable Apex inserts a record and an **after-save record-triggered Flow** on that
object runs a synchronous action that throws (e.g. email send fails), the **Apex insert
is rolled back** inside the same transaction — your action silently "fails" even though
the parent record (e.g. the Case, inserted earlier and not rolled back by a caught
exception) persists. Classic split-state symptom: Cases exist, child records don't.

**Fix:** run the side-effect on an **async-after-commit** path so a failure can't roll
back the trigger:
```xml
<start>
    <object>PCC_Request__c</object>
    <recordTriggerType>Create</recordTriggerType>
    <scheduledPaths>
        <name>Run_Async</name>
        <connector><targetReference>My_Side_Effect</targetReference></connector>
        <pathType>AsyncAfterCommit</pathType>
    </scheduledPaths>
    <triggerType>RecordAfterSave</triggerType>
</start>
```
Gotcha: an `AsyncAfterCommit` `<scheduledPaths>` must NOT include `<label>` (or
TimeSource/Offset*) — deploy fails with *"… cannot be set for ScheduledPath of PathType …"*.

**Beware ordering:** an Apex smoke test that passed earlier can start failing once you
deploy a record-triggered Flow on the same object — the Flow didn't exist during the
smoke test.

### 4.7 Outbound email is often blocked in trial/demo orgs — use Chatter for demos

Trial orgs frequently have **no verified sending domain and no Org-Wide Email Address**,
so Flow/Apex `emailSimple` fails with *"your email address domain isn't verified"* and
(on async paths) emails the admin on every run. For a robust demo, swap `emailSimple` for
a **Chatter post** (`actionType: chatterPost`, `text` + `subjectNameOrId = {!$Record.Case__c}`)
— no deliverability dependency, never errors, and it's visible in the UI. Document that
production reverts to email once a verified Org-Wide Email Address / domain is configured.

### 4.8 SOSL Knowledge search is unreliable across user contexts → score in Apex

SOSL `FIND` relevance can return nothing for long natural-language queries (stopwords,
`?`) and differs between admin and the agent user. For a small, bounded article set,
**skip SOSL** and do deterministic in-Apex keyword scoring: query online `Knowledge__kav`,
split the query into terms (drop stopwords, len ≥ 3), score each article by term hits
(weight Title higher), return the top match. Reliable and user-context-independent.

### 4.9 `Knowledge User` may be un-settable on the agent user's license

`User.UserPermissionsKnowledgeUser = true` can fail with *"Knowledge User is not allowed
for this License Type"* for the Einstein Agent User. Grant **`Knowledge__kav` object read
via the permission set** instead, and run Knowledge Apex `without sharing`.

### 4.10 Route by INTENT (information vs action), not by topic keyword

When the same domain (cards, payment limits, debit orders) appears in both a "service
request" bucket and a "query" bucket, keyword routing misfires. Instruct the router to
decide by **what the client wants**: action verbs (order, replace, reverse, increase,
amend, link, stop) → the request/case path; question forms (what, how, can I, fees,
steps) → the Knowledge path. Note even this leaves edge phrases (e.g. "what's my limit
and **can it be increased**?") ambiguous — accept it or add a clarifying turn.

### 4.11 Project & shell setup gotchas

- `sf project generate --name X` creates a **nested `X/` subfolder**; move contents to the
  repo root if you want a flat project (`setopt dotglob` then `mv X/* .` — **zsh has no
  `shopt`**).
- `sf project deploy start` is **atomic**: one failing component rolls back the entire
  deploy (the "Created" states you see are intended, not committed). Deploy all fixed
  components together and re-check; don't assume partial save.
- Apex **reserved words** can't be local variable names — `desc`, `like` fail to compile;
  use `descTxt`, `likeTerm`.

### 4.12 `agent_type` is IMMUTABLE — switching Employee↔Service is a NEW agent, not an edit

You cannot flip `agent_type` on an existing agent. The type is fixed at creation
(confirmed in the Agentforce Developer Guide and platform behaviour). Changing
`AgentforceEmployeeAgent` → `AgentforceServiceAgent` in the same `.agent` config and
republishing under the same `developer_name` does **not** convert the agent — you must
create a **new agent with a unique `developer_name`**, which produces a **new BotDefinition
and a new agent ID (`0Xx...`)**.

Practical recipe (non-destructive, keeps the old agent for rollback):
1. Copy the bundle to a new folder, e.g. `aiAuthoringBundles/<Name>_Service/<Name>_Service.agent`
   (+ a copied `.bundle-meta.xml`), reusing all the same `apex://` / `mcpTool://` targets.
2. Change only the config block: new `developer_name`, `agent_type: "AgentforceServiceAgent"`,
   add `default_agent_user`. Keep `agent_label` the same if you want the UI name unchanged.
3. Deploy → preview → publish → activate the new agent; then `sf agent deactivate --api-name <OldName>`.
4. The auto-generated metadata (`bots/`, `genAiPlanners/`, `genAiPlannerBundles/`,
   `genAiPlugins/`, `genAiFunctions/`) for the OLD agent stays behind — the new agent
   generates its own set on publish. Retrieve it afterward only if you want it in git.

Reusing the old name requires deleting the old agent first (destructive) and is rarely
worth it — downstream callers reference the agent by **ID**, not name (see §4.13).

### 4.13 Headless agents over the Agent API + an external channel (e.g. Telegram via n8n)

An agent can be driven entirely headless through the **Einstein AI Agent REST API**
(`https://api.salesforce.com/einstein/ai-agent/v1/agents/{agentId}/sessions` then
`.../sessions/{sessionId}/messages`), with the user-facing channel living **outside**
Salesforce (n8n bridging Telegram ↔ Agent API). In this setup there is **no native
Salesforce messaging channel / MIAW / Omni-Channel** to configure — "the messaging
channel" is just the external bridge.

Key consequences:
- Auth is a **client-credentials Connected App** (`isClientCredentialEnabled=true`). Scopes
  `Api` + `Chatbot` work for **both** Employee and Service agents over this endpoint; only
  move to `sfap_api` / `chatbot_api` if sessions get rejected. The **same** connected app
  serves a Service agent — you don't need a new one just because the agent type changed.
- The **agent ID is the coupling point** and is often **hardcoded** in the external client
  (e.g. an n8n Code node `SF_AGENT_ID`). Because a type switch mints a NEW agent ID (§4.12),
  the external bridge is **broken until you update that ID**. Always treat "update the
  hardcoded agent ID + smoke-test the channel end-to-end" as a required migration step, not
  an afterthought.
- Employee agents over the Agent API run as the **calling/run-as user**; Service agents run
  as their `default_agent_user`. Switching to Service is the right move for an
  external/unauthenticated audience precisely because every end user no longer needs to be a
  Salesforce user.

### 4.14 MCP-backed actions (`mcpTool://`) — credential access + a preview blind spot

When actions target `mcpTool://<id>` (an MCP server registered via
`ExternalServiceRegistration` + a `NamedCredential`/`ExternalCredential`), the running
agent user must be able to use that credential:
- If the External Credential uses a **Named Principal** (shared OAuth/client-credentials to
  the MCP server), the agent user usually needs nothing extra.
- If it uses a **Per-User Principal**, add `UserExternalCredential` object read **and**
  `<externalCredentialPrincipalAccesses>` to the agent user's custom PS (same pairing as
  §4.5 / agent-user-setup §3 caveats).

Preview blind spot: `mcpTool://` actions frequently **do not surface** in
`sf agent preview --authoring-bundle` (the `EnabledToolsStep` comes back empty), so you
cannot smoke-test them that way. Validate MCP tools by calling the MCP server **directly
with the official MCP SDK client** (see the test playbook), then confirm end-to-end on the
published agent. Don't conclude the action is "broken" from an empty preview tool list.

---

## Part 5 — Enabling & debugging the Email channel (Agentforce for Service on Email)

Lessons from taking a **service agent** live on **email** end-to-end on an SDO/trial org.
The channel = **Agentforce for Service on Email**, which rides on top
of **Email-to-Case**: inbound email → Case → the agent **owns** the Case → the product
(not your Flows) sends the conversational reply. The whole thing only fires if the agent
is the Case owner **synchronously at creation**.

### 5.1 Set the default org per-workspace, not globally

`sf config set target-org <alias>` writes to the project's `.sf/config.json` (Local scope)
— it does **not** touch global config. Use the **alias**, not the My-Domain URL — passing
the `*.lightning.force.com` URL fails with *"org … is not authenticated"*. Confirm scope
with `sf config get target-org --json` → `"location": "Local"`.

### 5.2 No real domain needed for a demo — use the on-demand Email-to-Case address

You do **not** need SPF/DKIM/DMARC or a verified sending domain to demo inbound email.
Salesforce generates an **on-demand Email-to-Case routing address** (a long
`…@<hash>.<region>.case.salesforce.com`) that accepts mail immediately. Setup → Email →
**Deliverability = All email** is enough for the demo. The SPF/DKIM/custom-domain work is
only for the **outbound** branded "From" address in production.

### 5.3 The routing address's "Email Address" field still must be a REAL, VERIFIED inbox

The Email-to-Case **Routing Address** form has two different addresses — don't confuse them:
- **Email Address** (what you type): a friendly address that **must be verified**. Salesforce
  emails a verification link to it. A fake value (`privatebanker@demo.example.com`) leaves the
  routing address **Verification: Pending**, and inbound mail is silently **not** converted to
  a Case. Use a real inbox you control (your own `@salesforce.com` or a Gmail) so you can click
  the link. Status must read **Verified**.
- **Email Services Address** (auto-generated, long `@…case.salesforce.com`): this is what you
  actually **send test email to**.

`Accept Email From` blank = accept from any sender (Gmail, corporate, anything) — good for demos.

### 5.4 Send to the EXACT generated address — pull it from the org, don't retype

Typos in the long hash address bounce with *"… is not a valid address"* (a
`mailer-daemon@mailerdaemon.mta.salesforce.com` reply). Reading it off a screenshot is
error-prone. Pull it verbatim via SOQL (note: `EmailServicesAddress` has **no `Name`
field**):
```bash
sf data query --query "SELECT LocalPart, EmailDomainName, IsActive FROM EmailServicesAddress" --target-org <alias>
# send-to address = <LocalPart>@<EmailDomainName>
```

### 5.5 Case Owner on the routing address = the Einstein Agent User

Set the routing address **Case Owner** to the agent's run-as user (the **Einstein Agent
User**, e.g. `<agent-run-as-user>@example.com`). The agent only auto-replies if it owns
the Case at creation (confirms §"Case ownership at creation"). Find the user — it may not
appear in Setup's default user-list filters but it exists (search by the agent-user
username prefix or by license type):
```bash
sf data query --query "SELECT Id, Name, Username, UserType, IsActive FROM User WHERE Username LIKE '%<agent-user-prefix>%'" --target-org <alias>
```
It has a special **Einstein Agent** license / **Einstein Agent User** profile. Confirm it
can own/create Cases (Create/Read/Edit on Case) via its assigned permission sets:
```bash
sf data query --query "SELECT Parent.Label, PermissionsCreate, PermissionsRead, PermissionsEdit FROM ObjectPermissions WHERE SObjectType='Case' AND ParentId IN (SELECT PermissionSetId FROM PermissionSetAssignment WHERE AssigneeId='<agentUserId>')" --target-org <alias>
```

### 5.6 Add the **Service Email connection** in Agent Builder BEFORE saving the Email Configuration

Setup → **Agentforce for Service on Email** → New Configuration fails to save with:
> *"Before you save the email configuration, add the email connection in the Agent Builder."*

Fix: open the agent in **Agent Builder** → **Connections** → add **Service Email**, then
**activate** that agent version. On older SDO orgs the **old connections layout only shows
"API"** — you must **upgrade to the new Connections layout** first before Service Email
appears (the "AI Agents: Connections UI Left-Rail Panel" pref / `AgentSurfacesBuilderPanel`).
Then return to Setup and the configuration saves. (Confirmed across multiple field threads.)

### 5.7 The Email Configuration form — required fields & the template placeholder

Setup → **Agentforce for Service on Email** → New Configuration requires:
- **Configuration Name** + **API Name** (auto-fills)
- **Email Template** — the wrapper template, and it **must contain the literal
  `[[[GENERATED_CONTENT]]]` placeholder** where the agent's reply is injected. Selecting a
  template without it errors: *"The email template … must contain the [[[GENERATED_CONTENT]]]
  placeholder."* Edit the template body to add it (e.g. `Dear …,\n\n[[[GENERATED_CONTENT]]]\n\n<signature>`).
- **Agentforce Service Agent** = your agent
- **Agentforce Service Agent Signature** + **Legal Disclosure** — both **mandatory** (any text)
- **Reply All** — leave unchecked

### 5.8 "We couldn't commit a version of this agent. Check Agent Script…" on activate

Generic activate/commit failure. The trailing error number is a **transient trace ID, not
the root cause**. Triage:
1. Open the builder **Inspect panel** / browser **Network tab** → the failing
   `publish`/`commit` response body usually names the real validation node (missing label,
   description > 255 chars, bad action input param, restricted picklist value).
2. **If all calls are 200 but commit still fails**, or a router-only/minimal agent fails,
   it's likely a **platform bug** (open P1s on 262: `W-22963131`, `W-22961149`,
   `W-22967688`) — file a case citing those; don't keep editing your script.
3. Workaround: commit/activate via CLI instead of the UI —
   `sf agent publish authoring-bundle` → `sf agent activate --api-name <name>`.
4. Common fixable cause: an action with an invalid input param — **re-add the action fresh
   from the Asset Library** so inputs auto-populate.

### 5.9 Debugging "I sent email but no Case was created" — the decisive ladder

Work top-down; each rung tells you which half of the pipeline is at fault.

1. **Confirm the mail actually reached Salesforce — Email Log Files.** Setup → **Email Log
   Files** → request a log → look for your inbound row: `Email Direction = Inbound`, Mail
   Event `R` (received) then `D` (delivered), `SPF Status = Pass`, and **no** matching
   `mailer-daemon` bounce. If you see that, **deliverability is NOT the problem** — the mail
   arrived and the failure is downstream (case creation). A typo'd send-to address instead
   shows a `mailer-daemon` bounce row.
2. **Confirm records really are missing** (not just a UI filter):
   ```bash
   sf data query --query "SELECT Id, Subject, Origin, CreatedDate FROM Case WHERE CreatedDate = TODAY ORDER BY CreatedDate DESC" --target-org <alias>
   sf data query --query "SELECT Id, Subject, FromAddress, Incoming, CreatedDate FROM EmailMessage ORDER BY CreatedDate DESC LIMIT 5" --target-org <alias>
   ```
   **Mail delivered (rung 1) + zero Case/EmailMessage = something rolled the transaction back.**
3. **Surface the actual error — the single highest-value step.** Setup → **Email-to-Case** →
   Edit → tick **"Notify sender about Email-to-Case processing errors"** → Save, then resend.
   Salesforce emails the **exact failing element** (flow/trigger/limit) to the sender. With
   this OFF (the default), every failure is silent — no bounce, no record, no clue.
4. Only if rung 1 shows the mail never arrived do you chase deliverability (spam/greylisting
   — Gmail senders are dropped more often than same-org `@salesforce.com` senders on trial orgs).

### 5.10 ROOT CAUSE we hit: a pre-existing SDO demo Flow rolled back the Email-to-Case insert

This is §4.6 in the wild, but from a **stock SDO demo flow you didn't write**, triggered on
**EmailMessage** (not your object). The error notification (5.9 rung 3) named it exactly:

> Flow `SDO_Service_Email_Message_On_Create_or_Update`, element `Notify_Case_Owner`
> (Send Custom Notification): **Missing required input parameter: customNotifTypeId**

What happened: inbound email → Case + EmailMessage created → this after-save flow on
EmailMessage runs → it queries for a `CustomNotificationType` where `Desktop=true AND
Mobile=true`, finds **none**, so `customNotifTypeId` is null → the Send Custom Notification
action throws → **unhandled fault rolls back the entire Email-to-Case transaction** → no
Case, no EmailMessage, no agent reply, and (notify off) no bounce. The trace even shows the
Case/EmailMessage IDs that were created-then-rolled-back.

**Find which flows can fire on Case/EmailMessage creation** (note: `FlowDefinitionView` is a
**standard** object — querying it with `--use-tooling-api` fails `INVALID_TYPE`):
```bash
sf data query --query "SELECT ApiName, TriggerType, TriggerObjectOrEventLabel FROM FlowDefinitionView WHERE IsActive = true AND TriggerObjectOrEventLabel IN ('Case','EmailMessage')" --target-org <alias>
```
Then check the org has the records the flow depends on:
```bash
sf data query --query "SELECT DeveloperName, MasterLabel FROM CustomNotificationType" --target-org <alias>
```

**Fixes (fastest first):**
- **Deactivate the offending SDO demo flow** if it's not part of your scenario (Setup →
  Flows → Deactivate). The agent's email reply does not depend on `SDO_Service_Email_Message_On_Create_or_Update`.
- **Or** create the missing dependency — a `CustomNotificationType` with **both Desktop and
  Mobile** enabled (Setup → Notification Builder → Custom Notifications) so the flow's query succeeds.
- **Or** add a fault path / null guard in the flow.

**Takeaway:** when enabling any feature that creates records (Email-to-Case, Web-to-Case),
audit **pre-existing record-triggered flows** on Case/EmailMessage first — a stock demo
flow with a synchronous, throwing action will silently roll back your new pipeline.

### 5.11 SOQL/CLI gotchas hit while debugging the email channel

| Symptom | Cause | Fix |
|---------|-------|-----|
| `No such column 'Name' on entity 'EmailServicesAddress'` | object has no `Name` field | use `LocalPart`, `EmailDomainName`, `IsActive` |
| `sObject type 'FlowDefinitionView' is not supported` | queried with `--use-tooling-api` | it's a **standard** object — drop the tooling flag |
| `sObject type 'EmailToCaseSettings' is not supported` | not a queryable sObject | configure via Setup UI / metadata, not SOQL |
| Inbound row shows `D` (delivered) but still no Case | "delivered to MTA" ≠ "case created"; a downstream flow/limit dropped it | enable error notification (5.9 rung 3) |

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
- [ ] (Service agent) Tested via `--use-live-actions` as the **agent user**, not just anonymous Apex as admin (§4.4)
- [ ] (Service agent) Custom PS grants the agent user class access + CRUD/FLS on every object/field the actions write, + `Knowledge__kav` read if used (§4.5)
- [ ] (Service agent) The **human** demo/admin user — not just the agent user — has FLS (profile or assigned permission set) on every custom field shown in the UI, AND those fields are on the page layout (§4.5, §3.1)
- [ ] (Service agent) `AgentforceServiceAgentUser` system PS assigned to the agent user BEFORE publish
- [ ] Record-triggered Flow side-effects (email/callout) run on an **async-after-commit** path so they can't roll back the triggering insert (§4.6)
- [ ] No reliance on outbound email in unverified trial orgs — Chatter/Task used for demo, email documented for production (§4.7)
- [ ] `.agent` matches the CLI-generated grammar (`subagent` vs `topic`); action outputs referenced in prompt text are first captured into variables (§4.1, §4.2)
- [ ] (Type switch) Treated Employee↔Service as a NEW agent with a unique `developer_name`; did NOT expect an in-place edit (§4.12)
- [ ] (Headless/Agent API) Updated the hardcoded agent ID in the external client (e.g. n8n `SF_AGENT_ID`) after the type switch and smoke-tested the channel end-to-end (§4.13)
- [ ] (MCP actions) Agent user has access to the MCP Named/External Credential; validated `mcpTool://` actions via the MCP SDK client because they may not surface in authoring-bundle preview (§4.14)
- [ ] (Email channel) Routing address **Email Address** is a real, **Verified** inbox; test email sent to the **generated `@…case.salesforce.com` Email Services Address** pulled verbatim via SOQL (§5.3, §5.4)
- [ ] (Email channel) Routing address **Case Owner = Einstein Agent User**; that user has Case Create/Read/Edit (§5.5)
- [ ] (Email channel) **Service Email connection** added in Agent Builder (new Connections layout) and the agent version **activated** before saving the Email Configuration (§5.6)
- [ ] (Email channel) Email Template contains the literal `[[[GENERATED_CONTENT]]]` placeholder; Signature + Legal Disclosure filled (§5.7)
- [ ] (Email channel) **"Notify sender about Email-to-Case processing errors"** enabled while debugging so silent failures surface (§5.9)
- [ ] (Email channel) Audited **pre-existing record-triggered flows on Case/EmailMessage** (e.g. stock SDO demo flows) — none throws synchronously and rolls back the Email-to-Case insert (§5.10)
