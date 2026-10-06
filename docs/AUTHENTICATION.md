# Authentication and Identity Delegation

## 1. Goal

Each authenticated MCP user must access Synology Drive as the corresponding Synology identity so existing NAS/Drive permissions remain authoritative.

The desired property is:

```text
ChatGPT user A -> Synology user A -> ACL A
ChatGPT user B -> Synology user B -> ACL B
```

not:

```text
ChatGPT users A/B/C -> one DSM service account
```

## 2. Separate authentication from downstream delegation

The system has two independent credentials:

1. **MCP credential** — proves who may call this MCP server.
2. **Synology credential** — proves which Synology user the current request acts as.

The server must validate the MCP credential, derive a stable principal, then resolve that principal to a downstream Synology credential.

Do not assume the MCP bearer token is valid for Synology. Token forwarding is only valid when issuer, audience, scopes, and Synology support explicitly match.

## 3. Preferred production model

Preferred flow:

```text
User
 |
 | authenticate/connect
 v
MCP authorization layer
 |
 | stable principal (subject)
 v
Identity binding store
 |
 | principal -> Synology account
 v
Credential store / token broker
 |
 | user-specific delegated token
 v
Synology Drive API
```

The MCP worker itself remains stateless.

## 4. Synology OAuth path

The first PoC must answer:

> Can the Synology Drive/Office API used by this project directly accept a user-specific access token issued through Synology DSM OAuth Service?

If **yes**:

```text
principal -> encrypted Synology refresh/access token -> Drive API
```

This is the cleanest design.

If **no**:

```text
principal
   |
   v
credential broker
   |
   | user-specific DSM/Drive session credential
   v
Drive API
```

The broker may maintain renewable downstream credentials in external storage. That does not make the MCP transport sessionful.

## 5. Identity binding

A binding record should minimally contain:

```text
mcp_issuer
mcp_subject
synology_user_id
credential_reference
created_at
updated_at
revoked_at
```

Use the authorization server's stable subject identifier, not display name or email alone, as the primary identity key.

## 6. Credential storage requirements

Production requirements:

- encrypt credentials at rest;
- keep encryption keys outside the application database;
- support rotation/revocation;
- never commit credentials to Git;
- never expose credentials to the model;
- never include secrets in MCP tool output;
- never log access/refresh tokens;
- prefer references to a secret manager over raw tokens in application rows.

## 7. Fixed service account mode

A fixed Synology account may be useful for local development or an administrator-only PoC.

It must be explicitly treated as **single-identity mode**.

Example:

```text
SYNO_AUTH_MODE=fixed
SYNO_USERNAME=...
SYNO_PASSWORD=...
```

This mode must not be presented as multi-user ACL preservation.

Production multi-user deployment should use:

```text
SYNO_AUTH_MODE=delegated
```

## 8. Authorization failures

The server must preserve the security meaning of downstream errors.

Examples:

- Synology says file not visible -> do not reveal metadata from another cache.
- Synology says forbidden -> return a sanitized authorization failure.
- credential revoked -> require reconnect/re-authorization.
- identity binding missing -> return "Synology account not connected", not a service-account fallback.

There must be **no automatic privilege fallback**.

## 9. Logout and revocation

Disconnecting a Synology account should:

1. revoke the downstream token/session when supported;
2. delete or tombstone the credential reference;
3. invalidate related user-scoped caches;
4. retain only the minimum audit metadata required by policy.

## 10. Open questions for PoC

- Does Synology Drive/Office API accept DSM OAuth bearer tokens directly?
- Which OAuth scopes are required for read-only Drive access?
- Does Synology expose a stable user identifier suitable for binding?
- What refresh/revocation behavior is supported?
- Is Drive search permission-filtered by the same native ACL semantics as direct Drive access?
- Which API endpoints are required to reproduce list/search/info/download safely?
