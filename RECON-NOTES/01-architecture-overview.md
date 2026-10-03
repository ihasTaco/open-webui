# Sub-Agent Architecture Recon

## Overview
Open WebUI's sub-agent system lets a lead model spawn parallel helper agents that run tool-driven conversations and report results back. Currently sub-agents **always inherit the parent model** — there is no way to route a sub-agent to a different model.

## Key Files

| File | Role |
|---|---|
| `backend/open_webui/tools/builtin.py` | `delegate_task()` tool definition (line ~1680) |
| `backend/open_webui/utils/subagents.py` | Core delegation logic — `delegate()` function (line ~270) |
| `backend/open_webui/utils/middleware.py` | Intercepts `delegate_task` calls, batches them for parallel execution (line ~6030) |
| `backend/open_webui/utils/tools.py` | Tool registration and spec generation (line ~54, ~654, ~789) |
| `backend/open_webui/config.py` | Env var configuration (lines ~2036-2042) |

## Current Flow

1. Model emits a `delegate_task(task, context, file_ids, background)` tool call
2. Middleware intercepts it and routes to `subagents.delegate()`
3. `delegate()` creates a new chat with the **same model_id** as the parent
4. Sub-agent runs in a new chat, gets tools, produces output
5. Output is injected back into parent chat

## The Model Inheritance Problem

In `subagents.py` line 319:
```python
'model_id': metadata.get('model_id') or (metadata.get('model') or {}).get('id'),
```

The sub-agent always uses `metadata['model_id']` — the parent's model. There is NO `model` parameter on `delegate_task` that would allow the orchestrator to specify a different model.

## Config Vars

```
ENABLE_SUBAGENTS=True
SUBAGENTS_BACKGROUND_ENABLED=True
SUBAGENTS_MAX_CONCURRENT=20
SUBAGENTS_MAX_ASYNC=20
SUBAGENTS_MAX_ITERATIONS=30
SUBAGENTS_MAX_OUTPUT=30000
SUBAGENTS_SYSTEM_PROMPT=""
```

## The System Prompt Hook

`SUBAGENTS_SYSTEM_PROMPT` env var injects custom instructions into every sub-agent. This is where we could inject the model catalog/routing guide — but the sub-agent can't ACT on it because `delegate_task` doesn't accept a model parameter.
