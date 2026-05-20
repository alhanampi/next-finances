# next-finance

A personal finance dashboard built with Next.js 16, React 19, Clerk authentication, and Drizzle ORM. Follows strict architectural standards enforced through coding guidelines and AI agent rules.

---

## Tech Stack

| Layer | Technology |
|---|---|
| Framework | Next.js 16.2.6 |
| Runtime | React 19.2.4 |
| Authentication | Clerk (`@clerk/nextjs` v7) |
| UI Components | shadcn/ui (base-sera style) |
| UI Primitives | Base UI (`@base-ui/react`) |
| Icons | Lucide React |
| Styling | Tailwind CSS v4 + tw-animate-css |
| ORM | Drizzle ORM |
| Validation | Zod |
| Language | TypeScript (strict) |
| Compiler | React Compiler (enabled) |

---

## Getting Started

### Prerequisites

- Node.js 20+
- A [Clerk](https://clerk.com) account with a project set up

### Installation

```bash
npm install
```

### Environment Variables

Create a `.env` file in the root with your Clerk credentials:

```env
NEXT_PUBLIC_CLERK_PUBLISHABLE_KEY=pk_...
CLERK_SECRET_KEY=sk_...
```

### Running

```bash
# Development
npm run dev

# Production build
npm run build
npm run start

# Lint
npm run lint
```

---

## Project Structure

```
next-finance/
├── app/
│   ├── globals.css          # Tailwind v4 + oklch theme tokens (light/dark)
│   ├── layout.tsx           # Root layout: ClerkProvider + nav + Poppins font
│   └── page.tsx             # Public landing page (/)
├── components/
│   └── ui/                  # shadcn/ui components (installed via CLI)
│       └── button.tsx
├── docs/                    # Coding standards (enforced for all contributors)
│   ├── auth.md
│   ├── data-fetching.md
│   ├── data-mutations.md
│   ├── routing.md
│   ├── server-components.md
│   └── ui.md
├── lib/
│   └── utils.ts             # cn() helper (clsx + tailwind-merge)
├── proxy.ts                 # Clerk middleware (replaces deprecated middleware.ts)
├── components.json          # shadcn/ui config
├── AGENTS.md                # AI agent instructions
└── CLAUDE.md                # Claude Code project config (references AGENTS.md)
```

---

## Architecture

### Route Structure

- `/` — Public landing page
- `/dashboard/*` — All authenticated routes (auto-protected via `proxy.ts`)

All user-facing features live under `/dashboard`. New routes added at `app/dashboard/[feature]/page.tsx` are automatically covered by the Clerk middleware matcher.

### Authentication

Route protection is handled entirely through `proxy.ts` using `clerkMiddleware()`. Server Components add a secondary `auth()` check as a belt-and-suspenders guard.

```ts
// proxy.ts — the single source of route protection
import { clerkMiddleware } from '@clerk/nextjs/server';
export default clerkMiddleware();
```

> `middleware.ts` is deprecated in this version — always use `proxy.ts`.

### Data Layer

All data access goes through `/data` helper functions. Direct DB queries in components, route handlers, or middleware are prohibited.

```
Component → Server Action (actions.ts) → /data helper → Drizzle ORM → DB
```

- **Server Actions** live in colocated `actions.ts` files, return `Promise<{ error: string } | void>`, and never call `redirect()` directly.
- **`/data` helpers** receive `userId` as a parameter — they contain no auth logic themselves.
- **userId** always comes from `await auth()` on the server — never from client input.

### Server Components (Next.js 16 specifics)

`params` and `searchParams` are **Promises** in Next.js 16 — always `await` them:

```ts
// Correct
export default async function Page({ params }: { params: Promise<{ id: string }> }) {
  const { id } = await params;
}
```

### UI

Only shadcn/ui components are used. Install new components via:

```bash
npx shadcn@latest add <component>
```

Date formatting uses `date-fns` with ordinal day + abbreviated month + full year (e.g., `"1st Sep 2025"`).

---

## Coding Standards

Full standards are documented in [`docs/`](./docs/). Summary:

| Standard | Rule |
|---|---|
| **Auth** | Clerk only — `@clerk/nextjs/server` for server, `@clerk/nextjs` for client |
| **Data fetching** | Server Components only, via `/data` helpers, Drizzle ORM |
| **Mutations** | Server Actions in `actions.ts`, Zod validation first, userId from `auth()` |
| **Routing** | All features under `/dashboard`, protected via `proxy.ts` |
| **Server Components** | Always `await params/searchParams`, `notFound()` for missing resources |
| **UI** | shadcn/ui only, date-fns for dates |

---

## Claude / AI Agent Commands

This project includes custom Claude Code slash commands in `.claude/commands/`:

### `/create-docs <filename> <description>`

Creates a new standards document at `docs/<filename>.md` describing the coding standards for a given layer of the app.

**Example:**
```
/create-docs payments "standards for Stripe integration and payment mutations"
```

### `/merge-and-create-branch <new-branch-name>`

Commits all current changes with an auto-generated commit message, merges the current branch into `main`, pushes `main` to the remote, then creates and pushes a new branch with the given name.

**Example:**
```
/merge-and-create-branch feature/transactions
```

---

## AI Agent Rules (`AGENTS.md`)

> **This is NOT the Next.js you know.**
>
> Next.js 16 has breaking changes — APIs, conventions, and file structure differ significantly from earlier versions. Any AI agent or contributor must read the relevant guide in `node_modules/next/dist/docs/` before writing code. Heed all deprecation notices.

Key breaking changes to be aware of:
- `params` and `searchParams` are Promises (always `await`)
- Middleware lives in `proxy.ts`, not `middleware.ts`
- React Compiler is enabled

---

## shadcn/ui Configuration

| Setting | Value |
|---|---|
| Style | `base-sera` |
| Base color | `taupe` |
| Icon library | Lucide |
| RSC | Enabled |
| CSS variables | Enabled |
| Tailwind CSS | `app/globals.css` |

Path aliases: `@/components`, `@/components/ui`, `@/lib`, `@/hooks`
