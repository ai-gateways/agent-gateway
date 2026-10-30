# Agent Gateway

An agent gateway is a network control point that sits between the callers that delegate work and the remote agents that perform it, applying authentication, access control, routing, and logging to agent-to-agent traffic. As teams move from one assistant to many specialized agents built by different teams or vendors, caller-to-agent connections grow faster than the agents themselves. Without a shared layer, nobody can answer which caller asked which agent to do what.

This guide explains what an agent gateway does, how it handles the A2A protocol, how it differs from MCP and LLM gateways, and what to check when choosing one.

## Contents

- [What is an agent gateway?](#what-is-an-agent-gateway)
- [Why agent-to-agent traffic needs a gateway](#why-agent-to-agent-traffic-needs-a-gateway)
- [How the A2A protocol works](#how-the-a2a-protocol-works)
- [How an agent gateway handles an A2A request](#how-an-agent-gateway-handles-an-a2a-request)
- [Agent gateway vs MCP gateway vs LLM gateway](#agent-gateway-vs-mcp-gateway-vs-llm-gateway)
- [Security and access control for agents](#security-and-access-control-for-agents)
- [Observability for multi-agent workflows](#observability-for-multi-agent-workflows)
- [How to evaluate an agent gateway](#how-to-evaluate-an-agent-gateway)
- [Open-source agent gateway projects](#open-source-agent-gateway-projects)
- [Related guides](#related-guides)

## What is an agent gateway?

An agent gateway is a proxy for agent-to-agent requests. Callers send tasks to the gateway instead of to each agent directly. The gateway verifies the caller, checks that the caller may use the requested agent, forwards the task over a transport the agent supports, and records the exchange. Agents keep their own logic; the gateway owns the policy around them.

```text
application         --+                                +--> internal agent: billing
orchestrator agent  --+--> agent gateway            --+--> internal agent: research
batch job           --+    (identity, access, logs)    +--> partner agent
```

Callers can be applications, batch jobs, or other agents. An orchestrator that splits a request into subtasks is itself a caller, so the gateway sees every delegation, not only the first hop.

## Why agent-to-agent traffic needs a gateway

Agent-to-agent traffic needs a gateway because delegation multiplies trust decisions. Every caller needs credentials for every agent it uses, every agent must decide who may call it, and every long-running task needs an audit trail. Handling those concerns once, at a shared layer, is cheaper and easier to verify than handling them in each agent.

Teams that connect agents directly tend to hit the same problems:

- **Credential sprawl.** Each caller stores a separate key or OAuth client per agent, so rotating one credential means finding every caller that holds it.
- **Lost attribution.** When an orchestrator calls a billing agent for a user, the billing agent sees the orchestrator, not the user.
- **Inconsistent access rules.** The A2A specification leaves [authorization logic to each agent server](https://a2a-protocol.org/latest/specification/#75-server-authorization-responsibilities), so one agent checks scopes, another trusts the network, and a third accepts anonymous requests.
- **Fragmented logs.** One user request can produce a dozen tasks across five agents with no shared ID to join them.
- **Unsafe retries.** A client that retries a timed-out A2A request can start the same task twice.

With one agent and one caller, the agent's own authentication and logs are usually enough. A gateway becomes useful once several callers share agents, agents come from different teams or vendors, or auditors need one record of who delegated what.

## How the A2A protocol works

The Agent2Agent (A2A) protocol is an open standard for one agent to discover another and send it work. Google introduced A2A in April 2025, and the [Linux Foundation launched the Agent2Agent project](https://www.linuxfoundation.org/press/linux-foundation-launches-the-agent2agent-protocol-project-to-enable-secure-intelligent-communication-between-ai-agents) in June 2025 with AWS, Cisco, Google, Microsoft, Salesforce, SAP, and ServiceNow. Version 1.0.0 is the [latest released specification](https://a2a-protocol.org/latest/specification/#agent2agent-a2a-protocol-specification).

An agent gateway works with five A2A concepts:

| A2A concept | What it is | What an agent gateway does with it |
|---|---|---|
| [Agent Card](https://a2a-protocol.org/latest/specification/#82-discovery-mechanisms) | JSON published at `/.well-known/agent-card.json` listing the agent's skills, interfaces, and auth schemes | Rewrites its endpoints to gateway addresses and serves it to callers |
| [Message](https://a2a-protocol.org/latest/specification/#37-messages-and-artifacts) | A turn from a user or agent, made of text, file, or data parts | Forwards it and records metadata |
| [Task](https://a2a-protocol.org/latest/specification/#41-core-objects) | A unit of work with an ID and a state such as working or completed | Tracks state and timing |
| [Context](https://a2a-protocol.org/latest/specification/#34-multi-turn-interactions) | An ID grouping related tasks into one conversation | Uses it as the session key for logs |
| [Artifact](https://a2a-protocol.org/latest/specification/#37-messages-and-artifacts) | An output of a task, such as a report or file | Records that it was produced |

The specification defines three protocol bindings: [JSON-RPC](https://a2a-protocol.org/latest/specification/#9-json-rpc-protocol-binding), [gRPC](https://a2a-protocol.org/latest/specification/#10-grpc-protocol-binding), and [HTTP+JSON/REST](https://a2a-protocol.org/latest/specification/#11-httpjsonrest-protocol-binding). A card can advertise several, so client and agent need not use the same one. For long-running work, clients follow progress through [polling, streaming, or push notifications](https://a2a-protocol.org/latest/specification/#35-task-update-delivery-mechanisms) to a client webhook. Agent Cards declare [security schemes](https://a2a-protocol.org/latest/specification/#45-security-objects) such as API keys, HTTP authentication, OAuth 2.0, OpenID Connect, and mutual TLS; enforcing them consistently across many agents is the gateway's job.

## How an agent gateway handles an A2A request

An agent gateway processes each A2A request as a short pipeline: authenticate the caller, check its grant for the requested agent, choose an upstream transport from the agent's card, forward the request, and log the result. Requests that fail identity or grant checks stop before any task starts on the agent.

```text
A2A request from caller
  -> authenticate caller ......... no identity: reject
  -> check agent grant ........... no grant for this agent: reject and log
  -> select transport from card .. JSON-RPC, gRPC, or HTTP+JSON
  -> forward to agent ............ unreachable before delivery: try next interface
  -> log operation, task ID, context ID, status, latency
response or task handle to caller
```

Each stage carries a design decision worth checking:

1. **Discovery.** Callers fetch the agent's card from the gateway, which returns endpoints pointing back at itself, so later calls cannot bypass it.
2. **Authentication.** The caller is identified by a key, an identity provider token, or a client certificate. A private gateway should reject requests with no identity.
3. **Grant check.** A grant for one agent should not imply access to every registered agent.
4. **Transport selection.** A client calling over JSON-RPC can reach an agent that only exposes gRPC.
5. **Recovery.** If an endpoint was unreachable, the gateway can try the next advertised interface. If the request may have reached the agent, replaying it risks a duplicate task, so the gateway should return the error and let the client look up the task.
6. **Logging.** The operation, task ID, context ID, transports, status, and latency are recorded.

For long-running tasks, the gateway should route push callbacks through itself with separate callback credentials, following the spec's [push notification security](https://a2a-protocol.org/latest/specification/#132-push-notification-security) guidance, and leave the task running when a client disconnects so the client can reconnect and fetch its state instead of resubmitting the work.

## Agent gateway vs MCP gateway vs LLM gateway

An agent gateway, an MCP gateway, and an LLM gateway govern three different hops in an AI system. The agent gateway controls who may delegate work to which agent. The [MCP gateway](https://github.com/ai-gateways/mcp-gateway#what-is-an-mcp-gateway) controls which tools an agent may call. The [LLM gateway](https://github.com/ai-gateways/llm-gateway#what-is-an-llm-gateway) controls which models an agent may use and at what cost.

```text
application
  -> agent gateway ... may this caller delegate to this agent?
       -> agent
            -> LLM gateway ... may this agent use this model, within budget?
            -> MCP gateway ... may this agent call this tool?
```

| | Agent gateway | MCP gateway | LLM gateway |
|---|---|---|---|
| Traffic | Caller to agent | Agent to tool server | Application or agent to model provider |
| Protocol | A2A | [Model Context Protocol](https://modelcontextprotocol.io/specification/2026-07-28/architecture#core-components) | Provider APIs, usually OpenAI-compatible |
| Unit of work | Task with a lifecycle | Tool call | Completion or embedding request |
| Policy question | May this caller delegate to this agent? | May this agent call this tool? | May this caller use this model, within what budget? |

A2A does not replace MCP. The specification describes the two as [complementary protocols](https://a2a-protocol.org/latest/specification/#appendix-b-relationship-to-mcp-model-context-protocol): MCP connects an agent to tools and data sources, and A2A connects one agent to another so it can delegate a task. Most production agent systems use both, governed either by separate gateways or by one gateway that covers both protocols.

The three hops should not share one permission: a caller allowed to use a research agent is not automatically allowed to use the most expensive model or a tool that writes to production data. When one gateway covers all three hops, the grants live in one policy model while remaining separate decisions.

## Security and access control for agents

Access control for agents starts with one rule: permission to call an agent is a separate grant from permission to use models or tools. An agent that accepts a task should make its own model and tool calls under an identity that those layers check again, ideally the original caller's.

The gateway controls who may send an agent a task and records the exchange. Which models and tools the agent then calls is governed by the LLM gateway and MCP gateway those calls pass through. That split leads to a few concrete controls:

- **Scoped grants.** Grant agent access per key, user, team, or project, and give new agents no access by default. The A2A specification requires servers to [scope results to the caller's authorized access boundaries](https://a2a-protocol.org/latest/specification/#131-data-access-and-authorization-scoping) on every operation.
- **Enforced authentication.** Reject requests that carry no identity instead of passing them as anonymous traffic. Authenticate callers once against the company's identity provider, then use separate per-agent credentials on the upstream call.
- **Credential forwarding.** Let the agent reuse the caller's credential on downstream model and tool calls, so attribution and the caller's limits survive the hop.
- **Destination allowlists.** Restrict which private addresses the gateway may reach, so a malicious Agent Card cannot point it at internal services.

## Observability for multi-agent workflows

Observability for multi-agent workflows means following one user request across every agent, model call, and tool call it produced. The A2A context ID is the natural session key, and the task ID identifies each unit of work inside that session. A gateway that records both shows a complete conversation instead of scattered requests.

To see inside a task, the agent attaches the same session and task IDs, usually as headers, to its model and tool requests. When the LLM and MCP gateways read those IDs, one query returns the agent operations, model calls with tokens and cost, and tool calls. OpenTelemetry defines standard attributes for this data, including [`gen_ai.conversation.id`](https://opentelemetry.io/docs/specs/semconv/registry/attributes/gen-ai/#gen-ai-conversation-id) and [`gen_ai.agent.name`](https://opentelemetry.io/docs/specs/semconv/registry/attributes/gen-ai/#gen-ai-agent-name), so traces land in existing tracing backends.

## How to evaluate an agent gateway

An agent gateway is worth evaluating on how well it understands A2A, how it separates permissions, how it fails, and where it runs. Generic API gateways can forward A2A over HTTP, but they rarely read Agent Cards, track task state, or link an agent's model and tool calls back to the caller who started the work.

| Criterion | What to check |
|---|---|
| Agent Card handling | Registers agents from their card URL, rewrites endpoints, and serves the current card |
| Agent grants | Per key, user, team, or project, separate from model and tool permissions |
| Transport coverage | JSON-RPC, HTTP+JSON, and gRPC, with client and agent free to differ |
| Failure behavior | Fails over only when a request never reached the agent; never replays streams |
| Long-running tasks | Routes push callbacks and supports task lookup after a disconnect |
| Tracing | Groups operations by context ID and links the agent's model and tool calls |
| Coverage | Governs model, tool, and agent traffic under one identity and policy model |
| Deployment | Self-hosted, in a private network, or fully air-gapped |
| Added latency | Published overhead measurements at realistic request rates |

The last two matter more for agents than for single model calls: one request can fan out into many tasks, so overhead and data residency apply to every hop. Adding a gateway early costs little when the same gateway already handles model and tool traffic.

## Open-source agent gateway projects

Open-source projects with documented A2A support are still few, since the protocol dates from April 2025. The projects below document A2A support in their own repositories or docs as of October 2026. Some API management platforms offer A2A proxying only in commercial editions, so they are not listed here.

| Project | Language | License | A2A support | Other traffic in the same gateway | Deployment |
|---|---|---|---|---|---|
| [Bifrost](https://github.com/maximhq/bifrost) | Go | Apache 2.0 | Registers agents from Agent Cards; JSON-RPC, REST, and gRPC transports; per-key agent grants; push notifications; agent logs linked to model and tool calls | LLM requests and MCP tool calls | Single binary via npx, Docker, Kubernetes, or Go SDK; self-hosted, including air-gapped |
| [ContextForge](https://github.com/IBM/mcp-context-forge) | Python | Apache 2.0 | Registers A2A agents by endpoint URL and exposes each one as an MCP tool | MCP servers and REST or gRPC APIs | PyPI, Docker, or Kubernetes |

Bifrost proxies A2A traffic as A2A, while ContextForge presents A2A agents as MCP tools; the fit depends on whether callers speak A2A directly or through an MCP client.

## Related guides

- [How an LLM gateway works](https://github.com/ai-gateways/llm-gateway#how-an-llm-gateway-works): routing, budgets, and failover for the model calls agents make
- [MCP gateway security](https://github.com/ai-gateways/mcp-gateway#mcp-gateway-security): token handling and tool poisoning on the agent-to-tool hop
- [AI Gateways](https://github.com/ai-gateways): all guides in this organization
