# Project Specification

This document defines the **normative engineering constraints** for `syno-chatgpt-drive`.

Terms such as **MUST**, **MUST NOT**, **SHOULD**, and **MAY** are used in the RFC sense. When implementation convenience conflicts with this document, this document wins.

# P0 — Container-only development and runtime

This is a **non-negotiable, highest-priority project rule**.

> **Nothing required to build, test, lint, debug, package, or run this project may be installed on the host. All project dependencies and all project execution MUST happen inside containers.**

## Mandatory rules

1. **The host MUST remain clean.**  
   Do not install Rust, Cargo, rustup, compiler toolchains, native libraries, package-manager dependencies, MCP SDKs, test tools, linters, formatters, or application runtime dependencies on the host.

2. **All development commands MUST execute in a container.**  
   This includes, without exception:
   - `cargo build`
   - `cargo check`
   - `cargo test`
   - `cargo fmt`
   - `cargo clippy`
   - integration tests
   - conformance tests
   - local MCP server execution
   - Synology API diagnostics that depend on project tooling
   - release builds

3. **All dependency installation MUST be declared in container build definitions.**  
   Required OS packages, Rust toolchains, Cargo dependencies, certificates, and helper binaries belong in a `Dockerfile` or another OCI build definition checked into this repository.

4. **No setup instruction may require host package installation.**  
   Documentation, scripts, CI, and agent instructions MUST NOT contain steps such as:
   ```text
   apt install ...
   brew install ...
   rustup ...
   cargo install ...
   npm install ...
   pip install ...
   ```
   when those commands would execute on the host.

5. **The only supported local execution path is containerized.**  
   A developer may bind-mount the source tree into a development container or build an image from the repository, but the compiler and application MUST run inside that container.

6. **Build and runtime environments MUST be reproducible.**  
   Toolchain and base-image versions SHOULD be pinned. A clean machine with only a compatible container engine and Git access SHOULD be able to build and run the project without installing language/runtime dependencies.

7. **Agents MUST preserve this invariant.**  
   Any coding agent working on this repository MUST treat host installation as prohibited. If a task appears to require a host dependency, the agent MUST instead modify the container image, build stage, or containerized development workflow.

8. **CI MUST use the same containerized path.**  
   CI may provide a container runtime, but the project build/test/lint steps themselves MUST execute inside the project's container image or declared build stages.

## Allowed host responsibilities

The host may provide infrastructure needed to launch containers, for example:

- Docker / another OCI-compatible container engine;
- Git or the mechanism used to obtain the source tree;
- bind-mounted source/data directories;
- network access required by the containers.

These are infrastructure responsibilities, not application dependencies.

## Enforcement

A change is **non-compliant** if it:

- requires Rust/Cargo or other project tooling on the host;
- documents host-side dependency installation;
- adds a test or build path that only works outside the container;
- assumes host libraries are present;
- bypasses the container image for local execution.

Such a change MUST NOT be merged until the containerized path is restored.

---

# P0 — MCP architecture

The MCP server MUST follow the project's stateless MCP v2 architecture:

- MCP protocol target: **2026-07-28**
- Rust implementation using the official `modelcontextprotocol/rust-sdk` / `rmcp`
- per-request identity
- no authorization dependency on `Mcp-Session-Id`
- no sticky-session requirement
- no process-local current-user state
- per-user downstream Synology credential resolution
- Synology remains the authoritative ACL enforcement point

Durable external state such as encrypted credential bindings, refresh metadata, revocation information, and audit events is allowed. MCP transport/session state MUST NOT become the authority for user identity.

# P0 — Authorization invariant

For principals A and B:

> A request authenticated as A MUST NOT receive a Synology credential, cache entry, search result, file metadata, file content, or authorization decision belonging exclusively to B.

The MCP layer MUST NOT replace Synology's native ACL model with a second authoritative folder-permission system.

# P1 — Initial functional scope

The first production-oriented milestone is read-only:

- list files/folders
- search files
- get file metadata
- download/read files
- preserve Synology authorization failures

Write operations are deferred until delegated identity and cross-user isolation are proven end to end.

# P1 — Development mode

A fixed DSM username/password mode MAY exist only for a single-user PoC.

It MUST be clearly labeled as development-only and MUST NOT be used to claim multi-user ACL preservation.

The production direction is delegated per-user Synology identity.

# Acceptance criteria

Before the project is considered ready for shared use:

1. the repository builds from a clean host using only the documented container workflow;
2. no host Rust/toolchain installation is required;
3. MCP v2 2026-07-28 requests work through the containerized server;
4. two users with different Synology permissions receive correctly different results;
5. requests may be handled by different MCP replicas without identity leakage;
6. no shared privileged DSM account is used as a silent fallback.
