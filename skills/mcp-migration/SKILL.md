---
name: mcp-migration
description: >
  Migrates a NestJS MCP server to the latest ITZ hardened template from
  github.ibm.com/itz/template-nestjs-mcp-server. Handles two scenarios:
  (A) an existing server forked from this template that has drifted behind main —
  fetches and merges the latest template changes, resolving infrastructure conflicts
  in favour of the template while preserving business logic; (B) a standalone NestJS
  MCP server with no template ancestry — brings in the full security infrastructure
  (guards, auth, filters, config, errors, health, tracing) alongside existing code
  and refactors tool implementations to comply with template patterns. Triggers on:
  "migrate to the template", "upgrade to the latest template", "bring my server up
  to date with the template", "migrate this MCP server", "apply the template to my
  server", "update template", "sync with template", "merge template changes",
  "migrate standalone server", "bring in the security infrastructure".
---

# MCP Migration

Migrates any NestJS MCP server to the latest ITZ hardened template. Two migration paths
are supported based on the server's history. Always detect the scenario first — the merge
strategy differs significantly between them.

Read `references/conflict-resolution.md` now and keep it in context for Phases 5 and 6.

---

## Phase 1 — Detect Scenario

Run these checks and report findings before doing anything else:

```bash
# Ancestry and remotes
git log --all --oneline | head -20
git remote -v

# Template infrastructure presence
ls src/guards/ 2>/dev/null
ls src/auth/ability.service.ts 2>/dev/null
ls src/filters/ 2>/dev/null
ls src/errors/tool-business.error.ts 2>/dev/null

# Template marker dependencies
grep -E '"@rekog/mcp-nest"|"@itz/core-utils"' package.json
```

**Scenario A — Out-of-date template fork:** Infrastructure files are present AND
`package.json` contains both `@rekog/mcp-nest` and `@itz/core-utils`. This server
was cloned or forked from the template and has drifted behind `main`.

**Scenario B — Standalone server:** Infrastructure files are absent OR the template
marker deps are missing from `package.json`. This server was built independently
and needs the full security infrastructure brought in.

If detection is ambiguous, ask the developer which scenario applies before continuing.

---

## Phase 2 — Pre-flight Checks

Establish a known-good baseline. **Stop and report if any check fails — do not proceed
with a dirty or broken tree.**

```bash
git status          # must be clean — no uncommitted changes, no untracked tool files
npm run build       # must compile without errors
npm test            # record the pass/fail count as the pre-migration baseline
```

Create a dedicated migration branch:
```bash
git checkout -b migration/template-$(date +%Y%m%d)
```

Record and report the current state:
- Versions of `@rekog/mcp-nest` and `@itz/core-utils` in `package.json`
- All files under `src/tools/`, `src/resources/`, `src/prompts/` (user-owned business logic)
- Any custom additions to `src/app.module.ts` beyond the placeholder (extra providers,
  additional imports, custom `McpModule.forRoot` options)

---

## Phase 3 — Set Up Template Remote

```bash
# Add the canonical template remote (skip if it already exists)
git remote | grep -q "^template$" || \
  git remote add template https://github.ibm.com/itz/template-nestjs-mcp-server.git

git fetch template main
```

If the fetch fails (network error, authentication issue, or the remote URL is incorrect),
stop and surface the error. Do not attempt to continue with a stale or missing remote ref.

---

## Phase 4 — Analyze the Delta

Summarize what has changed before any files are modified.

```bash
# Commits in template not yet in local (Scenario A)
git log --oneline HEAD..template/main

# All files changed in template
git diff HEAD template/main --name-only

# Infrastructure-only diff (template-owned files)
git diff HEAD template/main -- \
  src/guards/ src/auth/ src/filters/ src/config/ \
  src/errors/ src/health/ src/main.ts src/tracing.ts \
  src/app.controller.ts src/common/ src/types/ src/utils/ \
  package.json tsconfig.json
```

For **Scenario B**, identify which template infrastructure files are entirely absent locally:
```bash
for f in src/guards/techzone-auth.guard.ts \
         src/auth/ability.service.ts \
         src/filters/mcp-exception.filter.ts \
         src/errors/tool-business.error.ts \
         src/config/env.validation.ts \
         src/tracing.ts; do
  [ -f "$f" ] || echo "MISSING: $f"
done
```

Report a summary before proceeding:
- N infrastructure files to update (template wins — no review needed)
- M shared files to merge (`app.module.ts`, `package.json` — careful merge required)
- Names of user-owned files that will need compliance refactoring after the merge

---

## Phase 5 — Execute the Merge

**Scenario A — Out-of-date fork:**

Merge infrastructure files taking the template version, leaving shared and user files
for manual resolution:

```bash
git merge template/main --no-commit --no-ff
```

For any infrastructure file that conflicts, take the template version:
```bash
# Repeat for each conflicting infrastructure file
git checkout template/main -- <conflicting-file>
```

For user-owned files (`src/tools/`, `src/resources/`, `src/prompts/`) that conflict,
keep the local version:
```bash
git checkout HEAD -- src/tools/ src/resources/ src/prompts/
```

**Scenario B — Standalone server:**

Pull in the full infrastructure in one operation:
```bash
git checkout template/main -- \
  src/guards/ src/auth/ src/filters/ src/config/ \
  src/errors/ src/health/ src/main.ts src/tracing.ts \
  src/app.controller.ts src/common/ src/types/ src/utils/
```

Do not overwrite `src/tools/`, `src/resources/`, or `src/prompts/` — these are
user-owned and will be refactored in Phase 7.

---

## Phase 6 — Resolve Shared Files

Apply the rules from `references/conflict-resolution.md` exactly. The two files
requiring the most care are:

**`src/app.module.ts`:**
- Preserve the local `name` and `version` from `McpModule.forRoot`
- Replace the entire security configuration with the template's version
- All five template security providers must be present (see reference Section 2)
- Re-add any user providers (tool classes, resource classes) after the security block
- Verify there are no remaining conflict markers (`<<<<<<<`)

**`package.json`:**
- For every dependency present in both local and template: use the template's version
- Keep all deps that are only in the local `package.json`
- Add all deps that are only in the template
- See reference Section 3 for the full list of template-pinned packages

After updating `package.json`:
```bash
npm install
```

Scan for any remaining conflict markers in the entire repo:
```bash
grep -r "^<<<<<<< " src/ package.json tsconfig.json 2>/dev/null
```

If any are found, resolve them before continuing.

---

## Phase 7 — Post-Migration Compliance

**Step 1 — Build:**
```bash
npm run build
```
Fix all TypeScript compile errors before continuing. Common causes: changed guard
constructor signatures, updated `ToolBusinessError` constructor, new required env vars
in the Zod config schema.

**Step 2 — AUDIT existing tools:**
Invoke `mcp-builder` AUDIT mode on every file in `src/tools/`, `src/resources/`,
and `src/prompts/`. Fix all CRITICAL and HIGH findings before declaring the migration
complete. Document MEDIUM and LOW findings for follow-up.

**Step 3 — Scenario B only — Refactor pre-existing tools to template patterns:**

For each tool that existed before migration, refactor it to comply. Each tool must have:
- A Zod schema object with `.describe()` on every field, passed to `@Tool({ ... })`
- `@ToolRoles([...])` decorator on the tool class (coarse role gating)
- `AbilityService.can()` check for resource-level authorization
- `ToolBusinessError` for all LLM-visible error cases
- `if (request && ...)` guard around any `AbilityService` call (STDIO transport safety)
- Registration in `AppModule.providers`
- A co-located `.spec.ts` covering 4 test paths: happy, CASL denied, business error, STDIO

Read `references/tool-patterns.md` from the `mcp-builder` skill for the exact
implementation patterns.

**Step 4 — Full test suite:**
```bash
npm test
npm run test:e2e
```

If ArchUnit tests exist:
```bash
npm run test:arch
```

All tests must pass before committing. If the pre-migration baseline had fewer
passing tests than post-migration, investigate the regression before continuing.

---

## Phase 8 — Migration Checklist and Commit

Verify every item before committing:

- [ ] `npm run build` passes with zero errors
- [ ] `npm test` passes — equal or better than pre-migration baseline
- [ ] `npm run test:e2e` passes
- [ ] `mcp-builder` AUDIT shows no CRITICAL or HIGH findings on user-owned files
- [ ] `src/app.module.ts` contains all five required security providers (see reference Section 2)
- [ ] `package.json` template-pinned dep versions match the template (see reference Section 3)
- [ ] No conflict markers (`<<<<<<<`) anywhere in the codebase
- [ ] `package-lock.json` updated (`npm install` was run after `package.json` changes)

```bash
git add -A
git commit -m "chore: migrate to ITZ NestJS MCP server template $(date +%Y-%m-%d)"
```

Open a PR: `migration/template-<date>` → `main`.

Recommend running `mcp-governance` after the PR is merged to verify the full CIO
compliance checklist is still satisfied post-migration.
