---
name: mcp-governance
description: >
  Guides developers through meeting ITZ CIO governance requirements for MCP servers.
  Covers APM registration, CIO Path to Production approval, CODEOWNERS setup, domain
  scoping, OpenTelemetry exporter configuration for the ITZ estate, test coverage
  requirements, security documentation, architecture documentation, and architecture
  testing. Triggers on: "register with APM", "ITZ governance", "CIO compliance",
  "Path to Production", "CODEOWNERS", "domain scoping", "OpenTelemetry configuration
  for ITZ", "APM registration", "governance checklist", "onboard to the MCP gateway",
  "gateway onboarding", "submit for gateway", "CIO approval", "enterprise MCP gateway",
  "register my MCP server".
---

# MCP Governance

Guides developers through the CIO governance requirements that must be met before an
MCP server can be onboarded to the IBM Enterprise MCP Gateway and consumed by agents.

These are organisational gates, not code changes. A server that is technically correct
but fails governance checks will be blocked from the gateway.

---

## The Governance Checklist

Work through this checklist before submitting for gateway onboarding:

- [ ] **G001 — APM Registration** — Server registered in APM as a Technical Service
- [ ] **G002 — CIO Path to Production Approval** — Approval obtained and reference recorded
- [ ] **G003 — CODEOWNERS** — File updated to the correct owning Enterprise Application team
- [ ] **G004 — Domain Scoping** — Tools confirmed within a single declared domain
- [ ] **G005 — OpenTelemetry** — Traces exporting to IBM AI Observability (Instana)
- [ ] **G006 — Test Coverage** — Unit tests and deployment verification tests passing
- [ ] **G007 — Security Docs** — SECURITY.md, threat-model.md, and security-insights.yml present
- [ ] **G008 — Architecture Docs** — `docs/architecture/` complete
- [ ] **G009 — Architecture Tests** — ArchUnitTS tests passing

---

## G001 — APM Registration

Every MCP server in the ITZ estate must be registered in the CIO's **Application Portfolio
Management (APM)** system as a **Technical Service**, associated with the Enterprise
Application whose domain the server exposes.

**How to register:**

1. Go to the ITZ APM portal (internal — access via Cirrus Portal).
2. Select the Enterprise Application this server belongs to.
3. Add a new Technical Service with:
   - **Name:** The server's MCP name (e.g. `snow-mcp-server`)
   - **Type:** Technical Service
   - **Description:** What the server exposes and to whom
   - **Owner:** The Enterprise Application team
4. Record the assigned APM Service ID in `docs/governance-onboarding.md`:
   ```
   APM_SERVICE_ID: <your-service-id>
   ```

Registration is done through **"Onboard AI"** — the AI agent for the CIO's Enterprise AI
Platform. If you cannot access the portal, contact your CIO representative.

---

## G002 — CIO Path to Production Approval

Approval is an extension of the CIO's standard Path to Production process. It confirms:

- The Enterprise Application owner is aware of and approves the MCP server
- The server has a distinct scope (no duplicate capabilities with existing servers)
- IAM and transport security requirements (mTLS via ContextForge gateway) are met
- The MCP Server Responsibilities checklist is fulfilled

**Steps:**

1. Submit via the CIO's Path to Production portal (internal process).
2. Record the approval reference in `docs/governance-onboarding.md`:
   ```
   PATH_TO_PRODUCTION_REF: <fill in after approval>
   ```
3. Do not submit for gateway onboarding until this reference is populated.

A server without Path to Production approval **cannot be onboarded to the gateway**.

---

## G003 — CODEOWNERS

The `CODEOWNERS` file records which GitHub team owns this MCP server. Gateway onboarding
requires ownership attributed to the Enterprise Application team.

**File location:** `CODEOWNERS` at the repo root.

**Template default:**
```
* @ITZ/core-development-team
```

**What to do:** Replace `@ITZ/core-development-team` with the GitHub team that owns the
Enterprise Application. For example:
```
* @my-org/snow-platform-team
```

The team must be a GitHub team (not an individual) with at least write access to the repo.

---

## G004 — Domain Scoping

The CIO requires each MCP server to cover a **single declared domain**. A server that
exposes tools from multiple unrelated domains (e.g. incident management AND user
provisioning) will be rejected during gateway onboarding.

**Rules:**

1. The domain is declared in `src/app.module.ts` as the server name:
   `name: 'snow-mcp-server'` signals the ServiceNow domain.
2. Every tool's name must follow `<domain>_<verb>_<noun>` — the domain prefix is the
   scope declaration. If a tool doesn't fit the domain prefix, it belongs in a different
   server.
3. Prefer **intent-based tools** over thin CRUD wrappers:
   - ✅ `snow_incident_resolve` — a complete action with a single intent
   - ❌ `snow_incident_update(status='resolved')` — raw CRUD exposing backend internals

To check for cross-domain violations, run the `mcp-builder` AUDIT mode and look for
G005 findings.

---

## G005 — OpenTelemetry Configuration for ITZ

The CIO requires all MCP servers to export traces to **IBM AI Observability** (visible in
Instana at the IBM internal dashboard).

**This template's tracing is already wired** in `src/tracing.ts` — you only need to
supply the correct environment variables in production.

**Required env vars (store in Kubernetes Secrets — never in committed `.env`):**

```bash
# Full OTLP traces endpoint — include /v1/traces in the URL
OTEL_EXPORTER_OTLP_TRACES_ENDPOINT=https://<instana-otlp-host>/v1/traces

# Service name — must match the format: <domain>-mcp-server
OTEL_SERVICE_NAME=snow-mcp-server

# Auth headers for the OTLP endpoint
OTEL_EXPORTER_OTLP_TRACES_HEADERS=Authorization=Bearer <eal-client-secret>
```

**Critical:** Always use `OTEL_EXPORTER_OTLP_TRACES_ENDPOINT` (the traces-specific
variable), NOT `OTEL_EXPORTER_OTLP_ENDPOINT`. The generic endpoint variable causes the
SDK to append `/v1/traces` to whatever URL you provide, resulting in silent
double-pathing (`…/v1/traces/v1/traces`) and traces never arriving in Instana.

**EAL subscription:** The OTLP endpoint and credentials are provisioned via the EAL
(Enterprise AI Lifecycle) team through Cirrus Portal. Store the Client ID and Client
Secret in Kubernetes Secrets.

**Verification:** After deploying with OTEL configured, check Instana under your
`OTEL_SERVICE_NAME`. If no traces appear within 5 minutes of making requests, run
`mcp-builder` AUDIT and look for O-series findings.

---

## G006 — Test Coverage

**Unit tests** — all tools must have co-located spec files covering four mandatory paths:
1. Happy path (valid input, authorized user, expected output)
2. CASL denied (authorized role but resource-level permission check fails)
3. Business error (ToolBusinessError thrown and surfaced to LLM)
4. STDIO safety (no request context — ability checks guarded with `if (request && ...)`)

```bash
npm run test
npm run test:cov   # check coverage report
```

**Deployment verification tests** — required by the CIO for post-deployment smoke testing.
At minimum, the `test/` directory must include tests covering:
```
✓ GET /healthz returns 200
✓ GET /readyz returns 200
✓ POST /mcp without token returns 401
✓ POST /mcp with valid token returns tool listing
```

If these tests are missing, use the `mcp-builder` skill to generate them, or refer to
`docs/governance-onboarding.md` → Test Automation Requirements.

---

## G007 — Security Documentation

Required before gateway onboarding:
- `SECURITY.md` at the repo root (OSPS VM-02.01 — minimum L1)
- `security-insights.yml` at the repo root (OSPS SA-03.01 — L2)
- `docs/security/threat-model.md` (OSPS SA-03.02 — L2)

To generate all of these, use the `mcp-security-docs` skill:
> *"generate security docs"* or *"add SECURITY.md"*

---

## G008 — Architecture Documentation

Required before gateway onboarding (CIO SA-01.01):
- `docs/architecture/` directory with C4 diagrams and ADRs

To generate, use the `mcp-architecture-docs` skill:
> *"generate architecture docs"* or *"document my MCP server"*

---

## G009 — Architecture Tests

Required for production code quality confidence:
- ArchUnitTS tests in `test/architecture/` enforcing layer boundaries and cycle freedom

To generate, use the `archunitts-testing` skill:
> *"add architecture tests"* or *"implement ArchUnit tests"*

Run:
```bash
npm run test:arch
```

---

## Submitting for Gateway Onboarding

Once all items in the checklist above are complete:

1. All tests pass: `npm run test && npm run test:e2e && npm run test:arch`
2. `docs/governance-onboarding.md` has `PATH_TO_PRODUCTION_REF` populated
3. `CODEOWNERS` is updated to the owning team
4. APM Service ID is recorded
5. Architecture and security docs are complete and committed

Submit via the CIO's ITZ Gateway Onboarding process. Reference the APM Service ID and
Path to Production reference in the submission form.
