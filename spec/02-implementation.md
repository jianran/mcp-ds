# MCP-DS — Implementation Guide

How to build a registry, add MCP-DS support to a client, or publish a server.

## 1. Running a Registry

### Minimal Registry (Single Server)

```
docker run -p 8080:8080 mcp-ds/registry \
  --namespace jianran \
  --store sqlite:///data/mcp-ds.db
```

### Production Registry

```
docker run -p 8080:8080 mcp-ds/registry \
  --store postgres://user:pass@host/mcpds \
  --namespace io.github.jianran \
  --peers https://registry.modelcontextprotocol.io,https://smithery.ai/api/mcp-ds \
  --sync-interval 300s \
  --trust-weights 0.3,0.3,0.2,0.2
```

### Configuration

| Flag | Default | Description |
|---|---|---|
| `--port` | `8080` | HTTP port |
| `--store` | `sqlite://mcp-ds.db` | Database backend |
| `--peers` | `""` | Comma-separated peer registry URLs |
| `--sync-interval` | `300s` | Peer sync interval |
| `--trust-weights` | `0.4,0.3,0.15,0.15` | Custom trust weights |

## 2. Adding MCP-DS to an Agent Client

### Concept

The MCP client (your agent runtime) already knows how to connect to a configured MCP server over stdio/SSE. MCP-DS adds a **resolver step** before connection:

1. Agent needs a capability → query MCP-DS registry
2. Get back server URIs + trust scores
3. Select appropriate server (highest trust, lowest latency, etc.)
4. Connect via standard MCP transport
5. Cache result for TTL

### Pseudocode

```python
class MCPDSResolver:
    def __init__(self, registry_url: str, trust_min: float = 0.5):
        self.registry = registry_url
        self.trust_min = trust_min
        self.cache = TTLCache(maxsize=256, ttl=300)

    def resolve(self, domain: str, action: str, **kwargs) -> MCPConnection:
        """Resolve a capability to a connected MCP server."""
        cache_key = f"{domain}/{action}/{kwargs}"
        
        if cache_key in self.cache:
            return self._connect(self.cache[cache_key])

        results = self._query({
            "domain": domain,
            "action": action,
            "trust_min": self.trust_min,
            **kwargs
        })

        if not results:
            raise CapabilityNotFound(domain, action)

        # Pick best match (highest trust, or custom policy)
        selected = self._select(results)
        self.cache[cache_key] = selected
        
        return self._connect(selected)

    def _query(self, params: dict) -> list:
        return requests.get(f"{self.registry}/v1/query", params=params).json()

    def _connect(self, server: dict) -> MCPConnection:
        return MCPConnection(server["uri"], auth=server.get("auth"))
```

## 3. Publishing a Server

### Manual (CLI)

```bash
mcp-ds publish \
  --server io.github.jianran/my-tool \
  --uri mcp://my-tool.example.com \
  --capabilities database/query,database/schema \
  --auth oauth2
```

### Automated (CI/CD)

```yaml
# .github/workflows/publish.yml
on:
  release:
    types: [published]

jobs:
  publish-mcp-registry:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: mcp-ds publish --oidc-token ${{ secrets.ACTIONS_ID_TOKEN_REQUEST_TOKEN }} ...
```

### Server Card (Optional but Recommended)

Place a `.well-known/mcp-server-card` at your server's root URL. Registries will discover it automatically during indexing:

```
https://my-tool.example.com/.well-known/mcp-server-card
```
