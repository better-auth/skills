---
name: Better Auth Best Practices
description: A skill to integrate better-auth. And auth framework for TypeScript projects.
---

Best practices for building Better Auth integrations. Use when implementing authentication, authorization, session management, OAuth providers, or any Better Auth integration.

When designing an authentication integration, always prefer [Better Auth's Documentation](https://better-auth.com/docs).

You should always default to the latest version of Better Auth unless the user specifies otherwise.

Better Auth is a TypeScript-first authentication framework that provides a secure, flexible, and developer-friendly approach to authentication. It supports multiple methods including email/password, OAuth providers, magic links, and passwordless authentication.

## Core Principles

### 1. Packaging

Most imports come from the npm package `better-auth` (initializer), `better-auth/plugins` (plugins), and `better-auth/adapters/*` (adapters). Some plugins use a scoped package like `@better-auth/stripe`.

### 2. Environment Variables

Better Auth requires two environment variables:
- `BETTER_AUTH_SECRET`: A high-entropy secret used for encryption and hashing (at least 32 characters).
- `BETTER_AUTH_URL`: The base URL for the server where Better Auth is mounted.

### 3. Auth Config

Initialize Better Auth by calling `betterAuth` with a config object. Export it as `const auth` or as a default export unless you are wrapping the initializer (for example, to pass runtime context in edge environments). The CLI looks for `auth.ts` in `./`, `./lib`, `./utils`, or those directories under `src`.

### 4. Database

The database connection is the most important part of the auth config. If no database is provided, Better Auth uses the in-memory adapter (non-persistent). Built-in adapters support database clients like `pg`, `mysql2`, `better-sqlite3`, and `bun:sqlite`. You can also use adapters like `drizzleAdapter`, `prismaAdapter`, and `mongodbAdapter`.

### 5. CLI

Use the Better Auth CLI to manage schema:
- `npx @better-auth/cli@latest migrate` applies schema directly for the built-in Kysely adapter.
- `npx @better-auth/cli@latest generate` generates schema for Prisma/Drizzle (apply with your ORM).
- The CLI supports `--config` to point to a non-default `auth.ts` location.

### 6. Authentication Methods

Configure methods in the auth config. Core methods include `emailAndPassword` and `socialProviders`. Enable email/password with `emailAndPassword: { enabled: true }`. Social providers are defined under `socialProviders`. For additional OAuth integrations, use the `genericOAuth` plugin. Other plugins include `username`, `phoneNumber`, `emailOtp`, `magicLink`, `passkey`, and `anonymous`. Plugins can add schema requirements, so re-run CLI generate/migrate when you add or change plugins.

### 7. Handler

All HTTP requests are handled by `auth.handler`. Mount it according to the framework-specific integration docs.

### 8. Client

Better Auth provides clients to interact with the auth server. Client files are typically named `auth-client.ts`.

React example client:

```ts
import { createAuthClient } from "better-auth/react"
export const authClient = createAuthClient({
    /** The base URL of the server (optional if you're using the same domain) */
    baseURL: "http://localhost:3000"
})
```

Other than React, there are clients for vanilla JS, Vue, Svelte, and Solid. See https://www.better-auth.com/docs/concepts/client

### 9. Type Safety

Better Auth is built with TypeScript and provides full type safety. Infer types with `auth.$Infer`.

### 10. Session

Sessions are stored in the database unless you provide `secondaryStorage`, which can be used for session data or rate limiting in high-performance stores.

### 11. Auth Options

Review https://github.com/better-auth/better-auth/blob/main/packages/core/src/types/init-options.ts for the full configuration surface. Key skill areas include:

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

---

## Plugin Architecture

Plugins follow a consistent structure with these key properties:

| Property | Description |
|----------|-------------|
| `id` | Unique string identifier |
| `endpoints` | Object of `createAuthEndpoint` calls that define API routes |
| `schema` | Database schema additions the plugin needs (triggers migrations) |
| `hooks.before` / `hooks.after` | Middleware that runs before/after matched endpoints |
| `init` | Called during auth initialization to modify context or options |
| `onRequest` / `onResponse` | Request/response interceptors |
| `$ERROR_CODES` | Plugin-specific error code constants |
| `rateLimit` | Custom rate limit rules for plugin endpoints |

When building custom plugins, use `createAuthEndpoint` and `createAuthMiddleware` from `@better-auth/core/api`. Use `sessionMiddleware` from `better-auth/api` to require authentication.

### Example Plugin Structure

```ts
import type { BetterAuthPlugin } from "@better-auth/core";
import { createAuthEndpoint, createAuthMiddleware } from "@better-auth/core/api";
import { sessionMiddleware } from "better-auth/api";
import { z } from "zod";

export const myPlugin = (options?: MyPluginOptions) => {
  return {
    id: "my-plugin",
    endpoints: {
      myEndpoint: createAuthEndpoint(
        "/my-plugin/action",
        {
          method: "POST",
          body: z.object({ data: z.string() }),
          use: [sessionMiddleware], // Require authentication
        },
        async (ctx) => {
          const user = ctx.context.session.user;
          // Implementation
          return { success: true };
        }
      ),
    },
    schema: {
      myTable: {
        fields: {
          userId: { type: "string", references: { model: "user", field: "id" } },
          data: { type: "string" },
        },
      },
    },
    $ERROR_CODES: {
      MY_ERROR: "My custom error message",
    },
  } satisfies BetterAuthPlugin;
};
```

---

## Client-Server Type Inference

Use `auth.$Infer` on the server and client plugins expose `$InferServerPlugin` for type bridging:

```ts
// Server plugin exports
export const myPlugin = () => ({
  id: "my-plugin",
  $ERROR_CODES: MY_ERROR_CODES,
  // ...
}) satisfies BetterAuthPlugin;

// Client plugin
export const myPluginClient = () => ({
  $InferServerPlugin: {} as ReturnType<typeof myPlugin>,
  getActions: ($fetch) => ({
    myAction: async () => {
      return $fetch("/my-plugin/action", { method: "POST" });
    },
  }),
}) satisfies BetterAuthClientPlugin;
```

---

## Social Providers

### Built-in Providers

`apple`, `atlassian`, `cognito`, `discord`, `dropbox`, `facebook`, `figma`, `github`, `gitlab`, `google`, `huggingface`, `kakao`, `kick`, `line`, `linear`, `linkedin`, `microsoft`, `naver`, `notion`, `paybin`, `paypal`, `polar`, `reddit`, `roblox`, `salesforce`, `slack`, `spotify`, `tiktok`, `twitch`, `twitter`, `vercel`, `vk`, `zoom`

### Generic OAuth

For providers not in this list, use the `genericOAuth` plugin with pre-built configurations for Auth0, Gumroad, HubSpot, Keycloak, Line, Microsoft Entra ID, Okta, Patreon, and Slack.

```ts
import { betterAuth } from "better-auth";
import { genericOAuth } from "better-auth/plugins";

const auth = betterAuth({
  // Built-in providers
  socialProviders: {
    github: {
      clientId: process.env.GITHUB_CLIENT_ID!,
      clientSecret: process.env.GITHUB_CLIENT_SECRET!,
    },
  },
  // Generic OAuth for other providers
  plugins: [
    genericOAuth({
      config: [
        {
          providerId: "custom-provider",
          clientId: process.env.CUSTOM_CLIENT_ID!,
          clientSecret: process.env.CUSTOM_CLIENT_SECRET!,
          authorizationUrl: "https://provider.com/oauth/authorize",
          tokenUrl: "https://provider.com/oauth/token",
          userInfoUrl: "https://provider.com/api/user",
        },
      ],
    }),
  ],
});
```

---

## Scoped Packages

Some plugins live in separate packages:

| Package | Description |
|---------|-------------|
| `@better-auth/stripe` | Stripe subscriptions and payments |
| `@better-auth/expo` | Expo/React Native client |
| `@better-auth/passkey` | Passkey/WebAuthn authentication |
| `@better-auth/sso` | SAML/OIDC enterprise SSO |
| `@better-auth/scim` | SCIM user provisioning |

---

## Framework Integrations

### Next.js

```ts
import { toNextJsHandler } from "better-auth/integrations/next-js";
import { auth } from "./auth";

export const { GET, POST } = toNextJsHandler(auth);
```

For Server Components cookie handling, add `nextCookies()` plugin:

```ts
import { nextCookies } from "better-auth/integrations/next-js";

const auth = betterAuth({
  plugins: [nextCookies()],
});
```

### Other Frameworks

- **SvelteKit**: Use `svelteKitHandler()`
- **SolidStart**: Use `solidStartHandler()`
- **TanStack Start**: Use `tanstackStartHandler()`
- **Node/Express**: Mount `auth.handler` directly as middleware

---

## Testing

For unit tests, use `getTestInstance()` from `better-auth/test-utils`:

```ts
import { getTestInstance } from "better-auth/test-utils";
import { describe, it, expect } from "vitest";

describe("my auth tests", () => {
  it("should authenticate user", async () => {
    const { client, signInWithTestUser } = await getTestInstance({
      plugins: [myPlugin()],
    });

    const { headers, user } = await signInWithTestUser();
    
    const result = await client.myPlugin.action({
      fetchOptions: { headers },
    });
    
    expect(result.data).toBeDefined();
  });
});
```

### What `getTestInstance()` Provides

- In-memory SQLite database by default (or postgres/mysql/mongodb via config)
- Automatic migration execution
- Pre-configured client with custom fetch
- Test user (`test@test.com` / `test123456`)
- `signInWithTestUser()` for authenticated test flows
- Automatic cleanup via `afterAll`

---

## Account Linking

Configure `account.accountLinking` carefully:

| Option | Description |
|--------|-------------|
| `enabled` | Toggle account linking (default: true) |
| `trustedProviders` | Array of provider IDs that can link without email verification |
| `allowDifferentEmails` | Allow linking accounts with different emails (security risk) |
| `allowUnlinkingAll` | Allow users to unlink all OAuth accounts |

Accounts are automatically linked when a user signs in with a different provider using the same verified email.

```ts
const auth = betterAuth({
  account: {
    accountLinking: {
      enabled: true,
      trustedProviders: ["google", "github"], // These can link without email verification
      allowDifferentEmails: false, // Keep false for security
    },
  },
});
```

---

## Session Cookie Strategies

`session.cookieCache.strategy` options:

| Strategy | Description |
|----------|-------------|
| `"compact"` | (default) Stores minimal session data in cookies |
| `"full"` | Stores complete session/user data |
| `"jwt"` | Uses signed JWT for stateless verification (requires `jwt` plugin) |

```ts
const auth = betterAuth({
  session: {
    cookieCache: {
      enabled: true,
      strategy: "compact",
      maxAge: 60 * 5, // 5 minutes
    },
  },
});
```

---

## Internal Adapter

Access database operations via `ctx.context.internalAdapter` in endpoints:

- `findUserByEmail()`, `findUserById()`
- `createUser()`, `updateUser()`, `deleteUser()`
- `createSession()`, `findSession()`, `deleteSession()`
- `linkAccount()`, `findOAuthUser()`
- `createVerification()`, `findVerification()`

These methods automatically apply `databaseHooks` (before/after lifecycle events).

---

## Database Hooks

Use `databaseHooks` for lifecycle events on core models:

```ts
const auth = betterAuth({
  databaseHooks: {
    user: {
      create: {
        before: async (user, ctx) => {
          // Modify user before creation
          return { data: { ...user, customField: "value" } };
        },
        after: async (user, ctx) => {
          // Audit log, send welcome email, etc.
          await sendWelcomeEmail(user.email);
        },
      },
    },
    session: {
      create: {
        after: async (session, ctx) => {
          // Track session creation
        },
      },
    },
  },
});
```

---

## Error Handling

Use `APIError` from `better-call` for throwing errors:

```ts
import { APIError } from "better-call";
import { BASE_ERROR_CODES } from "@better-auth/core/error";

// In an endpoint
throw new APIError("BAD_REQUEST", { 
  message: BASE_ERROR_CODES.INVALID_PASSWORD 
});

// Custom error codes in plugins
export const MY_ERROR_CODES = {
  CUSTOM_ERROR: "Custom error occurred",
  VALIDATION_FAILED: "Validation failed",
} as const;
```

Error codes should be exported as `$ERROR_CODES` in plugins. Reference `BASE_ERROR_CODES` from `@better-auth/core/error` for standard codes.

---

## Background Tasks

For edge runtimes (Vercel, Cloudflare Workers), configure `advanced.backgroundTasks.handler` with the platform's `waitUntil` function:

```ts
// Vercel
import { waitUntil } from "@vercel/functions";

const auth = betterAuth({
  advanced: {
    backgroundTasks: {
      handler: waitUntil,
    },
  },
});

// Cloudflare Workers (with AsyncLocalStorage)
const auth = betterAuth({
  advanced: {
    backgroundTasks: {
      handler: (p) => execCtxStorage.getStore()?.waitUntil(p),
    },
  },
});
```

---

## Secondary Storage

Use `secondaryStorage` for high-performance session/rate-limit storage:

```ts
import { betterAuth } from "better-auth";
import { Redis } from "ioredis";

const redis = new Redis();

const auth = betterAuth({
  secondaryStorage: {
    get: async (key) => {
      const value = await redis.get(key);
      return value ? JSON.parse(value) : null;
    },
    set: async (key, value, ttl) => {
      if (ttl) {
        await redis.setex(key, ttl, JSON.stringify(value));
      } else {
        await redis.set(key, JSON.stringify(value));
      }
    },
    delete: async (key) => {
      await redis.del(key);
    },
  },
});
```

---

## IP Address Handling

Configure IP address extraction for rate limiting and session tracking:

```ts
const auth = betterAuth({
  advanced: {
    ipAddress: {
      ipAddressHeaders: ["x-real-ip", "x-forwarded-for", "cf-connecting-ip"],
      disableIpTracking: false, // Only disable if absolutely necessary
    },
  },
});
```

---

## Resources

- [Better Auth Documentation](https://better-auth.com/docs)
- [GitHub Repository](https://github.com/better-auth/better-auth)
- [Examples](https://better-auth.com/docs/examples)
- [Init Options Reference](https://github.com/better-auth/better-auth/blob/main/packages/core/src/types/init-options.ts)

Always consult the official documentation for the most up-to-date best practices and API changes.
