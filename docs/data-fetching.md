# Data Fetching

## Rules

### All data fetching must happen in Server Components

Data **must only** be fetched inside React Server Components. Do not fetch data in:

- Client Components (`"use client"`)
- Route Handlers (`app/api/*/route.ts`)
- Middleware
- Any other mechanism

If a Client Component needs data, fetch it in a Server Component ancestor and pass it down as props.

### Database queries must go through `/data` helper functions

Never query the database directly inside a component. All database access must go through helper functions in the `/data` directory.

```
/data
  workouts.ts      ← helper functions for workout queries
  exercises.ts     ← helper functions for exercise queries
  ...
```

A Server Component calls a helper; the helper queries the database using Drizzle ORM and returns typed results.

### Use Drizzle ORM — no raw SQL

All queries inside `/data` helpers must use Drizzle ORM. Do not use raw SQL strings (`db.execute(sql`...`)`, `db.run(...)`, template literal SQL, etc.).

```ts
// correct
const workouts = await db
  .select()
  .from(workoutsTable)
  .where(eq(workoutsTable.userId, userId));

// forbidden
const workouts = await db.execute(
  sql`SELECT * FROM workouts WHERE user_id = ${userId}`,
);
```

### Users can only access their own data

Every `/data` helper that returns user-owned data **must** filter by the authenticated user's ID. Never return rows without scoping them to the current user.

```ts
// correct — always scope to the authenticated user
export async function getWorkouts(userId: string) {
  return db
    .select()
    .from(workoutsTable)
    .where(eq(workoutsTable.userId, userId));
}

// forbidden — returns every user's data
export async function getWorkouts() {
  return db.select().from(workoutsTable);
}
```

The `userId` must come from the server-side session (e.g. via `auth()` from your auth library), never from a URL parameter, query string, or request body supplied by the client.

```ts
// correct — userId sourced from the server session
const { userId } = await auth();
const workouts = await getWorkouts(userId);

// forbidden — userId sourced from untrusted client input
const workouts = await getWorkouts(params.userId);
```

## Explicitly Prohibited

The following patterns are **never** acceptable in this codebase:

| Prohibited                                                            | Why                                                                                                                   |
| --------------------------------------------------------------------- | --------------------------------------------------------------------------------------------------------------------- |
| `fetch()` or data access inside a `"use client"` component            | Client Components run in the browser — DB access is impossible and API calls bypass server-side auth checks           |
| `fetch()` or DB queries inside a Route Handler (`app/api/*/route.ts`) | Route Handlers are an unnecessary indirection; use Server Components directly                                         |
| Inline DB queries inside a component (even a Server Component)        | All DB access must be encapsulated in `/data` helpers for consistency and auditability                                |
| Raw SQL — `db.execute(sql\`...\`)`, `db.run()`, template literal SQL  | Bypasses Drizzle's type safety and query builder                                                                      |
| Accepting `userId` (or any row identifier) from client input          | URL params, query strings, and request bodies are attacker-controlled; always derive `userId` from the server session |
| Queries without a `userId` filter on user-owned tables                | Would expose every user's data to whoever calls the helper                                                            |
| Calling `/data` helpers without first authenticating                  | A helper must never be reachable by an unauthenticated caller                                                         |

## Summary

| Rule                | Requirement                                                         |
| ------------------- | ------------------------------------------------------------------- |
| Where to fetch      | Server Components only                                              |
| How to query the DB | `/data` helper functions only                                       |
| ORM                 | Drizzle ORM — no raw SQL                                            |
| Data scoping        | Always filter by the authenticated `userId` from the server session |
