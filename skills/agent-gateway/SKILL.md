---
name: agent-gateway
description: Set up an agent gateway for A2A (Agent2Agent protocol) traffic, using the Agent Gateway in Bifrost, the open-source AI gateway by Maxim AI. Registers agents from their Agent Card URL, grants each key access to specific agents (separate from model and MCP tool access), and gives callers one gateway endpoint over JSON-RPC, REST, or gRPC. Covers caller credential forwarding, correlation headers that link agent tasks to LLM and MCP calls, push notifications, Agent Logs, and handling agent output as untrusted input. Use when the user wants an agent gateway or A2A gateway, to register, route, secure, or observe agent-to-agent calls, to control which apps or users can call which agents, to bridge A2A transports, or to trace an agent's tasks together with its model and tool calls.
compatibility: Runs Bifrost with npx (Node.js) or Docker. Needs a running A2A v1 agent with a reachable Agent Card URL.
metadata:
  author: Maxim AI
  version: "1.0"
---

# Agent gateway (A2A)

An agent gateway sits between A2A clients and A2A agents. Callers reach each registered agent at a gateway URL; the gateway checks that the caller is granted that agent, forwards the call over the transport the agent supports, and records every operation.

This skill builds that gateway with the Agent Gateway in Bifrost, the open-source AI gateway (Apache 2.0) built by Maxim AI.

```
Your app (A2A client)
   |  A2A over JSON-RPC, REST, or gRPC + x-bf-vk
   v
Bifrost Agent Gateway  --- agent grant check, transport bridge, Agent Logs
   |  A2A over the agent's advertised interface
   v
A2A agent  --- its own model and tool calls --->  Bifrost LLM / MCP gateway
```

For the agent's model calls use the `llm-gateway` skill; for its tool calls, `mcp-gateway`; for all three kinds of traffic, `ai-gateway`.

## When to use

- Several applications or teams call shared A2A agents, and each caller should reach only the agents it is granted.
- Agents live on a private network and callers should see one stable gateway URL.
- A client speaks one A2A transport (JSON-RPC, REST, gRPC) and the agent serves another.
- You need to follow a conversation from the A2A task into the LLM and MCP calls the agent made.

## Steps

### 1. Start Bifrost and set the public base URL

Agent Gateway protocol endpoints return `503` until **External Client URL** is set. It is the URL clients and agents use to reach Bifrost; it is advertised in served Agent Cards and used for push callbacks. Bifrost does not infer it from request headers.

```json
{
  "client": {
    "enforce_auth_on_inference": true,
    "a2a_external_client_url": "https://bifrost.example.com"
  }
}
```

Put that in `config.json` in the app directory, or set it in the dashboard under **Agents, Agent Settings**. For a local trial use `http://localhost:8080`. Then start Bifrost:

```bash
npx -y @maximhq/bifrost
# or
docker run -p 8080:8080 -v $(pwd)/data:/app/data maximhq/bifrost
```

`enforce_auth_on_inference` matters here: without it, requests that present no identity pass through anonymously, and agent grants do not make anonymous traffic private.

If the agent's card or runtime is on a private network, add its hostname, IP, or CIDR to `client.a2a_private_allowlist` (no scheme, port, or path).

### 2. Create a virtual key for the caller

```bash
curl -s -X POST http://localhost:8080/api/governance/virtual-keys \
  -H "Content-Type: application/json" \
  -d '{"name": "research-app", "is_active": true}' | jq '.virtual_key | {id, value}'
```

Keep both: agent grants use the key's `id`; the caller sends the key's `value`.

### 3. Register the agent

You need the agent's full Agent Card URL, including the path, for example `https://research.example.com/.well-known/agent-card.json`. The registration name becomes part of the gateway URL: lowercase letters, digits, and single hyphens (63 characters or fewer if you will use gRPC).

Dashboard: **Agents, Agent Catalog, New Agent**, paste the card URL, **Fetch**, check the preview, fill in request authentication, select the virtual keys, **Create**.

Or `config.json`:

```json
{
  "agents": [
    {
      "name": "library-research",
      "agent_card_url": "https://research.example.com/.well-known/agent-card.json",
      "enabled": true,
      "allow_by_default": false,
      "virtual_key_ids": ["<virtual-key-id>"],
      "forward_accepted_credential": true,
      "runtime_auth": {
        "type": "headers",
        "headers": {"Authorization": "env.RESEARCH_AGENT_AUTHORIZATION"}
      }
    }
  ]
}
```

- `virtual_key_ids`: the keys granted this agent, by ID, never the secret value.
- `allow_by_default`: `true` lets any valid virtual key call the agent without an explicit grant.
- `runtime_auth`: only if the agent needs its own credential; set the variable to the full header value (`Bearer <agent-token>`).
- `forward_accepted_credential`: passes the caller's Bifrost credential to the agent so the agent can reuse it on its own model and tool calls.

Management API equivalent: `POST /api/agents` with the same fields (`PUT /api/agents/{name}` replaces a registration, so send every field you want to keep).

### 4. Call the agent through Bifrost

```bash
export BIFROST_VK="sk-bf-..."

# Discovery: the gateway's card, with endpoints rewritten to Bifrost
curl -s http://localhost:8080/agents/a2a/library-research/.well-known/agent-card.json \
  -H "x-bf-vk: $BIFROST_VK"

# A2A v1 JSON-RPC
curl -s http://localhost:8080/agents/a2a/library-research/json-rpc \
  -H "x-bf-vk: $BIFROST_VK" \
  -H "Content-Type: application/json" \
  -H "A2A-Version: 1.0" \
  -d '{
    "jsonrpc": "2.0",
    "id": "research-1",
    "method": "SendMessage",
    "params": {
      "message": {
        "messageId": "question-1",
        "role": "user",
        "parts": [{"text": "Explain how to reuse one HTTP client across requests."}]
      }
    }
  }'
```

Give A2A SDK clients the gateway card URL, not the agent's own card. REST is at `/agents/a2a/<name>/rest`. gRPC needs a listener, wildcard DNS, and a restart: see [references/transports-and-push.md](references/transports-and-push.md).

### 5. Link the agent's own LLM and MCP calls

Agent access is separate from model and tool access: a key granted an agent still needs provider and MCP grants for the calls the agent makes on its behalf. Inside the agent, on every Bifrost LLM and MCP request made for a task, send:

| Header | Value |
|---|---|
| `x-bf-vk` | The caller's forwarded credential (requires `forward_accepted_credential`) |
| `x-bf-session-id` | The A2A context ID |
| `x-bf-agent-correlation-id` | The A2A task ID |

Code patterns and pitfalls: [references/agent-integration.md](references/agent-integration.md).

## Treat agent output as untrusted

Messages, artifacts, and push updates from an agent can carry text that came from outside your system, including instructions aimed at the next model that reads them.

- Grant each key only the agents it needs. Because the agent reuses the caller's forwarded key, that key's model and MCP grants also cap what an injected instruction can make the agent do.
- In the calling application, treat artifacts and message parts as data. Review them before passing them to a model that has tool access.
- In your push callback, require the Basic or Bearer credential you registered and compare `A2A-Notification-Token` with the token you set. Bifrost already rejects agent pushes that carry an invalid notification token or belong to another task.

## Verify

1. The gateway card request returns `200`, and its endpoints point at your External Client URL.
2. `SendMessage` returns a message or a task.
3. The same call without `x-bf-vk`, or with a key that has no grant for this agent, is refused.
4. **Observability, Agent Logs** shows the operation. Open the **Agent session** (grouped by context ID); with step 5 in place, the **LLM Logs** and **MCP Logs** tabs show the agent's calls for that task.

## Gotchas

- `503` (or gRPC `UNAVAILABLE`) on every agent route: External Client URL is unset or resolves to empty.
- Registration fails with "runtime origin is not permitted": the card advertises an endpoint on another origin; allow it under **Additional agent endpoints** (`credential_origins` in the API).
- A2A version mismatch: the gateway accepts A2A v1 and rejects an explicitly incompatible `A2A-Version`.
- A transient upstream `5xx` is not replayed, because the agent may already have started the task. Fetch the task before retrying.
- Access profiles and projects (`x-bf-project-name`) for agent grants are Bifrost Enterprise features ([Enterprise access profiles](https://docs.getbifrost.ai/features/agent-gateway/access-control#enterprise-access-profiles), [Enterprise projects](https://docs.getbifrost.ai/features/agent-gateway/access-control#enterprise-projects)).

## Reference

- [references/transports-and-push.md](references/transports-and-push.md): gRPC listener, upstream transport choice and failover, push notifications.
- [references/agent-integration.md](references/agent-integration.md): forwarding caller credentials and correlation headers inside the agent.
- Bifrost docs: [Agent Gateway overview](https://docs.getbifrost.ai/features/agent-gateway/overview#start-with-one-agent), [Register your first agent](https://docs.getbifrost.ai/features/agent-gateway/registration#register-from-deployment-configuration), [Agent access](https://docs.getbifrost.ai/features/agent-gateway/access-control#grant-a-virtual-key-access), [Read Agent Logs](https://docs.getbifrost.ai/features/agent-gateway/observability#follow-a-session), [Troubleshooting](https://docs.getbifrost.ai/features/agent-gateway/troubleshooting#gateway-operations-return-503-or-unavailable).
- A2A protocol: [specification](https://a2a-protocol.org/latest/specification/#11-key-goals-of-a2a).
