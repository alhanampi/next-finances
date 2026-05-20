# Data Mutations

## Rules

### All mutations must go through `/data` helper functions

Mutations **must only** be performed inside helper functions in the `/data` directory. Do not write Drizzle insert/update/delete calls anywhere else — not in Server Actions, not in components, not in Route Handlers.

```
/data
  workouts.ts      ← read AND write helpers for workouts
  exercises.ts     ← read AND write helpers for exercises
  ...
```

A Server Action calls a `/data` helper; the helper performs the mutation using Drizzle ORM and returns a typed result. The Server Action is responsible for auth, calling the helper, and revalidating the cache.

### Use Drizzle ORM — no raw SQL

All mutations inside `/data` helpers must use Drizzle's `.insert()`, `.update()`, and `.delete()` builders. Do not use raw SQL strings.

```ts
// correct
await db.insert(workouts).values({ userId, name, startedAt: new Date() });

// correct
await db
  .update(workouts)
  .set({ name })
  .where(and(eq(workouts.id, id), eq(workouts.userId, userId)));

// correct
await db
  .delete(workouts)
  .where(and(eq(workouts.id, id), eq(workouts.userId, userId)));

// forbidden
await db.execute(sql`INSERT INTO workouts ...`);
```

### Always scope mutations to the authenticated user

Every `/data` mutation helper that touches user-owned rows **must** include a `userId` filter in its WHERE clause. An update or delete without a `userId` guard would let one user modify another user's data.

```ts
// correct — WHERE scopes to both the row id AND the owner
export async function deleteWorkout(userId: string, workoutId: number) {
  await db
    .delete(workouts)
    .where(and(eq(workouts.id, workoutId), eq(workouts.userId, userId)));
}

// forbidden — anyone who knows the id can delete any row
export async function deleteWorkout(workoutId: number) {
  await db.delete(workouts).where(eq(workouts.id, workoutId));
}
```

### `userId` must come from the server session, never from client input

The `userId` passed into mutation helpers must be derived from `await auth()` inside the Server Action. Never accept it from form data, URL params, or any other client-supplied source.

```ts
// app/dashboard/actions.ts — correct
"use server";
import { auth } from "@clerk/nextjs/server";
import { deleteWorkout } from "@/data/workouts";

export async function deleteWorkoutAction(workoutId: number) {
  const { userId } = await auth();
  if (!userId) throw new Error("Unauthenticated");
  await deleteWorkout(userId, workoutId);
}

// forbidden — userId comes from untrusted form data
export async function deleteWorkoutAction(formData: FormData) {
  const userId = formData.get("userId") as string; // attacker-controlled
  await deleteWorkout(userId, Number(formData.get("workoutId")));
}
```

### Server Action params must be explicitly typed — no `FormData`

Server Action parameters must use explicit TypeScript types. Do not use `FormData` as a parameter type — it is untyped, requires manual casting, and bypasses static analysis.

```ts
// correct — explicit TypeScript types
export async function createWorkoutAction(name: string, startedAt: Date) { ... }

// forbidden — FormData is untyped
export async function createWorkoutAction(formData: FormData) { ... }
```

### Validate all Server Action arguments with Zod

Every Server Action must validate its arguments with Zod before doing anything else. Never trust that the caller passed valid data — validation must happen inside the action.

Define a Zod schema alongside each action and use `.parse()` to validate. Use `z.infer<>` to derive the TypeScript param type from the schema so the type and the validation rule stay in sync.

```ts
// app/dashboard/actions.ts
"use server";

import { z } from "zod";
import { auth } from "@clerk/nextjs/server";
import { revalidatePath } from "next/cache";
import { createWorkout } from "@/data/workouts";

const createWorkoutSchema = z.object({
  name: z.string().min(1).max(255),
});

export async function createWorkoutAction(
  params: z.infer<typeof createWorkoutSchema>,
) {
  const { name } = createWorkoutSchema.parse(params);

  const { userId } = await auth();
  if (!userId) throw new Error("Unauthenticated");

  await createWorkout(userId, name);
  revalidatePath("/dashboard");
}
```

### Server Actions must handle errors and return a result object

Every Server Action must wrap its `/data` helper call in a `try/catch` and return a typed result object. Never let DB errors propagate uncaught — the user must always receive actionable feedback.

**Return type:** `Promise<{ error: string } | void>` — `void` signals success; the calling Client Component is responsible for any navigation.

**Pattern:**

```ts
export async function createWorkoutAction(
  params: z.infer<typeof createWorkoutSchema>,
): Promise<{ error: string } | void> {
  const { name } = createWorkoutSchema.parse(params);

  const { userId } = await auth();
  if (!userId) throw new Error("Unauthenticated");

  try {
    await createWorkout(userId, name);
    revalidatePath("/dashboard");
  } catch (error) {
    return { error: "Failed to create workout. Please try again." };
  }
}
```

**Key rules:**
- Return `{ error: string }` for caught errors. Use a user-friendly message, not the raw exception message.
- The calling Client Component must check `result?.error`, display it via a shadcn `Alert`, and handle any navigation. Silent failures are not acceptable.

```tsx
// in the Client Component
startTransition(async () => {
  const result = await createWorkoutAction({ name });
  if (result?.error) {
    setError(result.error);
  } else {
    router.push("/dashboard");
  }
});
```

### Never call `redirect()` inside a Server Action

`redirect()` in Next.js works by throwing a special internal exception. Calling it inside a Server Action means it can be silently swallowed by a `try/catch`, produces confusing control flow, and makes the action harder to test. Navigation is a client concern — the Server Action should not decide where the user goes next.

**Rule:** Server Actions must never call `redirect()`. Return a result object instead and let the calling Client Component drive navigation with `router.push()` or `router.replace()`.

```ts
// forbidden — redirect() inside a Server Action
export async function createWorkoutAction(params: ...) {
  await createWorkout(userId, name);
  redirect("/dashboard"); // do not do this
}

// correct — action returns, client navigates
export async function createWorkoutAction(params: ...): Promise<{ error: string } | void> {
  try {
    await createWorkout(userId, name);
    revalidatePath("/dashboard");
  } catch (error) {
    return { error: "Failed to create workout. Please try again." };
  }
}
```

```tsx
// Client Component handles navigation
const result = await createWorkoutAction(params);
if (result?.error) {
  setError(result.error);
} else {
  router.push("/dashboard");
}
```

### Mutations are invoked via Server Actions in colocated `actions.ts` files

Server Actions are the only mechanism for triggering mutations from the client. Do not create Route Handlers (`app/api/*/route.ts`) for mutation purposes.

Every Server Action must live in an `actions.ts` file colocated with the route segment that uses it — not in a shared file, not inline in a component.

```
app/
  dashboard/
    page.tsx
    actions.ts        ← Server Actions for the dashboard route
    _components/
      ...
  workouts/
    [id]/
      page.tsx
      actions.ts      ← Server Actions for the workout detail route
```

Each `actions.ts` file must have `"use server"` at the top of the file (not per-function).

A Server Action must, in this order:
1. Validate arguments with Zod — reject invalid input immediately.
2. Call `await auth()` and verify `userId` is non-null.
3. Call the appropriate `/data` helper, passing `userId` from the session.
4. Call `revalidatePath()` or `updateTag()` after the mutation so the UI reflects the change.

```ts
// app/dashboard/actions.ts
"use server";

import { z } from "zod";
import { auth } from "@clerk/nextjs/server";
import { revalidatePath } from "next/cache";
import { createWorkout } from "@/data/workouts";

const createWorkoutSchema = z.object({
  name: z.string().min(1).max(255),
});

export async function createWorkoutAction(
  params: z.infer<typeof createWorkoutSchema>,
) {
  const { name } = createWorkoutSchema.parse(params);

  const { userId } = await auth();
  if (!userId) throw new Error("Unauthenticated");

  await createWorkout(userId, name);
  revalidatePath("/dashboard");
}
```

### `/data` helpers must not contain auth logic

Auth belongs in the Server Action, not in the `/data` helper. Helpers receive `userId` as a parameter and trust it. This keeps helpers composable and testable without mocking Clerk.

```ts
// correct — helper trusts the userId it receives
export async function createWorkout(userId: string, name: string) {
  await db.insert(workouts).values({ userId, name, startedAt: new Date() });
}

// forbidden — helper reaching out to auth() itself
export async function createWorkout(name: string) {
  const { userId } = await auth(); // wrong layer
  await db.insert(workouts).values({ userId, name, startedAt: new Date() });
}
```

## Explicitly Prohibited

| Prohibited | Why |
| ---------- | --- |
| Inline Drizzle calls inside a Server Action | All DB access must go through `/data` helpers |
| Inline Drizzle calls inside a component | Same as above |
| Raw SQL — `db.execute(sql\`...\`)` | Bypasses Drizzle's type safety |
| Update/delete without a `userId` WHERE clause on user-owned tables | Would allow one user to modify another user's data |
| Accepting `userId` from form data, URL params, or request body | Client input is attacker-controlled; always derive from `await auth()` |
| `FormData` as a Server Action parameter type | Untyped; use explicit TypeScript types instead |
| Server Action arguments that are not validated with Zod | Arguments must be validated before auth or DB access |
| Route Handlers for mutations | Use Server Actions in `actions.ts` instead |
| Server Actions defined outside of an `actions.ts` file | Actions must be colocated with their route segment in `actions.ts` |
| Inline `"use server"` functions inside components | Define all Server Actions in `actions.ts` files |
| Calling a mutation helper before verifying `userId` is non-null | A helper must never be reachable by an unauthenticated caller |
| Auth logic (`auth()` calls) inside a `/data` helper | Auth belongs in the Server Action layer |
| Unhandled DB errors in a Server Action | Wrap `/data` calls in `try/catch`; return `{ error: string }` so the UI can display feedback |
| Displaying raw exception messages to users | Use a fixed, user-friendly string; log or ignore the raw error |
| `redirect()` inside a Server Action | `redirect()` throws internally and conflicts with try/catch; return a result and let the Client Component navigate |

## Summary

| Rule | Requirement |
| ---- | ----------- |
| Where to mutate | `/data` helper functions only |
| ORM | Drizzle ORM — no raw SQL |
| Data scoping | Always filter by `userId` in update/delete WHERE clauses |
| `userId` source | `await auth()` in the Server Action — never client input |
| Invocation mechanism | Server Actions in colocated `actions.ts` files only |
| Param types | Explicit TypeScript types — no `FormData` |
| Argument validation | Zod `.parse()` — first thing in every Server Action |
| After mutation | Call `revalidatePath()` or `updateTag()` in the Server Action |
| Auth logic | Server Action layer only — `/data` helpers receive `userId` as a parameter |
| Error handling | Wrap `/data` calls in `try/catch`; return `{ error: string }` on failure |
| Navigation after mutation | Client Component calls `router.push()` on success — never `redirect()` inside the action |
