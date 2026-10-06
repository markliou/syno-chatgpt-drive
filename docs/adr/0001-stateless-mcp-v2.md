# ADR-0001: Stateless MCP v2 as the Core Server Model

- **Status:** Accepted
- **Date:** 2026-10-06

## Context

This project exposes Synology Drive to ChatGPT and other MCP hosts for multiple end users.

The primary security requirement is to preserve each user's existing Synology identity and ACLs. A design based on one long-lived MCP session or one shared DSM account would either require sticky server state or duplicate Synology authorization logic.

MCP v2 implements the 2026-07-28 protocol revision. Its modern HTTP model is per request: it does not use `Mcp-Session-Id`, and the TypeScript v2 server entry point `createMcpHandler()` constructs request-serving state from a factory.

This aligns with the desired deployment model.

## Decision

The project SHALL use a **stateless MCP v2 architecture**.

For the modern MCP path:

1. every request is independently authenticated;
2. user identity is derived from the current request;
3. no request depends on sticky routing to a previous MCP worker;
4. no authoritative user login/session state is stored in MCP worker memory;
5. downstream Synology credentials are resolved per authenticated principal;
6. Synology performs final file authorization using its native ACL model.

Persistent identity bindings, encrypted delegated credentials, revocation metadata, and audit events MAY exist in external durable stores. These are application data, not MCP transport-session state.

## Consequences

### Positive

- horizontal scaling without session affinity;
- process restarts do not destroy user identity state;
- clean separation between MCP authentication and Synology authorization;
- native Synology ACLs remain authoritative;
- easier auditing because every request has an explicit principal;
- no need to build a second folder-permission engine.

### Negative

- downstream credential lookup/refresh occurs independently of worker affinity;
- shared external storage is required for multi-user credential bindings;
- credential refresh requires concurrency control;
- multi-round protocol interactions require explicit protected `requestState` rather than hidden server state.

## Implementation guidance

Use:

```text
@modelcontextprotocol/server v2
createMcpHandler(factory)
```

Do not make modern requests depend on:

```text
Mcp-Session-Id
sticky load balancing
global currentUser
in-memory per-user Synology session as source of truth
```

If `requestState` is introduced, it must be integrity protected, principal-bound, operation-bound where practical, and expiring.

## Compatibility

Supporting legacy 2025-era MCP clients is optional. If enabled, use a stateless compatibility path and do not let legacy support alter the modern architecture.

## Security invariant

For any two principals A and B:

> A request authenticated as A must never obtain a Synology credential, cache entry, search result, file body, or authorization decision belonging exclusively to B.

This invariant must be tested before production deployment.
