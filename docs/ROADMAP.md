# Roadmap

## Phase 0 — Protocol and API validation

Goal: prove the two foundations before implementing a broad tool set.

**P0 prerequisite for every phase:** all development, build, test, diagnostics, and runtime execution must occur inside containers. No project dependency may be installed on the host.

- [ ] Add reproducible multi-stage Dockerfile for Rust build/runtime
- [ ] Add container-only development/test commands
- [ ] Create minimal MCP v2 Rust server using official `rmcp`
- [ ] Serve MCP 2026-07-28 over containerized Streamable HTTP
- [ ] Verify modern 2026-07-28 per-request operation
- [ ] Confirm no dependency on `Mcp-Session-Id` or sticky sessions
- [ ] Validate Synology Drive/Office API endpoint compatibility
- [ ] Determine whether DSM OAuth user access tokens work directly with required Drive APIs
- [ ] Document required DSM / Drive versions and OAuth scopes

**Exit criterion:** from a clean host with only a container engine, one authenticated user can execute a read-only Drive call through a stateless MCP v2 request.

## Phase 1 — Read-only delegated access

- [ ] MCP OAuth/auth middleware
- [ ] stable principal extraction
- [ ] identity binding model
- [ ] external credential resolver
- [ ] request-scoped Synology client
- [ ] list files/folders
- [ ] search files
- [ ] get metadata
- [ ] download/read file
- [ ] sanitize authorization errors
- [ ] structured audit logs without secrets

**Exit criterion:** two test users with different Synology ACLs receive different correct results through the same MCP endpoint.

## Phase 2 — Stateless scale and security

- [ ] run multiple MCP replicas
- [ ] verify no session affinity
- [ ] credential refresh concurrency control
- [ ] principal-scoped cache tests
- [ ] revocation/logout
- [ ] rate limiting
- [ ] secret-manager integration
- [ ] security tests for cross-user leakage
- [ ] MCP v2 conformance/compatibility tests
- [ ] optional stateless legacy-client compatibility

**Exit criterion:** requests can move freely between replicas without identity or authorization leakage.

## Phase 3 — ChatGPT deployment

- [ ] expose production MCP endpoint through approved network path
- [ ] configure OAuth metadata/client registration as required
- [ ] connect from ChatGPT
- [ ] validate per-user account linking
- [ ] verify native Synology ACL behavior end to end
- [ ] document admin deployment procedure
- [ ] document user onboarding/offboarding

**Exit criterion:** company users can connect one shared MCP service while preserving their individual Synology access rights.

## Phase 4 — Optional write operations

Only after the read path is proven safe:

- [ ] upload
- [ ] create folder
- [ ] move/rename
- [ ] delete
- [ ] share-link operations

Each write tool must receive an explicit security review.

Changing Synology ACLs through MCP is not planned for the initial project.

## Phase 5 — Optional content retrieval layer

File/metadata search is not the same as semantic knowledge retrieval.

A later layer may add:

```text
Synology Drive
    |
    v
document extraction/indexing
    |
    v
user-scoped semantic retrieval
    |
    v
MCP tools/resources
```

Any index must preserve source ACL semantics. Indexing must never turn inaccessible source documents into globally searchable embeddings or snippets.
