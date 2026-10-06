# ADR-0002: Container-only Development and Runtime

- **Status:** Accepted
- **Date:** 2026-10-06
- **Priority:** P0 / non-negotiable

## Context

The project must be reproducible without contaminating or provisioning the host with language toolchains and application dependencies.

Allowing contributors or coding agents to install Rust/Cargo, native libraries, or development tools directly on the host would create machine-specific state, weaken reproducibility, and make deployment behavior diverge from development behavior.

## Decision

All project build and execution activity SHALL occur inside containers.

The host SHALL NOT be used to install or execute project toolchains or dependencies.

This includes:

- Rust / rustup / Cargo;
- compiler and linker dependencies;
- system libraries needed by the application;
- linters and formatters;
- tests and conformance suites;
- the MCP server itself.

The repository SHALL provide the container definitions and commands needed for development, testing, and runtime.

## Consequences

### Positive

- reproducible developer environments;
- no host dependency drift;
- development and production paths stay aligned;
- agents cannot silently alter the host;
- simpler onboarding and cleanup.

### Negative

- container rebuilds may add some iteration overhead;
- host debugging shortcuts are intentionally unavailable;
- tools requiring native access must be deliberately passed through or exposed to the container.

## Agent rule

If an implementation step appears to require host installation, the correct response is to change the Dockerfile/container workflow.

Installing the dependency on the host is not an acceptable fallback.
