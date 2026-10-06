# Architecture

## 0. P0 execution constraint

The project is **container-only**.

- The host MUST NOT install Rust, Cargo, native project libraries, test tools, or application dependencies.
- Build, test, lint, debug, conformance testing, and server execution MUST run inside containers.
- Dependency installation belongs in checked-in OCI/Docker build definitions.
- Development agents MUST change the container workflow rather than install anything on the host.
- CI MUST exercise the same containerized build/test path.

This constraint has higher priority than developer convenience. See [Project Specification](SPEC.md).


## 1. Objective

`syno-chatgpt-drive` provides Synology Drive access to ChatGPT and other MCP hosts without collapsing all callers into a shared NAS identity.

The architectural rule is:

> **Authenticate at MCP, delegate identity downstream, authorize at Synology.**

Synology remains the authoritative policy enforcement point for users, groups, shared folders, Drive permissions, and document visibility.

## 2. High-level architecture

```text
+---------------------+
| ChatGPT / MCP Host  |
+----------+----------+
           |
           | MCP v2 request
           | authenticated end-user principal
           v
+----------+----------+
| Stateless MCP       |
| HTTP Endpoint       |
| the `rmcp` Streamable HTTP server  |
+----------+----------+
           |
           | RequestContext
           v
+----------+----------+
| Identity Resolver   |
| principal -> user   |
+----------+----------+
           |
           v
+----------+----------+       +----------------------+
| Credential Resolver|<----->| External Secret Store|
+----------+----------+       | token/link metadata  |
           |                  +----------------------+
           |
           | user-specific delegated credential
           v
+----------+----------+
| Synology Adapter    |
| Drive / Office API  |
+----------+----------+
           |
           v
+----------+----------+
| Synology DSM/Drive  |
| native ACL enforced |
+---------------------+
```

## 3. Stateless MCP v2 contract

The server targets the MCP v2 SDK implementing the 2026-07-28 protocol revision.

### Required properties

1. **Per-request server context**  
   HTTP requests are handled independently. The implementation should use `createMcpHandler(factory)`.

2. **No transport-session authority**  
   The modern path must not depend on `Mcp-Session-Id`, sticky load balancing, or an in-memory user session.

3. **Identity comes from the current request**  
   Authorization decisions must never use "the last user seen by this worker" or other process-local state.

4. **Any worker may serve any request**  
   Horizontal replicas must be interchangeable.

5. **Explicit state only when protocol flow requires it**  
   If a multi-round interaction requires `requestState`, it must be integrity protected, bound to the authenticated principal and operation, and expire quickly.

6. **External persistence is allowed**  
   Stateless MCP does not prohibit durable storage. Credential bindings, refresh tokens, revocation information, and audit logs belong in external services.

## 4. Request lifecycle

For a normal read operation:

```text
1. Receive MCP request
2. Validate MCP authorization
3. Extract stable principal identifier
4. Resolve principal -> Synology identity binding
5. Resolve/refresh downstream Synology credential
6. Create a request-scoped Synology client
7. Call Drive API
8. Let Synology apply native ACL
9. Return result
10. Discard request-scoped client/context
```

No step requires an MCP session stored in the worker.

## 5. Identity and authorization boundaries

There are two separate security relationships:

### A. MCP host -> MCP server

This answers:

> Who is calling this MCP endpoint?

The MCP access token is validated for issuer, audience, expiry, scopes, and principal identity.

### B. MCP server -> Synology

This answers:

> Which Synology identity is this request acting as?

A downstream credential must be resolved for the authenticated principal.

These credentials are **not assumed to be interchangeable**. A token issued for the MCP resource server must not be forwarded to Synology unless Synology explicitly accepts that issuer/audience and the deployment is designed for it.

## 6. Synology is the policy enforcement point

The MCP layer must not replicate the complete Synology ACL model.

Correct:

```text
principal A -> Synology credential A -> Synology decides access
principal B -> Synology credential B -> Synology decides access
```

Incorrect:

```text
all principals -> service account -> MCP-maintained path allow-list
```

The latter creates two permission systems that can drift.

## 7. Credential resolver

The credential resolver is the key abstraction.

Conceptual interface:

```rust
#[async_trait]
trait SynologyCredentialResolver {
    async fn resolve(&self, principal: &Principal) -> Result<SynologyCredential>;
}
```

Possible backends:

- direct Synology OAuth access/refresh token;
- an external token broker that exchanges or refreshes a Synology user session;
- development-only fixed credential.

Production code must not rely on a global `SYNO_USERNAME` / `SYNO_PASSWORD` to serve multiple end users.

## 8. Request-scoped client

Drive tools should be independent of authentication mechanics.

Desired dependency flow:

```rust
let principal = request_context.principal();
let credential = resolver.resolve(&principal).await?;
let synology = synology_client_factory.create(credential);

drive_tools.search(&synology, input).await
```

This makes the existing Drive tool logic reusable while replacing single-user authentication.

## 9. Caching

Caching is permitted only if authorization isolation is preserved.

Every cache key containing Synology data must include at least:

- stable principal or effective Synology identity;
- resource identifier/path;
- permission-sensitive variant where relevant.

Never share cached file contents or search results across identities unless the data is explicitly public and the policy is proven independent of user ACL.

## 10. Backward compatibility

The primary design target is MCP v2 / 2026-07-28.

If legacy 2025-era MCP clients are supported, they should use the SDK's stateless legacy compatibility path. Legacy compatibility must not introduce a requirement for sticky sessions into the modern architecture.

## 11. Deployment model

The target runtime supports ordinary horizontal scaling:

```text
               +----------------+
MCP requests ->| Load Balancer  |
               +--+----------+--+
                  |          |
             +----v---+  +---v----+
             | MCP A  |  | MCP B  |
             +----+---+  +---+----+
                  |          |
                  +-----+----+
                        |
               +--------v---------+
               | Credential Store |
               +--------+---------+
                        |
               +--------v---------+
               | Synology Drive   |
               +------------------+
```

No session affinity is required for modern MCP traffic.

## 12. Observability

Audit records should include:

- request/correlation ID;
- MCP principal ID;
- effective Synology identity;
- tool name;
- target resource identifier;
- success/failure;
- Synology authorization failure when applicable;
- latency.

Never log bearer tokens, refresh tokens, passwords, file contents, or raw authorization headers.
