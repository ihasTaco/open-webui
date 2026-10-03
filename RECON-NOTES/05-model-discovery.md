# Model Availability Detection for Sub-Agents

## Current State
Sub-agents cannot choose their model. The orchestrator model has no way to know which
models are available for delegation.

## Proposed Solution: Dynamic Model List in Tool Description

Instead of hardcoding model names, inject the admin-configured model list into the
`delegate_task` tool description at runtime.

### How it works:

1. Admin configures models in Open WebUI (Admin Settings → Models)
2. When tools are registered, `get_builtin_tool_spec()` generates the JSON schema
3. We modify the description to append available models
4. The orchestrator model sees the model list in the function description
5. The orchestrator picks the right model and passes it via the `model` parameter

### Alternative: `list_available_models` tool

A separate tool that returns configured models with their capabilities.
The orchestrator can call this first, then delegate with the right model.

Pro: Cleaner separation. Con: Extra round-trip.

### Recommended: Both

- Inject model list into `delegate_task` description for quick reference
- Also provide a `list_available_models` tool for detailed queries

## Model Descriptor Format

Each model in the list should include:
```json
{
  "id": "claude-sonnet-5",
  "name": "Claude Sonnet 5",
  "provider": "anthropic",
  "strengths": ["ui-layout", "component-architecture", "general-coding"],
  "cost_tier": "mid",
  "context": "1M"
}
```
