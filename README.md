# MCPCraft

A Claude Code plugin for scaffolding, developing, testing, and governing NestJS MCP servers from the ITZ hardened template.

## Installation

```bash
# Install from this directory
claude plugin install ./mcpcraft

# Or test locally without installing
claude --plugin-dir ./mcpcraft
```

## Skills

Invoke skills with the `/mcpcraft:<skill-name>` prefix:

| Skill | Trigger | What it does |
|-------|---------|-------------|
| `/mcpcraft:mcp-server-setup` | "create new MCP server", "scaffold MCP server" | Bootstrap a new server from the template: clone, configure `.env`, rename, first build |
| `/mcpcraft:mcp-builder` | "add a tool", "audit this tool", "add a resource" | Add or audit tools/resources/prompts against all 7 security layers and CIO naming rules |
| `/mcpcraft:mcp-migration` | "migrate this MCP server", "upgrade to the latest template" | Migrate an out-of-date template fork or standalone server to the latest template |
| `/mcpcraft:mcp-governance` | "ITZ governance", "CIO compliance", "gateway onboarding" | Walk through the 9 CIO governance gates (APM, Path to Production, CODEOWNERS, domain scoping, OTEL, tests, security docs, arch docs, arch tests) |
| `/mcpcraft:mcp-security-docs` | "add security docs", "generate threat model", "audit mcp security" | Generate or audit SECURITY.md, threat model, security-insights.yml, and CVD process |
| `/mcpcraft:mcp-architecture-docs` | "document my MCP server", "generate C4 diagrams", "create ADRs" | Generate or audit `docs/architecture/` with MCP-aware C4 diagrams and ADRs |
| `/mcpcraft:archunitts-testing` | "add architecture tests", "audit architecture tests", "gap analyse" | Implement or gap-analyse ArchUnitTS fitness tests for layer boundaries, cycle-freedom, and metrics |

## Agent

The `mcp-dev` agent orchestrates multi-skill workflows. Use it for tasks that span several skills:

```
@mcp-dev audit my entire MCP server
@mcp-dev set up and govern a new server
@mcp-dev migrate this server to the latest template
@mcp-dev check my server is production ready
```

| Trigger | Workflow |
|---------|---------|
| "audit my entire MCP server" | mcp-builder AUDIT → mcp-governance → mcp-security-docs AUDIT → mcp-architecture-docs AUDIT → archunitts-testing GAP_ANALYSIS → unified findings report |
| "set up and govern a new server" | mcp-server-setup → mcp-governance |
| "migrate this server" | mcp-migration (8-phase) → mcp-builder AUDIT → mcp-governance |
| "add a tool and confirm it meets all standards" | mcp-builder ADD → mcp-builder AUDIT → governance pre-flight |

## Template

This plugin is built around the [ITZ NestJS MCP Server Template](../itz-repos/AskTZ/mcp-servers/template-nestjs-mcp-server), which provides:

- HS256 JWT authentication via `TechzoneAuthGuard`
- Per-user Redis-backed rate limiting
- Two-layer CASL authorization
- Zod input validation on all tool parameters
- `ToolBusinessError` for LLM-visible error messages
- CloudEvents structured logging
- OpenTelemetry tracing
- ArchUnitTS architecture fitness tests
- MCP 2025-03-26 protocol compliance (streamable HTTP transport)
