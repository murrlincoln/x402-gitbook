# Community Discovery Layer: x402 Service Discovery API

> **Beyond the Bazaar — the community-built index of x402-payable services, with quality signals the official ecosystem page doesn't provide.**

The **x402 Service Discovery API** is an open-source, community-maintained service registry for x402-enabled endpoints. While the [Bazaar](bazaar-discovery-layer.md) lists CDP-facilitator services, the Discovery API indexes **any** x402-compatible service — on any facilitator, any chain, with enriched quality data.

**Live at:** `https://x402-discovery-api.onrender.com`

---

## Why This Exists

The Bazaar answers "what services use x402?" The Discovery API answers "**which service should my agent actually pay?**"

The difference matters when you have 251+ options:

| Signal | Bazaar | Discovery API |
|---|---|---|
| Service listing | ✅ | ✅ |
| Health check / uptime | ❌ | ✅ |
| Latency scoring | ❌ | ✅ |
| Facilitator compatibility | ❌ | ✅ |
| ERC-8004 trust signals | ❌ | ✅ |
| Category + capability tagging | ❌ | ✅ |
| MCP tool integration | ❌ | ✅ |
| Real-time auto-scanning | ❌ | ✅ (6hr refresh) |

---

## Quick Start (5 minutes)

### Option 1: Use the MCP Server (Recommended for Claude/Cursor/Windsurf)

```json
{
  "mcpServers": {
    "x402-discovery": {
      "command": "docker",
      "args": ["run", "-i", "--rm", "ghcr.io/rplryan/x402-discovery-mcp:latest"]
    }
  }
}
```

Then in your AI assistant:
> "Find me x402 services for web scraping under $0.01 per call"

### Option 2: HTTP API

```bash
# Find services by capability
curl "https://x402-discovery-api.onrender.com/discover?query=data+analysis&max_price_usd=0.05"

# Get full catalog
curl "https://x402-discovery-api.onrender.com/.well-known/x402-discovery"

# Check facilitator compatibility
curl "https://x402-discovery-api.onrender.com/facilitator-check?service_id=your_service"
```

### Option 3: Python (via x402-payment-harness)

```python
from x402_harness import X402Client

client = X402Client(private_key="your_eoa_key")
# Discover + pay in one flow
result = client.discover_and_pay(query="web scraping", max_price_usd=0.01)
```

---

## MCP Tools Available

The Discovery MCP server exposes 6 tools directly usable by Claude, Cursor, Windsurf, and any MCP-compatible AI assistant:

| Tool | Description |
|---|---|
| `x402_discover` | Find services by capability, price, category |
| `x402_health_check` | Real-time health status for any registered service |
| `x402_list_categories` | Browse all service categories |
| `x402_get_service` | Full details on a specific service |
| `x402_facilitator_check` | Verify facilitator compatibility before paying |
| `x402_register_service` | Add a new service to the catalog |

---

## Catalog Statistics

- **251+ services** indexed (auto-scanned from x402.org/ecosystem + awesome-x402)
- **6-hour refresh** cycle (auto-discovers new services)
- **Multi-chain**: Base (primary), Ethereum, Solana
- **Categories**: data, compute, agent, utility, AI/ML, DeFi, storage

---

## Registering Your Service

Any x402-compatible service can be listed:

```bash
curl -X POST https://x402-discovery-api.onrender.com/register \
  -H "Content-Type: application/json" \
  -d '{
    "name": "My x402 API",
    "url": "https://api.example.com/v1/data",
    "category": "data",
    "price_usd": 0.005,
    "network": "base-mainnet",
    "description": "Real-time on-chain data feed"
  }'
```

---

## Related Tools

- **[x402-payment-harness](https://github.com/rplryan/x402-payment-harness)** — EOA-based Python CLI for testing x402 payments without CDP. `pip install x402-payment-harness`
- **[x402-routenet](https://x402-routenet.onrender.com)** — Smart routing across multiple x402 providers with automatic failover
- **[MCP Registry](https://registry.modelcontextprotocol.io/servers/io.github.rplryan/x402-discovery-mcp)** — GitHub MCP Registry listing
- **[Glama](https://glama.ai/mcp/servers/@rplryan/x402-discovery-mcp)** — Glama MCP listing (pending review)

---

## Source Code

- Discovery API: https://github.com/rplryan/x402-discovery-mcp (API + MCP server)
- Payment Harness: https://github.com/rplryan/x402-payment-harness
- RouteNet: https://github.com/rplryan/x402-routenet
