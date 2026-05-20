# Auth Coding Standards

## Provider

**This app uses [Clerk](https://clerk.com) for authentication.** Do not introduce any other auth library (NextAuth, Auth.js, Lucia, etc.).

`ClerkProvider` must wrap the app at the root layout (`app/layout.tsx`) and nowhere else. It is already in place — do not add it to nested layouts or individual pages.

```tsx
// app/layout.tsx
import { ClerkProvider } from "@clerk/nextjs";

export default function RootLayout({
  children,
}: {
  children: React.ReactNode;
}) {
  return (
    <html>
      <body>
        <ClerkProvider>{children}</ClerkProvider>
      </body>
    </html>
  );
}
```

## Proxy (route protection)

Route protection is handled by `clerkMiddleware()` in `proxy.ts`. This is the Next.js 16 replacement for `middleware.ts` — do not create a `middleware.ts` file.

```ts
// proxy.ts
import { clerkMiddleware } from "@clerk/nextjs/server";

export default clerkMiddleware();
```

## Import rules

Which package you import from depends strictly on where the code runs:

| Context                                            | Import from            |
| -------------------------------------------------- | ---------------------- |
| Server Components, Server Actions, `/data` helpers | `@clerk/nextjs/server` |
| Client Components (`"use client"`)                 | `@clerk/nextjs`        |

Never import server-only exports (`auth`, `currentUser`) in a Client Component, and never import client-only hooks (`useAuth`, `useUser`) in a Server Component.

## Getting the current user (Server Components)

Use `auth()` from `@clerk/nextjs/server`. It is async — always `await` it.

```ts
import { auth } from "@clerk/nextjs/server";

const { userId } = await auth();
```

- `userId` is `string | null`. If `null`, the user is not signed in.
- Redirect unauthenticated users to `/sign-in` using Next.js's `redirect()`:

```ts
import { auth } from "@clerk/nextjs/server";
import { redirect } from "next/navigation";

const { userId } = await auth();
if (!userId) redirect("/sign-in");
```

- Pass `userId` down to `/data` helpers as a parameter. Never expose it to the client or derive it from client input (see the data-fetching doc).

## Auth UI (Client Components)

Use Clerk's pre-built components for all sign-in/sign-up UI. Do not build custom auth forms.

```tsx
"use client";

import { SignInButton, SignUpButton, UserButton, useAuth } from "@clerk/nextjs";
```

| Component / Hook              | Purpose                                                 |
| ----------------------------- | ------------------------------------------------------- |
| `<SignInButton mode="modal">` | Triggers sign-in modal                                  |
| `<SignUpButton mode="modal">` | Triggers sign-up modal                                  |
| `<UserButton />`              | Avatar/account dropdown for signed-in users             |
| `useAuth()`                   | Read `isSignedIn`, `userId`, etc. in a Client Component |
| `useUser()`                   | Access the full `User` object in a Client Component     |

## Explicitly Prohibited

| Prohibited                                                                  | Why                                                            |
| --------------------------------------------------------------------------- | -------------------------------------------------------------- |
| Any auth library other than Clerk                                           | One auth provider only                                         |
| `import { auth } from "@clerk/nextjs"` (no `/server`) in a Server Component | Wrong package; use `@clerk/nextjs/server` on the server        |
| `useAuth()` / `useUser()` in a Server Component                             | Hooks are client-only                                          |
| Custom sign-in/sign-up forms                                                | Use Clerk's built-in components                                |
| `middleware.ts`                                                             | Deprecated in Next.js 16; use `proxy.ts`                       |
| Deriving `userId` from URL params, query strings, or request bodies         | Client input is attacker-controlled; always use `await auth()` |
| Calling `/data` helpers before checking `userId`                            | Helpers must never be reachable by an unauthenticated caller   |

## Summary

| Concern                  | Solution                                                          |
| ------------------------ | ----------------------------------------------------------------- |
| Provider                 | `ClerkProvider` in root layout only                               |
| Route protection         | `clerkMiddleware()` in `proxy.ts`                                 |
| Server-side identity     | `const { userId } = await auth()` from `@clerk/nextjs/server`     |
| Unauthenticated redirect | `if (!userId) redirect("/sign-in")`                               |
| Auth UI                  | `SignInButton`, `SignUpButton`, `UserButton` from `@clerk/nextjs` |
| Client-side state        | `useAuth()` or `useUser()` from `@clerk/nextjs`                   |
