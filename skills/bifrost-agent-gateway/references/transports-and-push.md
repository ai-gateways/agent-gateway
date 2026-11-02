# Transports, gRPC, and push notifications

## Client-facing endpoints

For a registration named `library-research` and External Client URL `https://bifrost.example.com`:

| Transport | Endpoint |
|---|---|
| Agent Card | `https://bifrost.example.com/agents/a2a/library-research/.well-known/agent-card.json` |
| JSON-RPC | `https://bifrost.example.com/agents/a2a/library-research/json-rpc` |
| HTTP+JSON (REST) | `https://bifrost.example.com/agents/a2a/library-research/rest` |
| gRPC (when enabled) | `library-research.a2a.example.com:443` |

The client's transport and the agent's transport are chosen independently, so a REST client can reach a JSON-RPC-only agent. Docs: [Copy the client-facing endpoints](https://docs.getbifrost.ai/features/agent-gateway/configuration#copy-the-client-facing-endpoints).

## Enable gRPC

gRPC clients call the same service methods for every agent, so Bifrost gives each agent its own hostname on one shared listener and routes by the HTTP/2 `:authority`.

```json
{
  "server": {
    "read_buffer_size": 65536,
    "a2a_grpc_base_domain": "a2a.example.com",
    "a2a_grpc_port": 443
  }
}
```

- Both fields are required, are read at startup, and need a restart to change.
- Point wildcard DNS `*.a2a.example.com` at the gateway's gRPC entry point. Dialing `a2a.example.com:443` alone does not identify an agent.
- The listener is plaintext gRPC. Terminate TLS at your ingress or load balancer and preserve the agent-specific authority when proxying.
- If the public port differs from the bind port, set `BIFROST_A2A_GRPC_LISTEN_PORT` (for example advertise `443`, bind `9095`).
- `read_buffer_size` is required by the server schema; keep your existing value.

Docs: [Enable gRPC](https://docs.getbifrost.ai/features/agent-gateway/configuration#enable-grpc).

## How Bifrost picks the agent-facing transport

Bifrost reads the agent card's `supportedInterfaces`, keeps entries with A2A v1, a supported binding (`JSONRPC`, `HTTP+JSON`, `GRPC`), and a valid URL, and prefers them in card order. To change the first choice, reorder the card served by the agent, not the Bifrost registration.

| Failure on a unary call | This call | Later calls |
|---|---|---|
| Dial failure before delivery | Tries the next interface | Start from the advanced interface |
| HTTP `404`, `405`, `410`, `426` | Tries the next interface | Advance and refresh the cached card |
| A2A method not found, gRPC `Unimplemented` | Tries the next interface | Advance |
| HTTP `5xx` or ambiguous network failure | Returns the error, no replay | May advance |
| Auth, validation, or protocol error | Returns the error | No transport change |

Interfaces on one card should share a task store, because Bifrost does not copy task state between endpoints. Docs: [Transport selection and recovery](https://docs.getbifrost.ai/features/agent-gateway/transports-and-recovery#what-happens-when-an-interface-fails).

## Push notifications

For long tasks, the caller registers a callback instead of holding a stream open. The upstream agent must support A2A push notifications.

1. Call the A2A create-task-push-configuration operation **through the Bifrost endpoint** with the task ID, your callback URL, and callback auth (Basic or Bearer). You can also include a push configuration when sending a message.
2. Bifrost swaps in its own callback URL, `<external-client-url>/agents/a2a/<agent-name>/push/callback`, and a separate token before calling the agent. Allow the agent to reach that route.
3. Your callback receives `POST` requests with `Content-Type: application/a2a+json`, plus `A2A-Notification-Token` if you set a token.
4. Any `2xx` counts as delivered. Timeouts and other statuses are retried up to six attempts, 30 seconds each; redirects are not followed. Delivery is at least once, so make the handler idempotent.
5. When updates are no longer needed, call the A2A delete-task-push-configuration operation through Bifrost. Configurations are not removed when a task finishes. Deleting in the dashboard (**Agent Gateway, Push Configurations**) removes only Bifrost's copy.

Bifrost blocks private and loopback callback URLs by default. For local development only:

```json
{
  "server": {
    "read_buffer_size": 65536,
    "a2a_allow_private_push_callbacks": true
  }
}
```

In Agent Logs, **Push notification** is the agent's update reaching Bifrost; **Push delivery** is Bifrost's attempt to reach your callback. A `202` to the agent means queued, not delivered. Docs: [Push notifications](https://docs.getbifrost.ai/features/agent-gateway/push-notifications#check-delivery-and-errors).
