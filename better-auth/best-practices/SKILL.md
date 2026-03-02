---
name: better-auth-best-practices
description: Configure Better Auth server and client, set up database adapters, manage sessions, add plugins, and handle environment variables. Use when users mention Better Auth, betterauth, auth.ts, or need to set up TypeScript authentication with email/password, OAuth, or plugin configuration.
---

# Better Auth Integration Guide

**Always consult [better-auth.com/docs](https://better-auth.com/docs) for code examples and latest API.**

---

## Setup Workflow

1. Install: `npm install better-auth`
2. Set env vars: `BETTER_AUTH_SECRET` and `BETTER_AUTH_URL`
3. Create `auth.ts` with database + config
4. Create route handler for your framework
5. Run `npx auth migrate` (or `npx @better-auth/cli@latest migrate`)
6. Verify: call `GET /api/auth/ok` — should return `{ status: "ok" }`

---

## Quick Reference

### Environment Variables
- `BETTER_AUTH_SECRET` - Encryption secret (min 32 chars). Generate: `openssl rand -base64 32`
- `BETTER_AUTH_URL` - Base URL (e.g., `https://example.com`)

Only define `baseURL`/`secret` in config if env vars are NOT set.

### File Location
CLI looks for `auth.ts` in: `./`, `./lib`, `./utils`, or under `./src`. Use `--config` for custom path.

### CLI Commands

The new standalone CLI (`npx auth`) replaces the old `@better-auth/cli` package (now deprecated):

- `npx auth init` - Interactive setup wizard (config, database adapter, framework integration)
- `npx auth migrate` - Apply schema (built-in adapter)
- `npx auth generate` - Generate schema for Prisma/Drizzle
- `npx auth generate --adapter prisma` - Generate schema for a specific adapter without a config file
- `npx auth generate --adapter drizzle` - Same, for Drizzle
- `npx auth upgrade` - Upgrade Better Auth to the latest version

The old `@better-auth/cli` commands still work as aliases during the deprecation period.

**Re-run migrate/generate after adding/changing plugins.**

---

## Core Config Options

| Option | Notes |
|--------|-------|
| `appName` | Optional display name |
| `baseURL` | Only if `BETTER_AUTH_URL` not set. Supports dynamic config object: `{ allowedHosts, fallback, protocol }` for Vercel preview deployments and multi-domain setups. |
| `basePath` | Default `/api/auth`. Set `/` for root. |
| `secret` | Only if `BETTER_AUTH_SECRET` not set |
| `database` | Required for most features. See adapters docs. |
| `secondaryStorage` | Redis/KV for sessions & rate limits |
| `emailAndPassword` | `{ enabled: true }` to activate |
| `socialProviders` | `{ google: { clientId, clientSecret }, ... }` |
| `plugins` | Array of plugins |
| `trustedOrigins` | CSRF whitelist |

---

## Database

**Direct connections:** Pass `pg.Pool`, `mysql2` pool, `better-sqlite3`, `bun:sqlite`, or a Cloudflare D1 binding.

**ORM adapters:** Import from `better-auth/adapters/drizzle`, `better-auth/adapters/prisma`, `better-auth/adapters/mongodb` (re-exported from the main package), or directly from the extracted packages for smaller bundles: `@better-auth/drizzle-adapter`, `@better-auth/prisma-adapter`, `@better-auth/kysely-adapter`, `@better-auth/mongo-adapter`.

**Cloudflare D1:** Pass the D1 binding directly — auto-detected, no adapter setup required. Note: D1 does not support interactive transactions; Better Auth uses `batch()` for atomicity.

**Critical:** Better Auth uses adapter model names, NOT underlying table names. If Prisma model is `User` mapping to table `users`, use `modelName: "user"` (Prisma reference), not `"users"`.

---

## Session Management

**Storage priority:**
1. If `secondaryStorage` defined → sessions go there (not DB)
2. Set `session.storeSessionInDatabase: true` to also persist to DB
3. No database + `cookieCache` → fully stateless mode

**Cookie cache strategies:**
- `compact` (default) - Base64url + HMAC. Smallest.
- `jwt` - Standard JWT. Readable but signed.
- `jwe` - Encrypted. Maximum security.

**Key options:** `session.expiresIn` (default 7 days), `session.updateAge` (refresh interval), `session.cookieCache.maxAge`, `session.cookieCache.version` (change to invalidate all sessions).

---

## User & Account Config

**User:** `user.modelName`, `user.fields` (column mapping), `user.additionalFields`, `user.changeEmail.enabled` (disabled by default), `user.deleteUser.enabled` (disabled by default).

**Account:** `account.modelName`, `account.accountLinking.enabled`, `account.storeAccountCookie` (for stateless OAuth).

**Required for registration:** `email` and `name` fields.

---

## Email Flows

- `emailVerification.sendVerificationEmail` - Must be defined for verification to work
- `emailVerification.sendOnSignUp` / `sendOnSignIn` - Auto-send triggers
- `emailAndPassword.sendResetPassword` - Password reset email handler

---

## Security

**In `advanced`:**
- `useSecureCookies` - Force HTTPS cookies
- `disableCSRFCheck` - ⚠️ Security risk
- `disableOriginCheck` - ⚠️ Security risk  
- `crossSubDomainCookies.enabled` - Share cookies across subdomains
- `ipAddress.ipAddressHeaders` - Custom IP headers for proxies
- `ipAddress.ipv6Subnet` - Rate limit IPv6 by subnet prefix (default: 64)
- `database.generateId` - Custom ID generation or `"serial"`/`"uuid"`/`false`

**Rate limiting:** `rateLimit.enabled`, `rateLimit.window`, `rateLimit.max`, `rateLimit.storage` ("memory" | "database" | "secondary-storage"). Default sensitive-endpoint limits are 3 req/10s for sign-in/sign-up and 3 req/60s for password-reset/OTP.

**Secret key rotation** — rotate `BETTER_AUTH_SECRET` without invalidating existing data by providing a `secrets` array:

```ts
export const auth = betterAuth({
  secrets: [
    { version: 2, value: "new-secret-key" }, // first = active for new encryptions
    { version: 1, value: "old-secret-key" }, // kept for decryption
  ],
});
```

Or via environment variable: `BETTER_AUTH_SECRETS="2:new-secret,1:old-secret"`

---

## Hooks

**Endpoint hooks:** `hooks.before` / `hooks.after` — Pass a single `createAuthMiddleware` handler or an array of `{ matcher, handler }` objects. Both global and plugin hooks use the same `AuthMiddleware` type. Access `ctx.path`, `ctx.context.returned` (after), `ctx.context.session`.

**Database hooks:** `databaseHooks.user.create.before/after`, same for `session`, `account`. Useful for adding default values or post-creation actions.

**Important (1.5):** `after` database hooks (`create.after`, `update.after`, `delete.after`) now run **after the transaction commits**, not inside it. If you need atomic database writes from a hook, use the adapter directly within the main operation.

**Hook context (`ctx.context`):** `session`, `secret`, `authCookies`, `password.hash()`/`verify()`, `adapter`, `internalAdapter`, `generateId()`, `tables`, `baseURL`.

---

## Plugins

**Import from dedicated paths for tree-shaking:**
```
import { twoFactor } from "better-auth/plugins/two-factor"
```
NOT `from "better-auth/plugins"`.

**Popular plugins (bundled):** `twoFactor`, `organization`, `passkey`, `magicLink`, `emailOtp`, `username`, `phoneNumber`, `admin`, `bearer`, `jwt`, `multiSession`, `openAPI`, `genericOAuth`, `testUtils`.

**Extracted to their own packages (install separately):**
- `apiKey` → `@better-auth/api-key` (**removed from** `better-auth/plugins` in 1.5)
- OAuth 2.1 provider → `@better-auth/oauth-provider` (replaces deprecated `oidcProvider`)
- `electron` → `@better-auth/electron`
- `i18n` → `@better-auth/i18n`

**Breaking (1.5):** `apiKey` is no longer exported from `better-auth/plugins`. The `userId` field on `ApiKey` is renamed to `referenceId`.

Client plugins go in `createAuthClient({ plugins: [...] })`.

**Session update:** `authClient.updateSession({ ...fields })` updates custom additional session fields without re-authentication.

---

## Client

Import from: `better-auth/client` (vanilla), `better-auth/react`, `better-auth/vue`, `better-auth/svelte`, `better-auth/solid`.

Key methods: `signUp.email()`, `signIn.email()`, `signIn.social()`, `signOut()`, `useSession()`, `getSession()`, `revokeSession()`, `revokeSessions()`.

---

## Type Safety

Infer types: `typeof auth.$Infer.Session`, `typeof auth.$Infer.Session.user`.

For separate client/server projects: `createAuthClient<typeof auth>()`.

---

## Common Gotchas

1. **Model vs table name** - Config uses ORM model name, not DB table name
2. **Plugin schema** - Re-run CLI after adding plugins
3. **Secondary storage** - Sessions go there by default, not DB
4. **Cookie cache** - Custom session fields NOT cached, always re-fetched
5. **Stateless mode** - No DB = session in cookie only, logout on cache expiry
6. **Change email flow** - Sends confirmation to old email, then sends verification to new email (`sendChangeEmailConfirmation` was renamed from `sendChangeEmailVerification`)
7. **After hooks** - Database `after` hooks run post-transaction; don't rely on them for atomic DB writes
8. **apiKey plugin** - Moved to `@better-auth/api-key` package; `userId` field renamed to `referenceId`
9. **getMigrations import** - Must now be imported from `better-auth/db/migration`, not `better-auth`

---

## Resources

- [Docs](https://better-auth.com/docs)
- [Options Reference](https://better-auth.com/docs/reference/options)
- [LLMs.txt](https://better-auth.com/llms.txt)
- [GitHub](https://github.com/better-auth/better-auth)
- [Init Options Source](https://github.com/better-auth/better-auth/blob/main/packages/core/src/types/init-options.ts)