# MCP-DS — MCP Discovery Service

**The DNS for agent tools.**

A federated protocol for dynamic MCP server discovery by capability, with transitive trust.

## Why

MCP (Model Context Protocol) standardized how agents *connect* to tools — but not how they *find* them. Today, every MCP server is hard-configured in a JSON file before the agent starts:

```json
{
  "mcpServers": {
    "filesystem": { "command": "npx", "args": ["@modelcontextprotocol/server-filesystem", "/tmp"] }
  }
}
```

This breaks when:

- An agent needs a tool at runtime that wasn't pre-configured
- You have 14,000 servers across 8 registries and no way to query by capability
- You need to verify a server is legitimate before connecting
- A server endpoint changes and every config file needs updating

**MCP-DS fixes this** by adding a programmatic discovery layer — agent runtimes query for tools by capability, get back verified endpoints, and connect dynamically.

## The Stack (Where This Fits)

```
L4  Application           CrewAI / LangGraph / Claude SDK
                            ↑ A2A agent handoff
L3  Communication          MCP tools + resources
                            ↑ MCP-DS ← YOU ARE HERE
L2  Runtime                LangGraph / OpenAI SDK / Smolagents
                            ↑ OpenAI API format
L1  Models                 GPT-4o / Claude / Gemini / vLLM
```

## Protocol Overview

MCP-DS defines four things:

| Component | What | Analogy |
|---|---|---|
| **Registry API** | Standard REST/gRPC API that any registry implements | DNS server |
| **Capability Query** | Resolve `{domain, action, inputs}` → `[MCP server URLs + trust]` | SRV record lookup |
| **Server Card** | `.well-known/mcp-server-card` — discoverable metadata on any MCP host | robots.txt |
| **Trust Chain** | Signed attestations: registry signs server metadata, server signs its tool list | DNSSEC |

## Key Design Decisions

### Federated, Not Centralized

Multiple registries coexist. The protocol defines:

- A standard query API (→ any registry is interchangeable)
- A sync/zone-transfer mechanism (→ registries can mirror each other)
- A namespace model based on reverse-DNS (→ `io.github.username/server-name`)

No single registry is the source of truth. Clients query the registry they trust, or multiple registries and merge results.

### Query by Capability, Not Name

The primary query mode is *what the agent needs to do*, not *what server to use*:

```
Query:  { domain: "storage", action: "read_file", constraints: { "auth": "oauth2" } }
Result: [
  { uri: "mcp://filesystem.example.com", trust: 0.92, auth: "oauth2" },
  { uri: "mcp://s3-bridge.internal", trust: 0.85, auth: "api_key" }
]
```

### Transitive Trust

Trust is layered:

1. **Registry-level**: The official registry verifies namespace ownership (GitHub OAuth, domain DNS challenge)
2. **Publisher-level**: Signed attestations link a server version to a publisher identity
3. **Audit-level**: Third-party scanners (JFrog, etc.) can sign security audit results
4. **Client-level**: The agent runtime decides what trust threshold to accept

Every MCP-DS response includes a trust score (0.0 – 1.0) derived from these layers.

### Dynamic, Not Static

Servers come and go. MCP-DS is designed for:

- **Runtime resolution**: Agent says "I need X" → runtime queries registry → connects immediately
- **Health probing**: Registry periodically checks that servers are actually reachable
- **Version pinning**: Agent can request `>=1.2, <2.0` and get the right version
- **Deprecation signaling**: Registry marks servers as deprecated with migration hints

## API Surface

```
GET  /v1/query?domain=storage&action=read_file&auth=oauth2
     → { servers: [{ uri, trust, auth, version, publisher }] }

GET  /v1/servers/{namespace}/{name}
     → { server card with full metadata }

GET  /v1/servers/{namespace}/{name}/versions
     → { versions: [{ version, published_at, checksum }] }

GET  /v1/resolve?capability=mcp://capability-id
     → [resolved server URIs with trust chains]

GET  /v1/trust/{namespace}/{name}@{version}
     → { signed attestations from registries, publishers, auditors }
```

## Roadmap

| Phase | What | Status |
|---|---|---|
| **v0.1** | Server Card standard + basic query API | Spec draft |
| **v0.2** | Federated sync protocol between registries | Planned |
| **v0.3** | Trust chain spec (attestation format + verification) | Planned |
| **v1.0** | Production spec + reference implementation (registry + client SDK) | TBD |

---

*This is a protocol spec project. Read `spec/` for the detailed proposal.*
