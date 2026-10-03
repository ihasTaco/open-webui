# delegate_task Tool — Detailed Analysis

## Current Signature (builtin.py ~line 1680)

```python
async def delegate_task(
    task: str,
    context: str = '',
    file_ids: list[str] | None = None,
    background: bool = False,
    __request__: Request = None,
    __user__: dict = None,
    __metadata__: dict = None,
    __chat_id__: str = None,
    __message_id__: str = None,
) -> str:
```

## What's Missing

The function has NO `model` or `model_id` parameter. The sub-agent **always inherits** the parent model via metadata. This means:

1. You cannot say "delegate this styling task to Kimi K3"
2. You cannot say "delegate this backend task to GPT-6.1 Sol"
3. You cannot say "delegate this boilerplate to DeepSeek V4.1 Flash"
4. Every sub-agent runs the same model as the orchestrator

## Where Model is Bound

In `subagents.py` delegate() function (~line 319):

```python
run = {
    'model_id': metadata.get('model_id') or (metadata.get('model') or {}).get('id'),
    ...
}
```

The `run` dict is the sub-agent's configuration. It's built from the parent's metadata.
The `model_id` is passed through to `CHAT_COMPLETION_HANDLER` which actually runs the LLM call.

## Recursion Prevention

Line ~1707 in builtin.py:
```python
if getattr(__request__.state, 'internal', False) is True:
    return 'Error: sub-agents cannot delegate recursively.'
```

Sub-agents cannot spawn further sub-agents. This prevents infinite delegation chains.

## Tool Spec Generation

In `tools.py` line ~789:
```python
if func.__name__ == 'delegate_task' and not config.get('subagents.background_enabled'):
    parameters = spec.get('parameters', {})
    parameters.get('properties', {}).pop('background', None)
```

The `background` parameter is conditionally removed from the tool spec based on config.
This pattern shows how we can conditionally modify the delegate_task spec — we'd add a `model` parameter the same way.

## Middleware Batching

In `middleware.py` lines ~6030-6045:
```python
delegate_calls = [
    tool_call
    for tool_call in response_tool_calls
    if tool_call.get('function', {}).get('name') == 'delegate_task'
]
# Non-delegate calls execute first, then delegate calls run in parallel
```

Delegate calls are gathered and executed in parallel via `asyncio.gather()`.
This means all sub-agents currently run the same model — but there's nothing preventing
different model calls per sub-agent since each call is independent.
