# Code Flow: delegate_task from tool call to sub-agent execution

## Entry Point: Tool Call

1. Model emits tool call: `{"name": "delegate_task", "arguments": {"task": "...", "context": "...", "background": false}}`
2. Middleware (`middleware.py:6030`) detects `delegate_task` calls and batches them
3. All non-delegate calls execute first, then delegate calls run in parallel via `asyncio.gather()`

## Step-by-step in `subagents.delegate()`

```
delegate(task, context, background, file_ids, request, user_data, metadata, parent_chat_id, parent_message_id)
│
├─ 1. VALIDATE: Check task not empty, parent_chat_id exists, user exists
│
├─ 2. LOAD CONFIG: Get subagent settings from DB
│   ├─ subagents.background_enabled
│   ├─ subagents.max_concurrent / max_async
│   ├─ subagents.max_iterations / max_output
│   └─ subagents.system_prompt
│
├─ 3. BUILD RUN CONFIG (THE KEY SECTION — line 317-332)
│   run = {
│       'model_id': metadata.get('model_id'),     ← ALWAYS parent model
│       'session_id': metadata.get('session_id'),
│       'tool_ids': [...],                          ← Inherited from parent
│       'skill_ids': [...],                         ← Inherited from parent
│       'system_prompt': metadata.get('system_prompt'),
│       'tool_servers': [...],
│       'filter_ids': [...],
│       'terminal_id': metadata.get('terminal_id'),
│       'features': {...},
│       'files': [...],
│       'variables': {...},
│       'direct': bool(metadata.get('direct')),
│   }
│
├─ 4. CONCURRENCY CHECK
│   ├─ If background: check _background_active < max_async
│   └─ If foreground: acquire semaphore (max_concurrent)
│
├─ 5. CREATE SUB-AGENT CHAT (in DB)
│   ├─ New chat_id = uuid4()
│   ├─ Title: "Sub-agent: {task[:60]}"
│   ├─ Messages: system_prompt + user task
│   └─ Internal meta: {parent_chat_id, delegation_id, mode}
│
├─ 6. BUILD CHILD REQUEST & FORM DATA
│   ├─ Create internal Request object
│   ├─ Set max_tool_call_iterations
│   ├─ Build form_data with:
│   │   ├─ model: run['model_id']        ← Parent model used here
│   │   ├─ messages: [system, user]
│   │   ├─ stream: True
│   │   ├─ chat_id: new chat
│   │   └─ tools, skills, filters, files, etc.
│   │
│   └─ Call: request.app.state.CHAT_COMPLETION_HANDLER(child_request, form_data, user=user)
│       ↑ This is the actual LLM API call — uses run['model_id']
│
├─ 7. EXTRACT RESULT
│   ├─ Read message content from completed chat
│   ├─ Truncate if > max_output
│   └─ Return {status, summary, error}
│
├─ 8. FOREGROUND: Return result directly to parent
│   └─ String: sub-agent output (or error)
│
└─ 9. BACKGROUND: Queue result as pending internal message
    └─ Gets processed later by process_pending_internal_messages()
```

## Key Insight: Where to Inject Model Selection

The model override must happen at STEP 3 (building the run config).
If `model` param is provided, replace `metadata['model_id']` with the specified model.
The rest of the flow (chat creation, handler call, result extraction) stays identical.

## Tool Registration Path

```
tools.py:get_tools()
  └─ Checks: subagents.enable && not internal && not direct
      └─ builtin_functions.extend([delegate_task, timer])

tools.py (later):
  └─ For each func:
      ├─ get_async_tool_function_and_apply_extra_params()
      ├─ get_builtin_tool_spec(func)       ← Generates JSON Schema for OpenAI tool calling
      │   └─ convert_function_to_pydantic_model(func)
      │       └─ Uses type hints + docstring → Pydantic model → JSON Schema
      └─ tools_dict[func.__name__] = {tool_id, callable, spec, type}
```

The tool spec (what the model sees as the function definition) is auto-generated from
the Python function signature. Adding a `model` parameter to `delegate_task` will
automatically expose it in the tool spec with its docstring description.
