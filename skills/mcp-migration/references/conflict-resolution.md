# Conflict Resolution Reference

Used by the `mcp-migration` skill during Phases 5 and 6. Defines file ownership,
merge rules, and the canonical content required after migration.

---

## Section 1 — File Ownership Map

| File / Directory | Owner | Rule on conflict |
|---|---|---|
| `src/guards/` | Template | Always take template version — never edit |
| `src/auth/` | Template | Always take template version — never edit |
| `src/filters/` | Template | Always take template version — never edit |
| `src/config/env.validation.ts` | Template | Take template version; re-add user env vars in the user section at the bottom |
| `src/errors/tool-business.error.ts` | Template | Always take template version — never edit |
| `src/health/` | Template | Always take template version — never edit |
| `src/main.ts` | Template | Always take template version — never edit |
| `src/tracing.ts` | Template | Always take template version — never edit |
| `src/app.controller.ts` | Template | Always take template version |
| `src/app.service.ts` | Template | Always take template version |
| `src/common/` | Template | Always take template version |
| `src/types/` | Template | Always take template version |
| `src/utils/` | Template | Always take template version |
| `src/tools/` | **User** | Never overwrite — refactor to comply, keep logic |
| `src/resources/` | **User** | Never overwrite — refactor to comply, keep logic |
| `src/prompts/` | **User** | Never overwrite — refactor to comply, keep logic |
| `src/app.module.ts` | **Shared** | Careful merge — see Section 2 |
| `package.json` | **Shared** | Careful merge — see Section 3 |
| `tsconfig.json` | Template | Take template version |
| `.eslintrc.js` / `eslint.config.mjs` | Template | Take template version |
| `.prettierrc` | Template | Take template version |
| `jest.config.js` / `jest-e2e.json` / `jest-arch.json` | Template | Take template version |
| `Containerfile` | Template | Take template version |
| `tz-build.yml` | Template | Take template version |
| `CODEOWNERS` | **User** | Never overwrite |
| `README.md` | **User** | Never overwrite |
| `docs/` | **User** | Never overwrite |
| `.env.template` | Template | Take template version; user must re-add custom vars |
| `.gitignore` | Template | Take template version |

---

## Section 2 — `app.module.ts` Merge Rule

`app.module.ts` must contain the entire template security configuration block, with only
`name` and `version` preserved from the local version.

### What to preserve from local

```typescript
McpModule.forRoot({
  name: '<PRESERVE THIS FROM LOCAL>',     // ← e.g. 'snow-mcp-server'
  version: '<PRESERVE THIS FROM LOCAL>',  // ← e.g. '1.0.0'
  // everything else → replace with template
})
```

Also preserve any user-added providers (tool classes, resource classes, prompt classes)
in the `providers` array.

### Canonical imports block (copy verbatim from template)

```typescript
import { Module } from '@nestjs/common';
import { ConfigModule, ConfigService } from '@nestjs/config';
import { APP_FILTER } from '@nestjs/core';
import { McpModule, McpTransportType } from '@rekog/mcp-nest';
import { ThrottlerModule } from '@nestjs/throttler';
import { ThrottlerStorageRedisService } from '@nest-lab/throttler-storage-redis';
import { LoggerModule } from 'nestjs-pino';
import { AppController } from './app.controller';
import { AppService } from './app.service';
import { HealthController } from './health/health.controller';
import { HealthService } from './health/health.service';
import { AbilityService } from './auth/ability.service';
import { TechzoneAuthGuard } from './guards/techzone-auth.guard';
import { ThrottlerBehindProxyGuard } from './guards/throttler-proxy.guard';
import { McpExceptionFilter } from './filters/mcp-exception.filter';
import type { Env } from './config/env.validation';
import { join } from 'path';
```

### Canonical `imports` array (copy verbatim, restore `name` and `version`)

```typescript
imports: [
  ConfigModule.forRoot({ isGlobal: true }),
  ThrottlerModule.forRootAsync({
    useFactory: (config: ConfigService<Env>) => {
      const redisUrl = config.get('REDIS_URL', { infer: true });
      return {
        throttlers: [
          {
            ttl: config.get('RATE_LIMIT_TTL', { infer: true }) ?? 60000,
            limit: config.get('RATE_LIMIT_MAX', { infer: true }) ?? 60,
          },
        ],
        ...(redisUrl
          ? { storage: new ThrottlerStorageRedisService(redisUrl) }
          : {}),
      };
    },
    inject: [ConfigService],
  }),
  McpModule.forRoot({
    name: '<YOUR SERVER NAME>',       // ← restore from local
    version: '<YOUR SERVER VERSION>', // ← restore from local
    transport: McpTransportType.STREAMABLE_HTTP,
    streamableHttp: {
      enableJsonResponse: true,
      statelessMode: true,
    },
    capabilities: {
      tools: { listChanged: false },
      logging: {},
    },
    guards: [TechzoneAuthGuard, ThrottlerBehindProxyGuard],
    allowUnauthenticatedAccess: false,
  }),
  LoggerModule.forRoot({
    pinoHttp:
      process.env.NODE_ENV === 'production'
        ? {
            level: 'info',
            transport: {
              target: join(__dirname, 'utils', 'cloudevents-transport.js'),
            },
            serializers: {
              req: (req: { method: string; url: string }) => ({
                method: req.method,
                url: req.url,
              }),
              res: (res: { statusCode: number }) => ({
                statusCode: res.statusCode,
              }),
            },
          }
        : {
            level: 'debug',
            transport: {
              target: 'pino-pretty',
              options: {
                colorize: true,
                singleLine: true,
                translateTime: 'HH:MM:ss.l',
                ignore: 'pid,hostname',
              },
            },
            serializers: {
              req: (req: { method: string; url: string }) => ({
                method: req.method,
                url: req.url,
              }),
              res: (res: { statusCode: number }) => ({
                statusCode: res.statusCode,
              }),
            },
          },
  }),
],
```

### Required `controllers` array

```typescript
controllers: [AppController, HealthController],
```

### Required `providers` — security block (must be present, in this order)

```typescript
providers: [
  AppService,
  HealthService,
  AbilityService,
  TechzoneAuthGuard,
  ThrottlerBehindProxyGuard,
  { provide: APP_FILTER, useClass: McpExceptionFilter },
  // ↓ user providers go here (tools, resources, prompts)
],
```

If the server has additional user-written providers, add them after `McpExceptionFilter`.
Never reorder or remove the template security providers above.

### Hard rules for `app.module.ts`

- `TechzoneAuthGuard` must appear in BOTH `McpModule.forRoot({ guards: [...] })` and `providers`
- `ThrottlerBehindProxyGuard` must appear in BOTH `guards` and `providers`
- `AbilityService` must be in `providers` — it is injected by tool classes
- `{ provide: APP_FILTER, useClass: McpExceptionFilter }` must be last in the security block
- `allowUnauthenticatedAccess: false` must be set — never set this to `true`
- `statelessMode: true` is the template default; only change to `false` if the server
  explicitly requires stateful mode (see ADR-0009)

---

## Section 3 — `package.json` Merge Rule

For each dependency:
- **Present in both local and template:** use the template's version (do not keep a lower version)
- **Present only in local:** keep it (user-added dependency)
- **Present only in template:** add it

### Template-pinned dependencies (must be at these exact versions after migration)

**Runtime `dependencies`:**

| Package | Template version |
|---|---|
| `@rekog/mcp-nest` | `^1.9.8` |
| `@modelcontextprotocol/sdk` | `^1.24.2` |
| `@nestjs/common` | `^11.1.9` |
| `@nestjs/core` | `^11.0.1` |
| `@nestjs/platform-express` | `^11.0.1` |
| `@nestjs/config` | `4.0.4` |
| `@nestjs/throttler` | `6.5.0` |
| `@nest-lab/throttler-storage-redis` | `1.2.0` |
| `@itz/core-utils` | `1.0.70` |
| `@casl/ability` | `6.8.1` |
| `zod` | `^4.0.0` |
| `jsonwebtoken` | `9.0.3` |
| `ioredis` | `5.10.1` |
| `@opentelemetry/sdk-node` | `0.218.0` |
| `@opentelemetry/exporter-trace-otlp-http` | `0.218.0` |
| `@opentelemetry/auto-instrumentations-node` | `0.76.0` |
| `nestjs-pino` | `^4.5.0` |
| `pino-http` | `^11.0.0` |
| `helmet` | `^8.1.0` |

After updating `package.json`, run:
```bash
npm install
git add package-lock.json
```

### Hard rules for `package.json`

- Never downgrade a template-pinned dep below the version in the table above
- `zod` must be `^4.0.0` or later — `zod` v3 is incompatible with `@rekog/mcp-nest` ≥1.9
- `@nestjs/*` packages must all be on the same major version (currently 11)
- `@opentelemetry/*` packages must all be on the same minor version

---

## Section 4 — Common Conflict Patterns

### Guard constructor signature changed

The guard constructors (`TechzoneAuthGuard`, `ThrottlerBehindProxyGuard`) occasionally
change their constructor injection tokens between template versions.

**Resolution:** Always take the template version of the guard file. Do not attempt to
port the old constructor. If a user tool tests were injecting the guard directly, update
the test mocks to match the new constructor signature after the merge.

### New env var added to `env.validation.ts`

The template's Zod config schema may gain new required vars between versions.

**Resolution:** Take the template's full Zod schema. Then inspect whether the server had
any custom env vars appended to the user section of the file. If so, re-add them after
the last template-defined var. The user section is marked with a comment:
```typescript
// --- user-defined env vars below this line ---
```

### `McpModule.forRoot` options changed

The template may add or remove options in the `streamableHttp` block or `capabilities`.

**Resolution:** Take the template's full `McpModule.forRoot` block. Then restore only
`name` and `version` from the local version. Never carry forward removed options.

### New decorator added to `src/common/`

Decorators like `@ToolRoles` live in `src/common/`. If a new decorator is added, take
the template version. If an existing decorator signature changed, update any user tool
files that use it after completing the infrastructure merge.

### `ToolBusinessError` constructor changed

If `src/errors/tool-business.error.ts` has a new constructor signature, take the
template version. Then search user tools for usages and update them:
```bash
grep -r "new ToolBusinessError" src/tools/ src/resources/ src/prompts/
```

### `pino-http` / `nestjs-pino` serializer API changed

If the Pino serializer options in `app.module.ts` have changed, take the template's
`LoggerModule.forRoot` block verbatim.

### Stateful mode configuration

If the local server is running in stateful mode (`statelessMode: false`), restore this
after copying the template's `McpModule.forRoot` block:
```typescript
streamableHttp: {
  enableJsonResponse: true,
  statelessMode: false,
  sessionIdGenerator: () => randomUUID(),
},
```

Do not blindly set `statelessMode: true` on a server that was intentionally stateful.
Check ADR-0009 in `docs/architecture/adr/` if it exists.
