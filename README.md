# Agent Gateway: Governing Agent-to-Agent Traffic

**TL;DR**

- An agent gateway is a control point between the applications or agents that delegate work and the remote agents that perform it, enforcing identity, access, routing, and logging on agent-to-agent traffic.
- The Agent2Agent (A2A) protocol, a Linux Foundation project since June 2025, is the main protocol an agent gateway governs today, with JSON-RPC, gRPC, and HTTP+JSON bindings.
- An agent gateway complements an MCP gateway (agent-to-tool) and an LLM gateway (agent-to-model), and permission to call an agent should stay separate from permission to use models and tools.
- The useful evaluation criteria are Agent Card handling, per-caller agent grants, transport translation, safe failure handling, session-level tracing, and self-hosted deployment.

An agent gateway is a network control point that sits between the callers that delegate work and the remote agents that perform it, applying authentication, access control, routing, and logging to agent-to-agent traffic. As teams move from one assistant to many specialized agents built by different teams or vendors, caller-to-agent connections grow faster than the agents themselves. Without a shared layer, nobody can answer which caller asked which agent to do what. This guide covers what an agent gateway does, how it handles the A2A protocol, how it differs from MCP and LLM gateways, and what to check when choosing one.

## What Is an Agent Gateway?

An agent gateway is a proxy for agent-to-agent requests. Callers send tasks to the gateway instead of to each agent directly. The gateway verifies the caller, checks that the caller may use the requested agent, forwards the task over a transport the agent supports, and records the exchange. Agents keep their own logic; the gateway owns the policy around them.

![Applications and orchestrator agents send A2A requests to one agent gateway, which checks identity and access, then forwards each task to an internal or partner agent](https://articles-images-cdn.t3.tigrisfiles.io/diagrams/ai-gateways-agent-gateway/agent-gateway-placement.png)

*Figure 1: Callers integrate once with the agent gateway instead of with every remote agent's endpoint and credentials.*

As Figure 1 shows, callers can be applications, batch jobs, or other agents. An orchestrator that splits a request into subtasks is itself a caller, so the gateway sees every delegation, not only the first hop.

## Why Agent-to-Agent Traffic Needs a Gateway

Agent-to-agent traffic needs a gateway because delegation multiplies trust decisions. Every caller needs credentials for every agent it uses, every agent must decide who may call it, and every long-running task needs an audit trail. Handling those concerns once, at a shared layer, is cheaper and easier to verify than handling them in each agent.

Teams that connect agents directly tend to hit the same problems:

- **Credential sprawl.** Each caller stores a separate key or OAuth client per agent, so rotating one credential means finding every caller that holds it.
- **Lost attribution.** When an orchestrator calls a billing agent for a user, the billing agent sees the orchestrator, not the user.
- **Inconsistent access rules.** One agent checks a header, another trusts the network, and a third accepts anonymous requests.
- **Fragmented logs.** One user request can produce a dozen tasks across five agents with no shared ID to join them.
- **Unsafe retries.** A client that retries a timed-out A2A request can start the same task twice.

The [OWASP Top 10 for Agentic Applications](https://genai.owasp.org/resource/owasp-top-10-for-agentic-applications-for-2026/) lists identity and privilege abuse among the main agentic risks. A gateway puts identity checks and grants in one reviewable place.

## How the A2A Protocol Works

The Agent2Agent (A2A) protocol is an open standard for one agent to discover another and send it work. Google introduced A2A in April 2025, and the [Linux Foundation launched the Agent2Agent project](https://www.linuxfoundation.org/press/linux-foundation-launches-the-agent2agent-protocol-project-to-enable-secure-intelligent-communication-between-ai-agents) in June 2025 with AWS, Cisco, Google, Microsoft, Salesforce, SAP, and ServiceNow. Version 1.0 is the latest released specification.

An agent gateway works with five A2A concepts:

| A2A concept | What it is | What an agent gateway does with it |
|---|---|---|
| Agent Card | JSON at a well-known URL listing the agent's skills, endpoints, and auth schemes | Rewrites its endpoints to gateway addresses and serves it to callers |
| Message | A turn from a user or agent, made of text, file, or data parts | Forwards it and records metadata |
| Task | A unit of work with an ID and a state such as working or completed | Tracks state and timing |
| Context | An ID grouping related tasks into one conversation | Uses it as the session key for logs |
| Artifact | An output of a task, such as a report or file | Records that it was produced |

The [A2A specification](https://a2a-protocol.org/latest/specification/) defines three bindings: JSON-RPC, gRPC, and HTTP+JSON. A card can advertise several, so client and agent need not match. Long-running work uses streaming, task subscriptions, or push notifications to a client webhook. Agent Cards declare security schemes such as API keys, OAuth 2.0, OpenID Connect, and mutual TLS; enforcing them consistently across many agents is the gateway's job.

## How an Agent Gateway Handles an A2A Request

An agent gateway processes each A2A request as a short pipeline: authenticate the caller, check its grant for the requested agent, choose an upstream transport from the agent's card, forward the request, and log the result. Requests that fail identity or grant checks stop before any task starts on the agent.

![An A2A request passes through caller authentication, an agent grant check, and transport selection before reaching the remote agent; rejected calls stop early and completed operations reach agent logs](https://articles-images-cdn.t3.tigrisfiles.io/diagrams/ai-gateways-agent-gateway/agent-gateway-request-pipeline.png)

*Figure 2: Identity and grant checks run before any work reaches the agent, so a rejected call never starts a task.*

Each stage in Figure 2 carries a design decision worth checking:

1. **Discovery.** Callers fetch the agent's card from the gateway, which returns endpoints pointing back at itself, so later calls cannot bypass it.
2. **Authentication.** The caller is identified by a key, an identity provider token, or a client certificate. A private gateway should reject requests with no identity.
3. **Grant check.** A grant for one agent should not imply access to every registered agent.
4. **Transport selection.** A client calling over JSON-RPC can reach an agent that only exposes gRPC.
5. **Recovery.** If an endpoint was unreachable, the gateway can try the next advertised interface. If the request may have reached the agent, replaying it risks a duplicate task, so the gateway should return the error and let the client look up the task.
6. **Logging.** The operation, task ID, context ID, transports, status, and latency are recorded.

## Agent Gateway vs MCP Gateway vs LLM Gateway

An agent gateway, an MCP gateway, and an LLM gateway govern three different hops in an AI system. The agent gateway controls who may delegate work to which agent. The [MCP gateway](https://github.com/ai-gateways/mcp-gateway) controls which tools an agent may call. The [LLM gateway](https://github.com/ai-gateways/llm-gateway) controls which models an agent may use and at what cost.

![An application calls an agent through the agent gateway; the agent then reaches models through the LLM gateway and tools through the MCP gateway, sharing session and task IDs](https://articles-images-cdn.t3.tigrisfiles.io/diagrams/ai-gateways-agent-gateway/agent-mcp-llm-gateways.png)

*Figure 3: One delegated task crosses three hops, and shared session and task IDs let logs from all three be joined.*

| | Agent gateway | MCP gateway | LLM gateway |
|---|---|---|---|
| Traffic | Caller to agent | Agent to tool server | Application or agent to model provider |
| Protocol | A2A | [Model Context Protocol](https://modelcontextprotocol.io/specification/latest) | Provider APIs, usually OpenAI-compatible |
| Unit of work | Task with a lifecycle | Tool call | Completion or embedding request |
| Policy question | May this caller delegate to this agent? | May this agent call this tool? | May this caller use this model, within what budget? |

The three hops should not share one permission: a caller allowed to use a research agent is not automatically allowed to use the most expensive model or a tool that writes to production data. When one gateway covers all three hops, the grants live in one policy model while remaining separate decisions.

## Security and Access Control for Agents

Access control for agents starts with one rule: permission to call an agent is a separate grant from permission to use models or tools. An agent that accepts a task should make its own model and tool calls under an identity that those layers check again, ideally the original caller's.

That rule leads to a few concrete controls:

- **Scoped grants.** Grant agent access per key, user, team, or project, and give new agents no access by default.
- **Enforced authentication.** Reject requests that carry no identity instead of passing them as anonymous traffic.
- **Credential forwarding.** Let the agent reuse the caller's credential on downstream model and tool calls, so attribution and the caller's limits survive the hop.
- **Destination allowlists.** Restrict which private addresses the gateway may reach, so a malicious Agent Card cannot point it at internal services.

## Observability for Multi-Agent Workflows

Observability for multi-agent workflows means following one user request across every agent, model call, and tool call it produced. The A2A context ID is the natural session key, and the task ID identifies each unit of work inside that session. A gateway that records both shows a complete conversation instead of scattered requests.

To see inside a task, the agent attaches the same session and task IDs, usually as headers, to its model and tool requests. When the LLM and MCP gateways read those IDs, one query returns the agent operations, model calls with tokens and cost, and tool calls. The [OpenTelemetry semantic conventions for generative AI](https://opentelemetry.io/docs/specs/semconv/gen-ai/) give these traces standard attribute names for existing tracing backends.

## How to Evaluate an Agent Gateway

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

The last two matter more for agents than for single model calls: one request can fan out into many tasks, so overhead and data residency apply to every hop.

## Open-Source Agent Gateway Projects

Open-source projects with documented A2A support are still few, since the protocol dates from April 2025. The projects below document A2A support in their own repositories or docs as of October 2026. Some API management platforms offer A2A proxying only in commercial editions, so they are not listed here.

| Project | Language | License | A2A support | Other traffic in the same gateway | Deployment |
|---|---|---|---|---|---|
| [Bifrost](https://github.com/maximhq/bifrost) | Go | Apache 2.0 | Registers agents from Agent Cards; JSON-RPC, REST, and gRPC transports; per-key agent grants; push notifications; agent logs linked to model and tool calls | LLM requests and MCP tool calls | Single binary via npx, Docker, Kubernetes, or Go SDK; self-hosted, including air-gapped |
| [ContextForge](https://github.com/IBM/mcp-context-forge) | Python | Apache 2.0 | Registers A2A agents by endpoint URL and exposes each one as an MCP tool | MCP servers and REST or gRPC APIs | PyPI, Docker, or Kubernetes |

Bifrost proxies A2A traffic as A2A, while ContextForge presents A2A agents as MCP tools; the fit depends on whether callers speak A2A directly or through an MCP client.

## Frequently Asked Questions

### What is the difference between an agent gateway and an API gateway?

An API gateway routes HTTP requests and applies generic policies such as rate limits and API keys. An agent gateway understands an agent protocol such as A2A: it reads Agent Cards, tracks task states, translates between A2A bindings, and links an agent's model and tool calls to the original caller. An API gateway can carry A2A traffic, but it sees requests, not tasks.

### Is A2A a replacement for MCP?

No. MCP connects an agent to tools and data sources, such as a database or a ticketing system. A2A connects one agent to another so it can delegate a task and receive results. Most production agent systems use both: A2A between agents and MCP between each agent and its tools, governed either by separate gateways or by one gateway that covers both protocols.

### Does an agent gateway control what an agent does after it accepts a task?

Only partly. The gateway controls who may send the agent a task and records the exchange. Which models and tools the agent then calls is governed by the LLM gateway and MCP gateway those calls pass through. Forwarding the caller's identity matters for this reason: it lets the downstream layers apply the caller's own permissions and limits.

### How does an agent gateway handle long-running A2A tasks?

A2A gives clients three ways to follow long work: streaming updates, subscribing to a task by ID, and push notifications sent to a webhook. A gateway should support all three, route callbacks through itself with separate callback credentials, and leave the task running when a client disconnects. Clients can then reconnect and fetch the task state instead of resubmitting the work.

### What authentication does A2A support?

An A2A Agent Card declares the security schemes an agent accepts, including API keys, HTTP authentication such as bearer tokens, OAuth 2.0, OpenID Connect, and mutual TLS. The protocol leaves enforcement to each agent. A gateway can authenticate callers once against the company's identity provider, then use separate per-agent credentials on the upstream call.

### Do I need an agent gateway if I run only one agent?

Usually not at first. With one agent and one caller, the agent's own authentication and logs are enough. A gateway becomes useful once several callers share agents, agents come from different teams or vendors, or auditors need one record of who delegated what. Adding it early costs little when the gateway already handles model and tool traffic.

## Related

- [Routing, failover, and cost controls for model traffic](https://github.com/ai-gateways/llm-gateway)
- [Registration, authentication, and access control for MCP tool servers](https://github.com/ai-gateways/mcp-gateway)
- [All AI Gateways guides](https://github.com/ai-gateways)
