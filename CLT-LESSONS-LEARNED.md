# Custom Lightning Type (CLT) Rendering — Lessons Learned

## Problem
The invoiceResultCard LWC component was NOT rendering as a rich visual card in the Agentforce conversation. The agent outputted plain text instead of the custom component, despite all Lightning Type configuration (schema.json, renderer.json, meta.xml) being correct.

## Root Cause
The **Agent Script** had `filter_from_agent: True` on the action output and lacked the specific planner directives that tell the LLM to render the CLT component.

---

## Lesson 1: `filter_from_agent` Must Be `False` for Visual Outputs

- `filter_from_agent: True` hides the action output from the LLM entirely
- If the LLM never sees the output, it cannot include the renderable component in its response
- The CLT rendering pipeline only triggers when the LLM includes the component in its response
- **Use `True` only for outputs the LLM tracks internally but never displays visually**

```yaml
# WRONG — card will NOT render
outputs:
    extractionResult: object
        filter_from_agent: True   # <-- hides output from LLM
        is_displayable: True

# CORRECT — card renders
outputs:
    extractionResult: object
        filter_from_agent: False  # <-- LLM can include component in response
        is_displayable: True
```

## Lesson 2: Action Description Must Include `__show_tool_results__`

The action description must contain a special directive that tells the planner to trigger CLT rendering:

```yaml
# WRONG
extract_invoice:
    description: "Extracts data from an invoice."

# CORRECT
extract_invoice:
    description: "The output is always renderable; display it with __show_tool_results__. Always display the raw results, do not summarize. Display the output caption text as a blank string. Do NOT output any text outside the rendered card. Extracts data from an invoice."
```

## Lesson 3: Output Description Must Instruct "Do NOT Convert to Text"

Without this, the LLM may serialize the structured output as plain text instead of rendering it:

```yaml
# WRONG
outputs:
    extractionResult: object
        description: "Structured invoice extraction result"

# CORRECT
outputs:
    extractionResult: object
        description: "The output of this action is always renderable. Always use show_command to display this to the user. Do NOT convert to text. Structured invoice extraction result displayed as a rich card."
```

## Lesson 4: Reasoning Instructions Must Explicitly Say "Include the Component"

```yaml
# WRONG — tells LLM to be silent, so it never includes the component
reasoning:
    instructions: |
        Never output text. Always call the action. The result card displays automatically.

# CORRECT — explicitly tells LLM to include the rendered component
reasoning:
    instructions: |
        1. Call the action with the submitted data.
        2. ALWAYS include the returned LWC component in your response. Do not describe it in text.
        Even if the same document was already processed, you MUST call the action again.
```

## Lesson 5: The LWC, Lightning Type, and Renderer Were Never the Problem

Through 15 versions of debugging, changes were made to:
- The LWC component (removing lightning-icon, changing @api patterns, simplifying HTML)
- The Lightning Type schema.json
- The renderer.json format
- The meta.xml targets and apiVersion
- Deleting orphan Lightning Types

**None of these were the issue.** The component rendered correctly in the Lightning Type UI Preview from the start. The problem was entirely in how the Agent Script communicated with the planner about displaying the output.

## Lesson 6: Compare Against a Working Agent Script

The fastest path to the answer was retrieving a known-working Agent Script (Uphold Case Analysis Agent) from the same org and doing a side-by-side comparison. The differences were immediately obvious:

| Setting | Broken (AP Invoice) | Working (Uphold) |
|---------|---------------------|-------------------|
| `filter_from_agent` | `True` | `False` |
| Action description | Generic | Includes `__show_tool_results__` |
| Output description | Generic | Includes "use show_command" |
| Reasoning | "Never output text" | "ALWAYS include the returned LWC component" |

---

## Quick Checklist for CLT Rendering in Agent Script

- [ ] `filter_from_agent: False` on the output
- [ ] `is_displayable: True` on the output
- [ ] `complex_data_type_name` set to the Lightning Type name (e.g., `c__invoiceResult`)
- [ ] Action description includes `__show_tool_results__` directive
- [ ] Output description includes "Do NOT convert to text"
- [ ] Reasoning instructions say "ALWAYS include the returned LWC component"
- [ ] Lightning Type schema.json exists with `@apexClassType` binding
- [ ] `lightningDesktopGenAi/renderer.json` points to the correct LWC
- [ ] LWC meta.xml has `lightning__AgentforceOutput` target and matching `sourceType`
- [ ] LWC exposes `@api value` property
- [ ] GenAiFunction output schema has `copilotAction:isDisplayable: true`
- [ ] Agent Action "Output Rendering" dropdown is set to the Lightning Type name
- [ ] Agent Action "Show in conversation" checkbox is checked
