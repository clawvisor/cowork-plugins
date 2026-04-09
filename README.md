# Clawvisor Plugins for Claude Code

Official [Claude Code plugin marketplace](https://code.claude.com/docs/en/plugin-marketplaces) for [Clawvisor](https://clawvisor.com) — a credential-vaulting gateway for AI agents.

## Available Plugins

| Plugin | Connection | Description |
|--------|------------|-------------|
| **clawvisor** | Cloud | Connects to `app.clawvisor.com`. OAuth on first use. |
| **clawvisor-local** | Direct | Connects directly to your local daemon at `localhost:25297`. No relay involved. |

Both plugins provide the same MCP tools and authorization model. Choose based on your setup:

- **clawvisor** — recommended for most users. Works out of the box after daemon install.
- **clawvisor-local** — for development, air-gapped environments, or when you prefer direct local connections.

## Quick Start

```bash
# Add the marketplace
claude plugin marketplace add clawvisor/cowork-plugins

# Install one of the plugins (from within Claude Code)
/plugin install clawvisor@cowork-plugins
# or
/plugin install clawvisor-local@cowork-plugins
```

## Prerequisites

Install the Clawvisor daemon:

```bash
curl -fsSL https://clawvisor.com/install.sh | sh
```

## Learn More

- [Clawvisor documentation](https://clawvisor.com/docs)
- [Claude Code plugins](https://code.claude.com/docs/en/plugins)
