---
name: create-auth-skill
description: A skill to create auth service for new applications.
---
# Create Auth Skill

A skill to create authentication and authorization layers for TypeScript/JavaScript applications using Better Auth.

## Overview

This skill guides you through creating authentication and authorization layers for TypeScript/JavaScript applications using Better Auth. It covers new projects, existing projects without auth, and improving existing auth implementations.

## Decision Tree: Choosing Your Approach

```
User Request → Is this a new/empty project?
    │
    ├─ YES → Create new project with auth scaffolding
    │         ├─ 1. Identify framework (Next.js, Express, SvelteKit, etc.)
    │         ├─ 2. Choose database (PostgreSQL, SQLite, MongoDB, etc.)
    │         ├─ 3. Install dependencies
    │         ├─ 4. Create auth.ts and auth-client.ts
    │         ├─ 5. Set up API route handler
    │         ├─ 6. Run CLI to generate/migrate schema
    │         └─ 7. Add auth features (OAuth, 2FA, organizations, etc.)
    │
    └─ NO → Does the project already have authentication?
              │
              ├─ YES → Review and enhance existing implementation
              │         ├─ Audit current auth for security gaps
              │         ├─ Identify missing features
              │         ├─ Plan migration path if switching to Better Auth
              │         └─ Implement improvements incrementally
              │
              └─ NO → Add auth to existing project
                        ├─ 1. Analyze project structure and framework
                        ├─ 2. Install better-auth
                        ├─ 3. Create auth configuration
                        ├─ 4. Set up route handler
                        ├─ 5. Run schema migrations
                        └─ 6. Integrate auth into existing pages/components
```

---

## Step-by-Step Implementation

### Step 1: Install Dependencies

```bash
# Core package (always required)
npm install better-auth

# Optional: Scoped packages for specific features
npm install @better-auth/stripe    # Stripe payments/subscriptions
npm install @better-auth/passkey   # WebAuthn/Passkey authentication
npm install @better-auth/sso       # SAML/OIDC enterprise SSO
npm install @better-auth/scim      # SCIM user provisioning
npm install @better-auth/expo      # Expo/React Native support
```

### Step 2: Environment Variables

Create or update `.env`:

```env
# Required
BETTER_AUTH_SECRET=your-secret-key-at-least-32-characters-long
BETTER_AUTH_URL=http://localhost:3000

# Database (choose one)
DATABASE_URL=postgresql://user:password@localhost:5432/myapp
# or for SQLite
# DATABASE_URL=file:./auth.db

# OAuth Providers (add as needed)
GITHUB_CLIENT_ID=
GITHUB_CLIENT_SECRET=
GOOGLE_CLIENT_ID=
GOOGLE_CLIENT_SECRET=
```

Generate a secure secret:
```bash
openssl rand -base64 32
```

### Step 3: Create Server-Side Auth Configuration

Create `lib/auth.ts` (or `src/lib/auth.ts`):

#### Minimal Setup (SQLite, email/password only)

```ts
import { betterAuth } from "better-auth";
import Database from "better-sqlite3";

export const auth = betterAuth({
  database: new Database("./auth.db"),
  emailAndPassword: {
    enabled: true,
  },
});
```

#### Standard Setup (PostgreSQL, OAuth providers)

```ts
import { betterAuth } from "better-auth";
import { Pool } from "pg";

export const auth = betterAuth({
  database: new Pool({
    connectionString: process.env.DATABASE_URL,
  }),
  emailAndPassword: {
    enabled: true,
    sendResetPassword: async ({ user, url }) => {
      // Implement email sending
      console.log(`Reset password for ${user.email}: ${url}`);
    },
  },
  emailVerification: {
    sendVerificationEmail: async ({ user, url }) => {
      // Implement email sending
      console.log(`Verify email for ${user.email}: ${url}`);
    },
    sendOnSignUp: true,
  },
  socialProviders: {
    github: {
      clientId: process.env.GITHUB_CLIENT_ID!,
      clientSecret: process.env.GITHUB_CLIENT_SECRET!,
    },
    google: {
      clientId: process.env.GOOGLE_CLIENT_ID!,
      clientSecret: process.env.GOOGLE_CLIENT_SECRET!,
    },
  },
});

// Export types for client-side usage
export type Session = typeof auth.$Infer.Session;
```

#### Full-Featured Setup (with plugins)

```ts
import { betterAuth } from "better-auth";
import { Pool } from "pg";
import {
  admin,
  organization,
  twoFactor,
  bearer,
  openAPI,
} from "better-auth/plugins";
import { passkey } from "@better-auth/passkey";

export const auth = betterAuth({
  database: new Pool({
    connectionString: process.env.DATABASE_URL,
  }),
  emailAndPassword: {
    enabled: true,
    sendResetPassword: async ({ user, url }) => {
      // Send password reset email
    },
  },
  emailVerification: {
    sendVerificationEmail: async ({ user, url }) => {
      // Send verification email
    },
    sendOnSignUp: true,
    autoSignInAfterVerification: true,
  },
  socialProviders: {
    github: {
      clientId: process.env.GITHUB_CLIENT_ID!,
      clientSecret: process.env.GITHUB_CLIENT_SECRET!,
    },
    google: {
      clientId: process.env.GOOGLE_CLIENT_ID!,
      clientSecret: process.env.GOOGLE_CLIENT_SECRET!,
    },
  },
  plugins: [
    // Two-factor authentication
    twoFactor({
      otpOptions: {
        async sendOTP({ user, otp }) {
          // Send OTP via email/SMS
        },
      },
    }),
    // Passkey/WebAuthn
    passkey(),
    // Organization/team management
    organization({
      async sendInvitationEmail(data) {
        // Send invitation email
      },
    }),
    // Admin panel
    admin(),
    // Bearer token auth for APIs
    bearer(),
    // OpenAPI documentation
    openAPI(),
  ],
  // Session configuration
  session: {
    expiresIn: 60 * 60 * 24 * 7, // 7 days
    updateAge: 60 * 60 * 24, // Update session every 24 hours
    cookieCache: {
      enabled: true,
      maxAge: 60 * 5, // 5 minutes
    },
  },
  // Account linking
  account: {
    accountLinking: {
      enabled: true,
      trustedProviders: ["google", "github"],
    },
  },
  // Rate limiting
  rateLimit: {
    window: 10,
    max: 100,
  },
});

export type Session = typeof auth.$Infer.Session;
export type ActiveOrganization = typeof auth.$Infer.ActiveOrganization;
```

### Step 4: Create Client-Side Auth

Create `lib/auth-client.ts`:

#### React/Next.js Client

```ts
import { createAuthClient } from "better-auth/react";
import type { auth } from "./auth";

export const authClient = createAuthClient({
  // baseURL is optional if on same domain
  // baseURL: "http://localhost:3000",
});

// Export commonly used functions
export const { 
  signIn, 
  signUp, 
  signOut, 
  useSession,
  getSession,
} = authClient;
```

#### Client with Plugins

```ts
import { createAuthClient } from "better-auth/react";
import {
  adminClient,
  organizationClient,
  twoFactorClient,
} from "better-auth/client/plugins";
import { passkeyClient } from "@better-auth/passkey/client";
import type { auth } from "./auth";

export const authClient = createAuthClient({
  plugins: [
    organizationClient(),
    twoFactorClient({
      onTwoFactorRedirect() {
        window.location.href = "/two-factor";
      },
    }),
    passkeyClient(),
    adminClient(),
  ],
  fetchOptions: {
    onError(e) {
      if (e.error.status === 429) {
        // Handle rate limiting
        console.error("Too many requests");
      }
    },
  },
});

export const {
  signIn,
  signUp,
  signOut,
  useSession,
} = authClient;
```

#### Other Framework Clients

```ts
// Vue
import { createAuthClient } from "better-auth/vue";

// Svelte
import { createAuthClient } from "better-auth/svelte";

// Solid
import { createAuthClient } from "better-auth/solid";

// Vanilla JS
import { createAuthClient } from "better-auth/client";
```

### Step 5: Set Up API Route Handler

#### Next.js (App Router)

Create `app/api/auth/[...all]/route.ts`:

```ts
import { toNextJsHandler } from "better-auth/next-js";
import { auth } from "@/lib/auth";

export const { GET, POST } = toNextJsHandler(auth);
```

For Server Components cookie support, add `nextCookies()` plugin to auth config:

```ts
import { nextCookies } from "better-auth/next-js";

export const auth = betterAuth({
  // ... other config
  plugins: [
    nextCookies(),
    // ... other plugins
  ],
});
```

#### Next.js (Pages Router)

Create `pages/api/auth/[...all].ts`:

```ts
import { toNextJsHandler } from "better-auth/next-js";
import { auth } from "@/lib/auth";

export default toNextJsHandler(auth);
```

#### Express

```ts
import express from "express";
import { toNodeHandler } from "better-auth/node";
import { auth } from "./auth";

const app = express();

app.all("/api/auth/*", toNodeHandler(auth));

app.listen(3000);
```

#### SvelteKit

Create `src/hooks.server.ts`:

```ts
import { svelteKitHandler } from "better-auth/svelte-kit";
import { auth } from "$lib/auth";

export const handle = svelteKitHandler(auth);
```

#### SolidStart

```ts
import { solidStartHandler } from "better-auth/solid-start";
import { auth } from "./auth";

export const { GET, POST } = solidStartHandler(auth);
```

### Step 6: Run Database Migrations

```bash
# For built-in Kysely adapter (applies directly)
npx @better-auth/cli@latest migrate

# For Prisma (generates schema, apply with prisma migrate)
npx @better-auth/cli@latest generate --output prisma/schema.prisma
npx prisma migrate dev

# For Drizzle (generates schema, apply with drizzle-kit)
npx @better-auth/cli@latest generate --output src/db/auth-schema.ts
npx drizzle-kit push
```

### Step 7: Implement Auth UI Components

#### Sign In Page Example (React)

```tsx
"use client";

import { useState } from "react";
import { signIn } from "@/lib/auth-client";
import { useRouter } from "next/navigation";

export function SignInForm() {
  const [email, setEmail] = useState("");
  const [password, setPassword] = useState("");
  const [error, setError] = useState("");
  const [loading, setLoading] = useState(false);
  const router = useRouter();

  const handleSubmit = async (e: React.FormEvent) => {
    e.preventDefault();
    setLoading(true);
    setError("");

    const { error } = await signIn.email({
      email,
      password,
    });

    if (error) {
      setError(error.message);
      setLoading(false);
    } else {
      router.push("/dashboard");
    }
  };

  const handleOAuthSignIn = async (provider: "github" | "google") => {
    await signIn.social({
      provider,
      callbackURL: "/dashboard",
    });
  };

  return (
    <form onSubmit={handleSubmit}>
      {error && <div className="error">{error}</div>}
      
      <input
        type="email"
        value={email}
        onChange={(e) => setEmail(e.target.value)}
        placeholder="Email"
        required
      />
      
      <input
        type="password"
        value={password}
        onChange={(e) => setPassword(e.target.value)}
        placeholder="Password"
        required
      />
      
      <button type="submit" disabled={loading}>
        {loading ? "Signing in..." : "Sign In"}
      </button>
      
      <div className="oauth-buttons">
        <button type="button" onClick={() => handleOAuthSignIn("github")}>
          Sign in with GitHub
        </button>
        <button type="button" onClick={() => handleOAuthSignIn("google")}>
          Sign in with Google
        </button>
      </div>
    </form>
  );
}
```

#### Protected Route Example

```tsx
"use client";

import { useSession } from "@/lib/auth-client";
import { redirect } from "next/navigation";

export function ProtectedPage() {
  const { data: session, isPending } = useSession();

  if (isPending) {
    return <div>Loading...</div>;
  }

  if (!session) {
    redirect("/sign-in");
  }

  return (
    <div>
      <h1>Welcome, {session.user.name}!</h1>
      <p>Email: {session.user.email}</p>
    </div>
  );
}
```

#### Server-Side Session Check (Next.js)

```ts
import { auth } from "@/lib/auth";
import { headers } from "next/headers";
import { redirect } from "next/navigation";

export default async function DashboardPage() {
  const session = await auth.api.getSession({
    headers: await headers(),
  });

  if (!session) {
    redirect("/sign-in");
  }

  return (
    <div>
      <h1>Dashboard</h1>
      <p>Welcome, {session.user.name}!</p>
    </div>
  );
}
```

---

## Common Auth Features

### Two-Factor Authentication

```ts
// Server: Add plugin
import { twoFactor } from "better-auth/plugins";

plugins: [
  twoFactor({
    otpOptions: {
      async sendOTP({ user, otp }) {
        await sendEmail({
          to: user.email,
          subject: "Your verification code",
          body: `Your code is: ${otp}`,
        });
      },
    },
  }),
]

// Client: Enable 2FA
const { data } = await authClient.twoFactor.enable({
  password: "user-password",
});
// data.totpURI - QR code URI for authenticator apps
// data.backupCodes - Save these for recovery
```

### Organization/Team Management

```ts
// Server: Add plugin
import { organization } from "better-auth/plugins";

plugins: [
  organization({
    async sendInvitationEmail(data) {
      await sendEmail({
        to: data.email,
        subject: `Join ${data.organization.name}`,
        body: `You've been invited by ${data.inviter.user.name}`,
      });
    },
  }),
]

// Client
import { organizationClient } from "better-auth/client/plugins";

// Create organization
await authClient.organization.create({
  name: "My Team",
  slug: "my-team",
});

// Invite member
await authClient.organization.inviteMember({
  email: "member@example.com",
  role: "member",
});
```

### Admin Panel

```ts
// Server
import { admin } from "better-auth/plugins";

plugins: [
  admin({
    adminUserIds: ["admin-user-id"],
  }),
]

// Client
import { adminClient } from "better-auth/client/plugins";

// List all users (admin only)
const users = await authClient.admin.listUsers({
  limit: 10,
  offset: 0,
});

// Ban user
await authClient.admin.banUser({
  userId: "user-id",
});
```

### Passkey/WebAuthn

```ts
// Server
import { passkey } from "@better-auth/passkey";

plugins: [passkey()]

// Client
import { passkeyClient } from "@better-auth/passkey/client";

// Register passkey
await authClient.passkey.addPasskey();

// Sign in with passkey
await authClient.signIn.passkey();
```

---

## Database Adapters

### Built-in Kysely Adapter (Recommended for quick setup)

```ts
import { betterAuth } from "better-auth";
import Database from "better-sqlite3";
import { Pool } from "pg";

// SQLite
const auth = betterAuth({
  database: new Database("./auth.db"),
});

// PostgreSQL
const auth = betterAuth({
  database: new Pool({ connectionString: process.env.DATABASE_URL }),
});
```

### Prisma Adapter

```ts
import { betterAuth } from "better-auth";
import { prismaAdapter } from "better-auth/adapters/prisma";
import { PrismaClient } from "@prisma/client";

const prisma = new PrismaClient();

const auth = betterAuth({
  database: prismaAdapter(prisma, {
    provider: "postgresql", // or "mysql", "sqlite"
  }),
});
```

### Drizzle Adapter

```ts
import { betterAuth } from "better-auth";
import { drizzleAdapter } from "better-auth/adapters/drizzle";
import { db } from "./db";

const auth = betterAuth({
  database: drizzleAdapter(db, {
    provider: "pg", // or "mysql", "sqlite"
  }),
});
```

### MongoDB Adapter

```ts
import { betterAuth } from "better-auth";
import { mongodbAdapter } from "better-auth/adapters/mongodb";
import { MongoClient } from "mongodb";

const client = new MongoClient(process.env.MONGODB_URI!);
const db = client.db("myapp");

const auth = betterAuth({
  database: mongodbAdapter(db),
});
```

---

## Security Checklist

- [ ] Set `BETTER_AUTH_SECRET` to a strong, unique value (32+ characters)
- [ ] Enable HTTPS in production (`advanced.useSecureCookies: true`)
- [ ] Configure `trustedOrigins` for cross-origin requests
- [ ] Set appropriate rate limits for auth endpoints
- [ ] Enable email verification for new accounts
- [ ] Implement password reset flow
- [ ] Consider 2FA for sensitive applications
- [ ] Review `account.accountLinking` settings
- [ ] Never disable CSRF protection in production
- [ ] Use `databaseHooks` for audit logging

---

## CLI Reference

```bash
# Generate database schema
npx @better-auth/cli@latest generate

# Apply migrations (Kysely adapter only)
npx @better-auth/cli@latest migrate

# Specify config file location
npx @better-auth/cli@latest migrate --config ./src/lib/auth.ts

# Generate for specific output
npx @better-auth/cli@latest generate --output ./prisma/schema.prisma
```

---

## Resources

- [Better Auth Documentation](https://better-auth.com/docs)
- [Examples Repository](https://github.com/better-auth/examples)
- [Plugins Reference](https://better-auth.com/docs/concepts/plugins)
- [CLI Documentation](https://better-auth.com/docs/concepts/cli)
- [API Reference](https://better-auth.com/docs/api-reference)

---

## Troubleshooting

### Common Issues

1. **"Secret not set"**: Add `BETTER_AUTH_SECRET` to environment variables
2. **"Invalid Origin"**: Add your domain to `trustedOrigins`
3. **Cookies not setting**: Check `baseURL` matches your domain, enable secure cookies in production
4. **OAuth callback errors**: Verify redirect URIs in provider dashboard match your auth config
