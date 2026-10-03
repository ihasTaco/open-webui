# Implementation Plan: Multi-Model Sub-Agents

## Changes Required

### 1. `builtin.py` — Add `model` parameter to `delegate_task`

Add a new parameter:
```python
model: str | None = None,  # Model ID or name to use for this sub-agent
```

Pass it through to `subagents.delegate()`.

### 2. `subagents.py` — Accept and use the model override

In `delegate()`:
- Accept the `model` parameter
- If provided, use it instead of `metadata['model_id']`
- Validate the model exists (check against available models)

### 3. `subagents.py` — Inject model catalog into system prompt

The `SUBAGENTS_SYSTEM_PROMPT` env var is already injected into sub-agent system prompts.
We can use this to provide a model catalog so the orchestrator knows what models are available.
But this is a separate concern — the sub-agent doesn't need to know about other models,
the ORCHESTRATOR needs to know so it can route tasks.

### 4. Add model-aware tool descriptions

Update the `delegate_task` docstring to tell the orchestrator which models are available:

```python
"""
Delegates focused work to a parallel sub-agent.

Available models:
- deepseek-v4.1-flash: Best for boilerplate, CRUD, scaffolding, simple bugs
- claude-sonnet-5: Best for UI layout, component architecture, general coding
- claude-opus-5.5: Best for debugging complex bugs, architecture, code review
- gpt-6.1-sol: Best for backend API, structured outputs, terminal workflows
- kimi-k3: Best for styling, visual polish, frontend CSS (expensive: $14.25/M output)
- gemini-3.1-pro: Best for whole-codebase analysis (2M context window)
- gpt-6-luna: Best for documentation, README, simple text tasks (cheapest: $0.10/$0.50)
...
"""
```

### 5. Model discovery mechanism

The orchestrator needs to know which models are available. Options:

**Option A**: Hardcode the model catalog in the SUBAGENTS_SYSTEM_PROMPT env var.
Simple, works immediately. Downside: static, must update manually.

**Option B**: Auto-discover from configured models. Add a `list_available_models` tool
that the orchestrator can call to see what's available. More dynamic.

**Option C**: Inject the model list into the delegate_task description at runtime
(from the admin-configured models). Best UX, most dynamic.

### 6. Model validation

When `model` is specified, validate against available models before spawning.
If the model doesn't exist or isn't configured, return a helpful error listing available models.

## Files to Modify

| File | Change |
|---|---|
| `backend/open_webui/tools/builtin.py` | Add `model` param to `delegate_task`, update docstring |
| `backend/open_webui/utils/subagents.py` | Accept `model` in `delegate()`, override `model_id` in `run` dict, validate model |
| `backend/open_webui/utils/tools.py` | Inject model list into tool description dynamically |
| `backend/open_webui/config.py` | No changes needed (existing env vars sufficient) |
| `backend/open_webui/utils/middleware.py` | No changes needed (model routing happens before middleware) |

## Backward Compatibility

- `model` parameter defaults to `None` — if not provided, behavior is identical to current
- Existing `SUBAGENTS_SYSTEM_PROMPT` still works
- No config changes required
