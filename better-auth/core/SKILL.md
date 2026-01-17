---
name: better-auth-best-practices
description: Best practices for building Better Auth integrations. Use when implementing authentication, authorization, session management, OAuth providers, or any Better Auth integration.
---

When designing an authentication integration, always prefer [Better Auth's Documentation](https://better-auth.com/docs).

You should always default to the latest version of Better Auth unless the user specifies otherwise.

Better Auth is a TypeScript-first authentication framework that provides a secure, flexible, and developer-friendly approach to authentication. It supports multiple methods including email/password, OAuth providers, magic links, and passwordless authentication.

**Core Principles:**

1. **Packaging**: Most imports come from the npm package `better-auth` (initializer), `better-auth/plugins` (plugins), and `better-auth/adapters/*` (adapters). Some plugins use a scoped package like `@better-auth/stripe`.

2. **Env Vars**: Better Auth requires two environment variables:
   - `BETTER_AUTH_SECRET`: A high-entropy secret used for encryption and hashing (at least 32 characters).
   - `BETTER_AUTH_URL`: The base URL for the server where Better Auth is mounted.

3. **Auth Config**: Initialize Better Auth by calling `betterAuth` with a config object. Export it as `const auth` or as a default export unless you are wrapping the initializer (for example, to pass runtime context in edge environments). The CLI looks for `auth.ts` in `./`, `./lib`, `./utils`, or those directories under `src`.

4. **Database**: The database connection is the most important part of the auth config. If no database is provided, Better Auth uses the in-memory adapter (non-persistent). Built-in adapters support database clients like `pg`, `mysql2`, `better-sqlite3`, and `bun:sqlite`. You can also use adapters like `drizzleAdapter`, `prismaAdapter`, and `mongodbAdapter`.

5. **CLI**: Use the Better Auth CLI to manage schema:
   - `npx @better-auth/cli@latest migrate` applies schema directly for the built-in Kysely adapter.
   - `npx @better-auth/cli@latest generate` generates schema for Prisma/Drizzle (apply with your ORM).
   - The CLI supports `--config` to point to a non-default `auth.ts` location.

6. **Authentication Methods**: Configure methods in the auth config. Core methods include `emailAndPassword` and `socialProviders`. Enable email/password with `emailAndPassword: { enabled: true }`. Social providers are defined under `socialProviders`. For additional OAuth integrations, use the `genericOAuth` plugin. Other plugins include `username`, `phoneNumber`, `emailOtp`, `magicLink`, `passkey`, and `anonymous`. Plugins can add schema requirements, so re-run CLI generate/migrate when you add or change plugins.

7. **Handler**: All HTTP requests are handled by `auth.handler`. Mount it according to the framework-specific integration docs.

8. **Client**: Better Auth provides clients to interact with the auth server. Client files are typically named `auth-client.ts`.

React example client

```ts
import { createAuthClient } from "better-auth/react"
export const authClient = createAuthClient({
    /** The base URL of the server (optional if you're using the same domain) */
    baseURL: "http://localhost:3000"
})
```

Other than React, there are clients for vanilla JS, Vue, Svelte, and Solid. See https://www.better-auth.com/docs/concepts/client

9. **Type Safety**: Better Auth is built with TypeScript and provides full type safety. Infer types with `auth.$Infer`.

10. **Session**: Sessions are stored in the database unless you provide `secondaryStorage`, which can be used for session data or rate limiting in high-performance stores.

11. **Auth Options**: Review https://github.com/better-auth/better-auth/blob/main/packages/core/src/types/init-options.ts for the full configuration surface. Key skill areas include:
   - **Rate limiting**: Configure `rateLimit` defaults, per-path `customRules`, and storage (`memory`, `database`, `secondary-storage`, or `customStorage`).
   - **Advanced security**: Use `advanced.useSecureCookies`, and be cautious with `advanced.disableCSRFCheck` or `advanced.disableOriginCheck` (these reduce protections).
   - **Trusted origins**: Prefer dynamic `trustedOrigins` when running behind proxies or multiple hostnames.
   - **Cookie strategy**: Customize cookie names/attributes, prefixes, and cross-subdomain settings via `advanced.cookies`, `advanced.cookiePrefix`, and `advanced.crossSubDomainCookies`.
   - **Session tuning**: Adjust `session.expiresIn`, `session.updateAge`, `session.cookieCache`, and `session.freshAge` based on UX/security needs.
   - **Account linking**: Configure `account.accountLinking` carefully, especially `allowDifferentEmails` and `allowUnlinkingAll`.
   - **Email flows**: Use `emailVerification` hooks and `emailAndPassword.sendResetPassword` to implement secure verification and recovery.
   - **Data model mapping**: Map fields and add metadata via `user.fields`, `session.fields`, `account.fields`, and `verification.fields`.
   - **Database hooks**: Use `databaseHooks` to enforce policies or auditing on user/session/account/verification lifecycle events.
   - **Error handling**: Centralize response behavior with `onAPIError` and customize the default error page if needed.
   - **Telemetry**: Control data collection with `telemetry.enabled` and `telemetry.debug`.
   - **Experimental flags**: Keep `experimental.joins` off unless the adapter supports it and the docs recommend it.

**Resources:**
- [Better Auth Documentation](https://better-auth.com/docs)
- [GitHub Repository](https://github.com/better-auth/better-auth)
- [Examples](https://better-auth.com/docs/examples)

Always consult the official documentation for the most up-to-date best practices and API changes.
