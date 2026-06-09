# MCP-DS v0.1 — Protocol Specification

> Status: **Draft** · Editors: Stefan · Last updated: 2026-06-09

## 1. Introduction

MCP-DS is a federated discovery protocol for MCP servers. It lets agent runtimes find MCP servers by **capability** at runtime, with **verifiable trust** — without hard-coding endpoints in config files.

### 1.1 Problem Statement

The MCP ecosystem has 14,000+ servers across dozens of directories, but:

- There is no standard programmatic query API
- There is no way to discover servers *by what they do*
- There is no cross-registry federation mechanism
- There is no trust verification layer (41% of servers have zero auth)
- Every runtime vendor builds their own discovery UX

### 1.2 Design Goals

- **Capability-driven**: Query by what you need, not by name
- **Federated**: No single source of truth; registries sync and compete
- **Verifiable**: Every server answer carries a trust chain
- **Runtime-native**: Agents discover tools dynamically, not at deploy time
- **Minimal**: Thin protocol layer on top of existing MCP infrastructure

### 1.3 Non-Goals

- MCP-DS does not host server code — it's a metadata directory
- MCP-DS does not define tool schemas — that's MCP's job
- MCP-DS does not handle authentication to servers — that's MCP transport's job

## 2. Terminology

| Term | Definition |
|---|---|
| **Registry** | A service implementing the MCP-DS API |
| **Server** | An MCP server (tool provider) |
| **Server Card** | `.well-known` metadata document served by an MCP server |
| **Publisher** | Entity that publishes an MCP server to a registry |
| **Namespace** | Reverse-DNS prefix that ties servers to verified identities |
| **Capability** | A `{domain, action}` pair describing what a tool does |
| **Trust Score** | A 0.0–1.0 value derived from attestation chain |
| **Attestation** | A signed claim about a server (identity, security audit, etc.) |

## 3. Registry API

### 3.1 Base URL

All registry URLs are prefixed with a configurable base:

```
https://registry.example.com/v1
```

### 3.2 Capability Query

Primary endpoint. Resolves an agent's capability need to MCP server endpoints.

```
GET /v1/query
  ?domain=storage
  &action=read_file
  &auth=oauth2
  &version_min=1.2
  &trust_min=0.7
  &limit=10
```

**Response:**

```json
{
  "query": {
    "domain": "storage",
    "action": "read_file",
    "auth": "oauth2",
    "version_min": "1.2",
    "trust_min": 0.7
  },
  "results": [
    {
      "server": "io.github.mcp/filesystem",
      "uri": "mcp://filesystem.example.com",
      "version": "1.3.0",
      "trust": 0.92,
      "auth": ["oauth2"],
      "capabilities": ["read_file", "write_file", "list_files"]
    },
    {
      "server": "com.internal/secure-fs",
      "uri": "mcp://secure-fs.internal.corp:3000",
      "version": "2.1.0",
      "trust": 0.85,
      "auth": ["oauth2", "mtls"],
      "capabilities": ["read_file"]
    }
  ],
  "cursor": "abc123"
}
```

Results are ordered by **trust score descending**, then by **version recency**.

### 3.3 Server Details

```
GET /v1/servers/io.github.mcp/filesystem
```

**Response:**

```json
{
  "namespace": "io.github.mcp",
  "name": "filesystem",
  "display_name": "Filesystem MCP Server",
  "description": "Read, write, and list files on the local filesystem",
  "publisher": {
    "id": "io.github.mcp",
    "name": "Model Context Protocol",
    "verified": true,
    "verification": {
      "method": "github_oauth",
      "account": "modelcontextprotocol"
    }
  },
  "versions_url": "/v1/servers/io.github.mcp/filesystem/versions",
  "trust_url": "/v1/servers/io.github.mcp/filesystem/trust",
  "homepage": "https://github.com/modelcontextprotocol/servers"
}
```

### 3.4 Server Versions

```
GET /v1/servers/io.github.mcp/filesystem/versions
```

```json
{
  "server": "io.github.mcp/filesystem",
  "versions": [
    {
      "version": "1.3.0",
      "published_at": "2026-05-15T10:00:00Z",
      "checksum": "sha256:abc123...",
      "deprecated": false,
      "packages": [
        { "type": "npm", "url": "https://www.npmjs.com/package/@mcp/filesystem", "name": "@mcp/filesystem" },
        { "type": "docker", "url": "https://ghcr.io/mcp/filesystem", "name": "ghcr.io/mcp/filesystem:1.3.0" }
      ]
    }
  ]
}
```

### 3.5 Trust Chain

```
GET /v1/servers/io.github.mcp/filesystem/trust
```

```json
{
  "server": "io.github.mcp/filesystem",
  "trust_score": 0.92,
  "attestations": [
    {
      "type": "namespace_verification",
      "issuer": "registry.modelcontextprotocol.io",
      "subject": "io.github.mcp",
      "method": "github_oauth",
      "signed_at": "2026-01-10T00:00:00Z",
      "expires_at": "2027-01-10T00:00:00Z"
    },
    {
      "type": "security_audit",
      "issuer": "jfrog.io/xray",
      "subject": "io.github.mcp/filesystem@1.3.0",
      "result": "pass",
      "vulnerabilities": { "critical": 0, "high": 0, "medium": 2, "low": 5 },
      "signed_at": "2026-05-16T00:00:00Z"
    },
    {
      "type": "publisher_signed",
      "issuer": "io.github.mcp",
      "subject": "io.github.mcp/filesystem@1.3.0",
      "checksum": "sha256:abc123...",
      "signed_at": "2026-05-15T10:00:00Z"
    }
  ]
}
```

### 3.6 Pagination

All list endpoints use cursor-based pagination:

```json
{
  "data": [...],
  "next_cursor": "eyJvZmZzZXQiOjIwfQ=="
}
```

Pass `?cursor=eyJvZmZzZXQiOjIwfQ==` for the next page. Absent `next_cursor` means end of results.

## 4. Server Card

Every MCP server SHOULD serve a `.well-known/mcp-server-card` document for passive discovery:

```
GET https://filesystem.example.com/.well-known/mcp-server-card
```

```json
{
  "mcp_ds_version": "0.1",
  "server": "io.github.mcp/filesystem",
  "version": "1.3.0",
  "description": "Read, write, and list files on the local filesystem",
  "capabilities": [
    { "domain": "storage", "action": "read_file" },
    { "domain": "storage", "action": "write_file" },
    { "domain": "storage", "action": "list_files" }
  ],
  "auth": ["oauth2", "none"],
  "transport": ["sse", "streamable-http"],
  "registries": [
    "https://registry.modelcontextprotocol.io",
    "https://smithery.ai/api/mcp-ds"
  ],
  "checksum": "sha256:abc123..."
}
```

Clients use the Server Card to verify that a server's self-description matches what the registry says about it.

## 5. Namespace Model

Namespaces follow reverse-DNS convention:

```
io.github.{username}.{repo}          — GitHub account
com.{domain}.{path}                  — Verified domain
org.{organization}.{project}         — Organization
```

### 5.1 Verification Methods

| Method | How It Works | For |
|---|---|---|
| **GitHub OAuth** | Publisher authenticates via GitHub OAuth, claims `io.github.{username}` | Individual devs |
| **GitHub OIDC** | CI/CD pipeline publishes using OIDC token | Automated publishing |
| **DNS Challenge** | Publisher proves domain ownership by serving a TXT record | Enterprise / custom domains |
| **HTTP Challenge** | Publisher serves a verification file at `/.well-known/mcp-ds-verify` | Quick domain verification |

## 6. Federated Sync

Registries can mirror or index each other.

### 6.1 Zone Transfer

```
GET /v1/zone-transfer?since=2026-06-01T00:00:00Z
```

Returns all servers added, updated, or removed since the given timestamp. Enables incremental synchronization.

### 6.2 Remote Query

Registries can forward queries to peer registries and aggregate results:

```
GET /v1/query?domain=storage&action=read_file&include_peers=true
```

The responding registry queries its known peers and merges results, deduplicating by server identity.

## 7. Trust Model

### 7.1 Trust Score Calculation

```
trust_score = W_ns * S_ns + W_sec * S_sec + W_pop * S_pop + W_age * S_age
```

Where:

- `S_ns` = namespace verification score (0 or 1)
- `S_sec` = security audit score (0–1)
- `S_pop` = popularity rank (0–1, normalized by downloads)
- `S_age` = uptime / reliability score (0–1)

Default weights: `W_ns = 0.4, W_sec = 0.3, W_pop = 0.15, W_age = 0.15`

Clients MAY override weights. Registries MUST publish their default weights with every response.

### 7.2 Attestation Format

All attestations are JSON Web Signatures (JWS):

```json
{
  "payload": {
    "type": "security_audit",
    "subject": "io.github.mcp/filesystem@1.3.0",
    "result": "pass",
    "ts": "2026-05-16T00:00:00Z"
  },
  "protected": {
    "alg": "EdDSA",
    "kid": "jfrog-xray-2026-01"
  },
  "signature": "..."
}
```

## 8. Transport

MCP-DS registries MUST support HTTPS. They MAY additionally support:

- **HTTPS** (primary)
- **gRPC** (for high-throughput agent environments)
- **IPFS / DHT** (for fully decentralized discovery)

## 9. Security Considerations

- Registries must rate-limit query endpoints
- Trust scores are advisory — the client runtime makes the final decision
- Server Cards can be self-published; registries validate them before indexing
- Attestation signatures MUST use expiring keys with rotation
- Capability queries leak intent — registries should not log query details

## 10. Appendix: Example Queries

```
# Find database tools
GET /v1/query?domain=database&action=query&trust_min=0.8

# Find notification tools near me
GET /v1/query?domain=notifications&action=send&auth=oauth2&nearby=true

# Resolve exact capability URI
GET /v1/resolve?capability=mcp://storage/read_file%40v1

# Discover servers that don't require auth (for dev)
GET /v1/query?domain=storage&action=read_file&auth=none&trust_min=0
```
