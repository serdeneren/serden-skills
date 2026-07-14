---
name: serden-agentforce-clone
description: >-
  Clone an Agent Script (.agent / AiAuthoringBundle) agent out of its
  environment into an isolated Salesforce org — a "workshop" — including
  rebuilding its dependent Flows and GenAiPromptTemplates as hard-coded
  mocks so the clone runs standalone, then publishing and activating it for
  preview testing, all without touching the original agent. Use when asked
  to clone, port, or stand up a copy of an existing Agentforce agent in a
  new/target org, or to mock its backing Flows/prompt templates so it can
  run independently of its original environment.
---

# Cloning an Agentforce Agent Script Agent Into a Workshop Environment

Like pulling an engine out of a car and putting it on a workbench: clone the
agent into an isolated org so it can be run, poked at, and experimented on
freely — without any risk to the original environment. Distilled from
cloning two versions of an Agent Script bundle into a fresh org this way,
including standing up mock Flows and prompt templates so the clone runs
standalone. Org names, bundle names, user handles, and IDs below are
placeholders — substitute the real values for your task.

Related skills to combine with this one:
- `developing-agentforce` / `agentforce-generate` / `sf-ai-agentscript` — Agent Script (`.agent`) syntax, actions, subagents, guards.
- `deploying-metadata` / `sf-deploy` — `sf project deploy start`, org auth.
- `generating-flow` — building the mock Autolaunched/Omni-Channel Flows.
- `agentforce-observe` / `platform-tracing-agentforce-configure` — trace inspection beyond what the CLI preview gives you.

## Workflow

1. Scaffold the DX project (see Pitfall 1).
2. Copy the source `.agent` file into a new `aiAuthoringBundles/<Bundle>/` folder with a matching naked `bundle-meta.xml` (Pitfall 2).
3. For every Flow/PromptTemplate the agent calls that doesn't exist in the target org, build a mock with hard-coded output matching the agent's expected variable types (Pitfalls 3–6).
4. Deploy, `sf agent publish authoring-bundle`, activate, and preview-test (Pitfall 7).

## Pitfall 1 — `sf config set target-org` fails with `InvalidProjectWorkspaceError`

The CLI requires a valid DX project even just to set the default org. Create a minimal `sfdx-project.json` at the repo root before doing anything else:

```json
{
  "packageDirectories": [{ "path": "force-app", "default": true }],
  "namespace": "",
  "sourceApiVersion": "62.0"
}
```

## Pitfall 2 — `AiAuthoringBundle` cloning: naked bundle vs `<target>`

A brand-new, independent clone needs a **naked** `bundle-meta.xml` (no `<target>` element):

```xml
<AiAuthoringBundle xmlns="http://soap.sforce.com/2006/04/metadata">
    <bundleType>AGENT</bundleType>
</AiAuthoringBundle>
```

`<target>SomeBotDeveloperName.v3</target>` is for basing a new version off an *existing* bundle you already deployed — it does not let you re-edit an already-published version in place, and pointing it at the wrong name can cause `sf project retrieve start` to pull down a differently-named bundle than you expect. If that happens, delete the stray retrieved folder and re-check your bundle name/target wiring before retrying.

**Immutability trap:** once an `AiAuthoringBundle` version is published, its `.agent` content cannot be changed via `sf project deploy start`, and it fails with:

```
content cannot be changed once the bundle version is published.
```

There is no metadata-deploy path to edit a published version's script. Plan for this:
- Get the `.agent` content right (or right enough) **before** your first publish.
- If you must keep iterating on behavior after publishing, do it through the **Flows and PromptTemplates** the agent calls (those stay mutable) rather than the `.agent` file itself.
- Treat "publish" as a checkpoint, not a save point.

## Pitfall 3 — Type mismatches between agent variables and mock Flow outputs

Agent Script `list[object]` variables often don't bind cleanly to Autolaunched Flow outputs you can quickly hard-code. The practical mock pattern: convert both the top-level `@variables` declaration **and** the corresponding action's output parameter from `list[object]` to `string`, and have the mock Flow emit a JSON string literal. Keep the `set @variables.X = @outputs.X` assignment unchanged — only the type annotations change, not the data flow.

## Pitfall 4 — Flow deployment gotchas

- **"Nothing is connected to the Start element"** — an Autolaunched Flow needs at least one element wired from Start; a trivial `Assignment` → flagged as an output variable is enough to satisfy this for a mock.
- **Flow silently missing after a combined deploy reporting `success: true`** — check for `rollbackOnError: true` in the deploy result. If another component in the same deploy failed, the whole transaction (including your seemingly-successful flow) can roll back. Redeploy the affected component(s) individually to confirm.
- **Omni-Channel Flow type** requires a variable literally named `recordId`, of type Text, marked "available for input" — without it, deployment fails with a specific metadata validation error naming that requirement.

## Pitfall 5 — GenAiPromptTemplate: required-input mismatches in preview

If an agent action calls a `GenAiPromptTemplate` with an input marked `required: true`, and your preview/mock context has nothing populated for it (e.g. no real Case ID), the action fails validation before it even runs, with an error like `must specify at least one input/output`. Fix by:
1. Editing the template to set that input's `required` to `false`.
2. Bumping the template version (e.g. `_1` → `_2`) — published prompt template versions are immutable the same way `.agent` bundles are.
3. Adding `<activeVersionIdentifier>` at the top level of the `-meta.xml` pointing at the version you actually want active — omitting this causes the same "must specify input/output" error even after fixing the field.
4. Give the agent variable feeding that input a mutable default (e.g. `mutable string = "PREVIEW_CASE"`) so preview sessions have something non-null to pass through.

## Pitfall 6 — Agent Script syntax gotchas when porting scripts across versions

- Boolean literals must be capitalized: `True`/`False`, not `true`/`false`. Older exported script versions can contain lowercase literals that fail validation on redeploy — grep for `= true` / `= false` and fix all instances.
- Watch for automatic validation warnings surfaced after edits; they catch this class of issue immediately.

## Pitfall 7 — Preview CLI mechanics

- `sf agent preview send` requires **both** `--api-name <BundleDeveloperName>` and `--session-id`; omitting either gives `Exactly one of the following must be provided: --api-name, --authoring-bundle`.
- Trace files from `sf agent preview` can come back as empty `{}` for some orgs/bundles. `turn-index.json` and `transcript.jsonl` reliably give you the conversational exchange, but not action-level trace detail — don't rely on CLI traces alone; use the Agentforce Builder UI preview pane for anything requiring per-action trace visibility.
- `--output-dir` is not a valid flag on `sf agent preview start` in current CLI versions.
- Session/agent variables persist **across turns within one preview session**. Start a fresh session per independent test case, or earlier state will contaminate later assertions.
- The agent executes as its configured `default_agent_user`, not your authenticated CLI/org user — your personal Apex debug trace flags won't capture its Flow invocations. There is no straightforward server-side log workaround for this from the CLI; rely on preview traces instead.

When investigating an issue on the clone, keep the cloned `.agent` script's structure identical to the source and only vary the mock Flow/PromptTemplate data around it — that's the whole point of the workshop: the engine (the agent's logic) stays untouched; only the test bench (mock data) around it changes.
