---
name: agentforce-slack-research
description: Research Agentforce technical issues by searching a curated set of internal Slack channels and canvases. Use when troubleshooting Agentforce agent behavior, Agent Script DSL issues, action invocation failures, Flow integration problems, or any Agentforce-related question that may have been discussed by the SE or field community.
---

# Agentforce Slack Research

Searches internal Salesforce Slack channels for Agentforce technical guidance, known issues, and community solutions.

## Channels to search (in priority order)

1. `#help-field-new-agentforce-builder-and-script` — Agent Script DSL, builder issues
2. `#agentforce-technical-support` — technical deep-dives and support cases
3. `#support-af-dc-issue-awareness` — known bugs and issue awareness
4. `#support-agentforce-swarming` — active swarming on field issues
5. `#mastering-agentforce-for-solution-engineers` — SE learning, best practices
6. `#agentforce-fde-learning-community` — FDE community knowledge
7. `#support-afdc-help` — DC-specific help
8. `#broadcast-agentforce-ai-doc` — doc releases and updates
9. `#broadcast-data-ai-technical-reference-guide` — reference guides
10. `#help-agentforce-data-library-and-rag-action` — RAG / data library actions
11. `#agentforce-proserv-help` — ProServ field guidance

## Workflow

1. **Formulate 2–3 targeted search queries** from the problem description (use specific technical terms: action names, error messages, variable types, DSL keywords).
2. **Search each relevant channel** using the Slack MCP `slack_search_messages` tool with each query.
3. **Read thread context** for promising hits using `slack_get_thread_replies` — surface answers buried in threads.
4. **Synthesise findings** into a concise answer: root cause, confirmed fix, workarounds, links to threads.
5. If nothing found in messages, **search canvases** in the same channels using `slack_search_messages` with `in:#channel-name` scoping.

## Output format

```
## Slack Research Findings

### Query used
- "<query 1>"
- "<query 2>"

### Relevant threads
- [#channel > @author]: Summary of finding. [link]

### Conclusion
Root cause / recommended fix / open question.
```

## Tips
- Agent Script DSL issues: search for the exact DSL keyword (`available_when`, `mutable`, `is_user_input`, `with`, `set`)
- Action not firing: search "action not called", "LLM skipping action", "slot fill", "required input"
- Flow not executing: search "flow interview", "SystemModeWithoutSharing", "live actions"
- Always check `#support-af-dc-issue-awareness` for known regressions before deep-diving
