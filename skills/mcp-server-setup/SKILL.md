---
name: mcp-server-setup
description: >
  Guides developers through scaffolding and configuring a new MCP server from the ITZ
  NestJS MCP server template. Covers forking or cloning the template, configuring the
  .env file (TECHZONE_JWT_SECRET, REDIS_URL, PORT, NODE_ENV, ALLOWED_ORIGINS, OTEL_*
  variables), running the first build (npm install, npm run build, npm run test),
  renaming the server in app.module.ts, removing the placeholder tool, and completing
  the first commit checklist. Triggers on: "create new MCP server", "scaffold MCP
  server", "setup MCP server from template", "initialize new MCP server",
  "new ITZ MCP server", "get started with this template", "what do I do first",
  "fork the template", "clone the template", "configure the template".
---

# MCP Server Setup

Guides a developer from a fresh clone of the ITZ NestJS MCP server template to a
running, correctly configured server ready for the first tool to be added.

This skill is for initial setup only. Once the server is running and you are ready to
add tools, use the `mcp-builder` skill. For governance steps (APM registration, Path to
Production), use the `mcp-governance` skill.

---

## Step 1 — Clone or Fork the Template

Two paths:

**GitHub Template (recommended for new ITZ servers):**
```
1. Open https://github.com/JosephSearle/template-nestjs-mcp-server
2. Click "Use this template" → "Create a new repository"
3. Name the new repo after your Enterprise Application domain, e.g. snow-mcp-server
4. Clone your new repo locally
```

**Direct clone (for evaluation or non-ITZ use):**
```bash
git clone https://github.com/JosephSearle/template-nestjs-mcp-server <your-server-name>
cd <your-server-name>
git remote set-url origin <your-new-repo-url>
```

---

## Step 2 — Install Dependencies

```bash
npm install
```

Node.js 20+ is required. If you use nvm:
```bash
nvm use
npm install
```

---

## Step 3 — Configure the Environment

Copy `.env.template` to `.env`:
```bash
cp .env.template .env
```

Edit `.env` and fill in the following values:

| Variable | Required | Description |
|---|---|---|
| `TECHZONE_JWT_SECRET` | **YES** | HS256 shared secret for JWT validation. Must match the gateway's signing key. Use a minimum 32-character random string in development. |
| `PORT` | Recommended | HTTP port. Default is `3000`. |
| `NODE_ENV` | Recommended | Set to `development` locally, `production` in deployed environments. |
| `REDIS_URL` | Optional (local) | Redis connection URL. Omit to use the in-memory store locally. Required in production for rate limiting persistence. |
| `ALLOWED_ORIGINS` | Optional | Comma-separated list of allowed CORS origins. |
| `OTEL_EXPORTER_OTLP_TRACES_ENDPOINT` | Production only | Full OTLP HTTP endpoint for trace export (e.g. `https://<instana>/v1/traces`). Use the traces-specific URL — see G005 in `mcp-governance` for the critical double-pathing warning. |
| `OTEL_SERVICE_NAME` | Production only | Service name in Instana. Use the format `<domain>-mcp-server`, e.g. `snow-mcp-server`. |
| `OTEL_EXPORTER_OTLP_TRACES_HEADERS` | Production only | Auth headers, e.g. `Authorization=Bearer <eal-client-secret>`. Store in Kubernetes Secrets in production — never commit. |

**Security rules:**
- Never commit `.env` to git (it is already in `.gitignore`).
- Never use a weak or shared `TECHZONE_JWT_SECRET` in production.
- OTEL credentials must be stored in Kubernetes Secrets, never in source code.

---

## Step 4 — Rename the Server

Open `src/app.module.ts` and update the `McpModule.forRoot` configuration:

```typescript
McpModule.forRoot({
  name: 'your-server-name',   // ← change to kebab-case, e.g. 'snow-mcp-server'
  version: '1.0.0',
  // leave all security configuration as-is
})
```

The name appears in the MCP `initialize` handshake and in the Enterprise MCP Gateway's
tool namespace. Use kebab-case matching your repo name.

---

## Step 5 — Remove the Placeholder Tool

The template ships with placeholder files that must be removed before deployment:

```bash
rm src/tools/placeholder.tool.ts
# Remove these too if present:
rm -f src/resources/placeholder.resource.ts
rm -f src/prompts/placeholder.prompt.ts
```

Then open `src/app.module.ts` and remove any placeholder imports and provider entries.

**Do NOT remove any of the following** — these are the security infrastructure:
- `TechzoneAuthGuard` — JWT validation (Layer 2)
- `ThrottlerBehindProxyGuard` — rate limiting (Layer 3)
- `AbilityService` — CASL authorization (Layer 5)
- `McpExceptionFilter` — error channel separation (Layer 6)
- `HealthService` / `HealthController` — Kubernetes probes

---

## Step 6 — Run the First Build and Tests

```bash
npm run build    # TypeScript compile + lint + type-check
npm run test     # Jest unit tests
```

Both must pass before adding tools. Start the server:
```bash
npm run start:dev
```

Verify the health endpoint:
```bash
curl http://localhost:3000/healthz
# Expected: {"status":"ok"}
```

Verify unauthenticated requests are rejected:
```bash
curl -X POST http://localhost:3000/mcp \
  -H "Content-Type: application/json" \
  -d '{"jsonrpc":"2.0","method":"tools/list","id":1}'
# Expected: 401 Unauthorized
```

---

## Step 7 — First Commit Checklist

Before your first commit, verify:

- [ ] `.env` is NOT staged (`git status` must not show `.env`)
- [ ] Placeholder tool removed from `src/tools/` and from `src/app.module.ts`
- [ ] Server name updated in `src/app.module.ts`
- [ ] `npm run build` passes
- [ ] `npm run test` passes
- [ ] `README.md` updated to describe your server (title, purpose, Getting Started)
- [ ] `CODEOWNERS` updated to reflect the owning GitHub team (see `mcp-governance` G003)

```bash
git add src/app.module.ts src/tools/ README.md CODEOWNERS
git commit -m "feat: scaffold <your-server-name> from ITZ NestJS MCP server template"
```

---

## What to Do Next

Once the server is running and tests pass:

1. **Add your first tool** — use the `mcp-builder` skill: *"add a tool called..."*
2. **Complete governance onboarding** — use the `mcp-governance` skill: *"ITZ governance checklist"*
3. **Generate architecture docs** — use the `mcp-architecture-docs` skill
4. **Generate security docs** — use the `mcp-security-docs` skill
5. **Add architecture tests** — use the `archunitts-testing` skill
