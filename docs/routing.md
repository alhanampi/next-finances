# Routing Coding Standards

## Route Structure

All application routes live under `/dashboard`. The root route (`/`) is a public landing page only — it contains no authenticated content.

```
app/
├── page.tsx                        # public landing page
└── dashboard/
    ├── page.tsx                    # protected: /dashboard
    └── [feature]/
        └── page.tsx                # protected: /dashboard/[feature]
```

Never create authenticated pages outside of `app/dashboard/`. If a new feature needs its own page, it belongs at `app/dashboard/[feature]/page.tsx`, not at the root.

## Route Protection

All `/dashboard` routes are protected — unauthenticated users must not be able to access them. Protection is enforced at the proxy layer, not inside individual pages or layouts.

### proxy.ts (the only correct approach)

Use `clerkMiddleware` with `createRouteMatcher` in `proxy.ts`. This is the Next.js 16 convention — `middleware.ts` is deprecated and must not be used.

```ts
// proxy.ts  (top-level, next to app/)
import { clerkMiddleware, createRouteMatcher } from "@clerk/nextjs/server";

const isProtectedRoute = createRouteMatcher(["/dashboard(.*)"]);

export default clerkMiddleware(async (auth, req) => {
  if (isProtectedRoute(req)) {
    await auth.protect();
  }
});
```

`auth.protect()` redirects unauthenticated requests to the Clerk sign-in page automatically — no manual redirect needed at the proxy layer.

### What NOT to do

| Prohibited                                                         | Why                                                           |
| ------------------------------------------------------------------ | ------------------------------------------------------------- |
| `middleware.ts`                                                    | Deprecated in Next.js 16; use `proxy.ts`                      |
| Auth checks inside `app/dashboard/layout.tsx` as the sole guard   | Proxy protection must come first; layout checks are secondary |
| Protected pages outside `app/dashboard/`                          | All auth-gated content belongs under `/dashboard`             |
| Redirecting to a custom sign-in page                              | Use Clerk's built-in sign-in UI (see `auth.md`)               |

## Secondary Guard (Server Components)

The proxy enforces access at the network edge, but individual pages and Server Actions should still verify the session when they need the user's identity:

```ts
// app/dashboard/some-feature/page.tsx
import { auth } from "@clerk/nextjs/server";
import { redirect } from "next/navigation";

export default async function Page() {
  const { userId } = await auth();
  if (!userId) redirect("/sign-in"); // belt-and-suspenders
  // ...
}
```

This is a secondary check — never the primary one. Do not skip the proxy-level protection and rely solely on per-page redirects.

## Adding New Routes

1. Create the page under `app/dashboard/`.
2. The proxy matcher `/dashboard(.*)` already covers it — no changes to `proxy.ts` are needed.
3. Add a secondary `auth()` guard in the page if the page reads user-specific data.

## Summary

| Concern                  | Solution                                                 |
| ------------------------ | -------------------------------------------------------- |
| App entry point          | `/` — public, no auth required                           |
| All authenticated UI     | `/dashboard` and sub-routes only                         |
| Primary route protection | `clerkMiddleware` + `createRouteMatcher` in `proxy.ts`   |
| Secondary identity check | `const { userId } = await auth()` inside the page/action |
| Unauthenticated redirect | Handled automatically by `auth.protect()` in proxy       |
