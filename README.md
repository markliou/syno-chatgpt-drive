# syno-chatgpt-drive

A **stateless MCP v2 server** that lets ChatGPT and other MCP hosts access Synology Drive while preserving each user's **native Synology identity and ACLs**.

The goal is deliberately narrow:

> Authenticate every MCP request as an end user, resolve that user to a delegated Synology credential, and let Synology remain the source of truth for file authorization.

This project is **not** intended to create a second folder-permission system inside MCP.

## Design target

- **MCP v2 / 2026-07-28 protocol**
- **Stateless request handling**
- **No sticky sessions and no `Mcp-Session-Id` dependency**
- **Per-request user identity**
- **Per-user Synology delegated credentials**
- **Synology-native user/group/folder ACL enforcement**
- **Horizontally scalable MCP workers**
- **Read-only first**
- **Least privilege by default**

See [Architecture](docs/ARCHITECTURE.md) and [ADR-0001](docs/adr/0001-stateless-mcp-v2.md).

## Why this project exists

A conventional Synology MCP server is commonly configured with one service account:

```text
ChatGPT users
    |
    v
MCP server
    |
    v
single DSM service account
    |
    v
Synology Drive
```

That model flattens every caller into one Synology identity. The MCP layer then either exposes too much data or must reimplement access control.

`syno-chatgpt-drive` instead targets:

```text
ChatGPT / MCP host
        |
        | authenticated principal on every request
        v
Stateless MCP v2 server
        |
        | resolve downstream credential for this principal
        v
Synology Drive API
        |
        | authenticated as the actual Synology user
        v
Synology ACL / group membership / Drive permissions
```

Authorization remains in Synology.

## MCP v2 philosophy

The implementation targets the modern MCP protocol era introduced by the **2026-07-28** specification.

For HTTP serving this means:

- each request carries its own protocol/client context;
- the server does not depend on an MCP transport session;
- there is no `Mcp-Session-Id` on the modern path;
- any worker may handle any request;
- multi-round-trip state, if ever required, must use an explicit integrity-protected `requestState` handle rather than hidden server session state.

The TypeScript implementation should use the v2 `@modelcontextprotocol/server` package and `createMcpHandler()`.

**Stateless does not mean storage-free.** Long-lived identity links, encrypted downstream OAuth credentials, audit records, and revocation metadata may be stored externally. They must not become process-local conversational/session state.

## Authorization model

The target request path is:

```text
MCP request
   |
   +-- validated MCP/OAuth principal
   |
   v
Identity Resolver
   |
   +-- principal -> Synology account binding
   |
   v
Credential Resolver
   |
   +-- obtain/refresh that user's delegated Synology credential
   |
   v
Synology Drive client
   |
   v
Synology enforces native ACL
```

The MCP server must **not** maintain its own authoritative folder allow-list.

See [Authentication](docs/AUTHENTICATION.md).

## Initial tool scope

The first usable version is intentionally read-only:

- list files/folders
- search files
- get file metadata
- download/read a file
- expose clear authorization failures from Synology

Write operations such as upload, move, delete, share, and permission changes are out of scope until identity delegation and audit behavior are proven safe.

## Non-goals for v0.x

- Building a semantic RAG/indexing platform
- Reimplementing Synology ACLs in the MCP server
- Sharing one privileged DSM service account among end users
- Maintaining sticky MCP sessions
- Storing DSM passwords in application configuration
- Bypassing Synology permission checks

Content indexing or semantic retrieval can be added later as a separate layer, provided every result remains user-scoped.

## Status

**Design / PoC.**

The first technical question to validate is whether the Synology Drive/Office API used by the adapter can consume a user-specific DSM OAuth access token directly. If not, the fallback design is a stateless MCP worker plus an external per-user Synology session/credential broker.

See [Roadmap](docs/ROADMAP.md).

## References

- MCP TypeScript SDK v2: https://ts.sdk.modelcontextprotocol.io/v2/
- MCP 2026-07-28 support notes: https://ts.sdk.modelcontextprotocol.io/v2/migration/support-2026-07-28
- Synology OAuth Service: https://www.synology.com/dsm/feature/oauth_service
- Synology Productivity API: https://www.synology.com/dsm/feature/productivityapi
- Existing Synology Office MCP reference: https://github.com/vocweb/synology-mcp-server

## License

Not selected yet.
