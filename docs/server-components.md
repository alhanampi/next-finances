# Server Components

## Rules

### `params` and `searchParams` are Promises — always `await` them

In this project (Next.js 16), `params` and `searchParams` are **fully async**. They are `Promise` objects, not plain objects. You **must** `await` them before accessing any property.

```ts
// correct
export default async function WorkoutPage({
  params,
}: {
  params: Promise<{ workoutId: string }>;
}) {
  const { workoutId } = await params;
}

// forbidden — params is a Promise, not a plain object
export default async function WorkoutPage({
  params,
}: {
  params: { workoutId: string };
}) {
  const { workoutId } = params; // runtime error
}
```

The same rule applies to `searchParams`:

```ts
// correct
export default async function DashboardPage({
  searchParams,
}: {
  searchParams: Promise<{ date?: string }>;
}) {
  const { date } = await searchParams;
}
```

> **This is a Next.js 16 breaking change.** Pre-v16 patterns that destructure `params` directly without `await` will not work and must not be used.

### Page components must be `async` functions

Server Component pages must be declared as `async` so they can `await` params, auth, and data fetches.

```ts
// correct
export default async function EditWorkoutPage({ params }: { params: Promise<{ workoutId: string }> }) {
  const { workoutId } = await params;
  // ...
}

// forbidden
export default function EditWorkoutPage({ params }: { params: Promise<{ workoutId: string }> }) {
  // cannot await inside a non-async function
}
```

### Route params are always strings — parse before use

Values in `params` arrive as strings regardless of what the URL segment looks like. Convert to the required type before use, and handle invalid input.

```ts
const { workoutId } = await params;
const id = parseInt(workoutId, 10);
if (isNaN(id)) notFound();
```

Never pass a raw param string to a `/data` helper that expects a number.

### Call `notFound()` for missing resources

If a resource does not exist or does not belong to the authenticated user, call `notFound()` from `next/navigation`. Do not redirect to another page or return an empty UI.

```ts
import { notFound } from "next/navigation";

const workout = await getWorkoutById(userId, id);
if (!workout) notFound();
```

### Authenticate before fetching data

Check auth at the top of every page Server Component before accessing params or fetching data. See `docs/auth.md` for the full pattern.

```ts
const { userId } = await auth();
if (!userId) redirect("/sign-in");

const { workoutId } = await params;
```

## Explicitly Prohibited

| Prohibited | Why |
| --- | --- |
| Destructuring `params` without `await` | `params` is a `Promise` in Next.js 16 — accessing properties directly is a runtime error |
| Typing `params` as `{ key: string }` instead of `Promise<{ key: string }>` | Incorrect type causes TypeScript to allow the broken pattern |
| Passing raw string params to helpers expecting a number | Type mismatch leads to incorrect queries or runtime errors |
| Using `redirect()` instead of `notFound()` for missing resources | `notFound()` is the correct semantic signal for a resource that doesn't exist |
| Fetching data before authenticating | An unauthenticated caller must never reach a `/data` helper |

## Summary

| Concern | Requirement |
| --- | --- |
| `params` type | `Promise<{ key: string }>` |
| Accessing params | `const { key } = await params` |
| `searchParams` type | `Promise<{ key?: string }>` |
| Accessing searchParams | `const { key } = await searchParams` |
| Numeric route params | Parse with `parseInt`, call `notFound()` if `isNaN` |
| Missing resources | `notFound()` from `next/navigation` |
| Auth order | Authenticate before accessing params or fetching data |
