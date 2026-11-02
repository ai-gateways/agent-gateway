# Inside the agent: caller credentials and correlation headers

Agent Gateway sees the hop between the client and the agent. To see the model and tool calls the agent makes for each task, the agent sends those calls through Bifrost too, carrying two things: who the caller is, and which conversation and task the call belongs to.

## 1. Forward the caller's credential

Set `forward_accepted_credential: true` on the registration (dashboard: **Forward Bifrost authentication**). Bifrost then passes the credential it accepted from the caller to the agent. The agent reads it from the inbound request and reuses it on its Bifrost LLM and MCP requests. For a caller that authenticated with `x-bf-vk`, forward that same header value.

The caller's key needs its own grants for the providers, models, and MCP tools the agent uses. An agent grant does not cover them.

If the agent's own credential and the forwarded credential use the same header, read [If both use the same header](https://docs.getbifrost.ai/features/agent-gateway/authentication#if-both-use-the-same-header) before enabling forwarding.

## 2. Attach correlation headers

| Header | Value from the A2A server's execution context |
|---|---|
| `x-bf-session-id` | Context ID: groups the whole conversation |
| `x-bf-agent-correlation-id` | Task ID: groups the calls for one task |

Use the IDs your A2A server assigns. Do not substitute the JSON-RPC request ID, the message ID, or a tracing UUID.

LLM call from the agent:

```bash
curl http://localhost:8080/v1/chat/completions \
  -H "Content-Type: application/json" \
  -H "x-bf-vk: $CALLER_BIFROST_VK" \
  -H "x-bf-session-id: $A2A_CONTEXT_ID" \
  -H "x-bf-agent-correlation-id: $A2A_TASK_ID" \
  -d '{"model": "openai/gpt-4o-mini", "messages": [{"role": "user", "content": "Summarize the findings."}]}'
```

MCP tool call from the agent (use the exact tool name registered in Bifrost):

```bash
curl 'http://localhost:8080/v1/mcp/tool/execute?format=chat' \
  -H "Content-Type: application/json" \
  -H "x-bf-vk: $CALLER_BIFROST_VK" \
  -H "x-bf-session-id: $A2A_CONTEXT_ID" \
  -H "x-bf-agent-correlation-id: $A2A_TASK_ID" \
  -d '{"id": "lookup-1", "type": "function", "function": {"name": "<registered-tool-name>", "arguments": "{}"}}'
```

With the OpenAI Python SDK, build the credential and headers from the current task on every call so concurrent tasks never share them:

```python
from openai import OpenAI

def call_model(task, prompt):
    client = OpenAI(base_url="http://localhost:8080/openai", api_key=task.caller_vk)
    return client.chat.completions.create(
        model="openai/gpt-4o-mini",
        messages=[{"role": "user", "content": prompt}],
        extra_headers={
            "x-bf-vk": task.caller_vk,              # forwarded caller credential
            "x-bf-session-id": task.context_id,     # A2A context ID
            "x-bf-agent-correlation-id": task.id,   # A2A task ID
        },
    )
```

`task` stands for your A2A framework's execution context; read the IDs and forwarded credential from it.

## Pitfalls

- Work that continues after the inbound HTTP request ends must keep the task's credential and IDs in that task's own state. A shared mutable header map can attribute one caller's work to another.
- Follow-up messages keep the context ID; a new task in the same conversation gets a new task ID.
- If a tab is missing in Agent Logs, check the outgoing headers, logging settings, the time window, and the caller's access before assuming no calls were made.

Docs: [Connect model and tool calls](https://docs.getbifrost.ai/features/agent-gateway/integrating-your-agent#attach-the-correlation-headers).
