# Project Context

## Stack & Versions

- **Next.js 15** (App Router, not Pages Router)
- **React 19** (Server Components by default)
- **TypeScript** — strict mode, no `any`
- **SQLite** via `better-sqlite3` (local) or Turso (production)
- **Tailwind CSS v4** + **shadcn/ui** for components
- **next-auth v5** for authentication
- **Zod** for all validation (API routes, forms, env vars)
- **drizzle-orm** for database schema & migrations

## Folder Structure

```
src/
├── app/                    # App Router pages & API routes
│   ├── (auth)/             # Auth group (login, register) — no layout wrapper
│   ├── dashboard/          # Protected dashboard pages
│   ├── api/                # API route handlers
│   │   ├── auth/[...nextauth]/  # NextAuth config
│   │   └── webhooks/       # External webhooks (Stripe, etc.)
│   ├── layout.tsx          # Root layout
│   └── page.tsx            # Landing page (public)
├── components/
│   ├── ui/                 # shadcn/ui primitives (don't modify)
│   ├── forms/              # Form components with Zod validation
│   └── shared/             # Reusable non-UI components (pagination, empty states)
├── lib/
│   ├── db/                 # Database
│   │   ├── schema.ts       # Drizzle schema definitions
│   │   ├── migrations/     # Generated migration files (don't edit by hand)
│   │   └── index.ts        # DB connection singleton
│   ├── auth.ts             # NextAuth configuration
│   ├── stripe.ts           # Stripe client initialization
│   └── utils.ts            # Shared utilities (cn, formatDate, etc.)
├── hooks/                  # Custom React hooks
├── types/                  # Shared TypeScript types
└── middleware.ts           # Next.js middleware (auth guards)
```

## Naming Conventions

- **Files**: `kebab-case.tsx` for components, `kebab-case.ts` for utilities
- **Components**: `PascalCase` — `UserProfile.tsx`, not `user-profile-component.tsx`
- **Hooks**: `use` prefix — `useAuth.ts`, `usePagination.ts`
- **API routes**: RESTful — `GET /api/users`, `POST /api/users`, not `/api/getUsers`
- **DB tables**: `snake_case` plural — `user_profiles`, `subscription_plans`
- **Constants**: `UPPER_SNAKE_CASE` — `MAX_RETRY_COUNT`, `DEFAULT_PAGE_SIZE`
- **Types**: `PascalCase` — `UserProfile`, `ApiResponse`, no `I` prefix

## Database Rules

### Schema
- Define all tables in `src/lib/db/schema.ts` using Drizzle
- Every table gets: `id` (text, cuid via `crypto.randomUUID()`), `created_at`, `updated_at`
- Use `text()` for IDs, not `serial()` — SQLite doesn't do auto-increment well
- Foreign keys: `user_id` references `users.id`, always `onDelete: "cascade"`

### Migrations
- **Never edit migration files by hand.** Always change schema.ts, then run:
  ```bash
  npx drizzle-kit generate
  npx drizzle-kit push   # local dev
  ```
- For production (Turso):
  ```bash
  npx drizzle-kit generate
  npx drizzle-kit migrate
  ```
- Migration files go in `src/lib/db/migrations/` — commit them to git
- If you need to modify an existing column, create a new migration. Don't modify old ones.

### Queries
- Use Drizzle query builder, not raw SQL (except for complex aggregations)
- Always use parameterized queries — never string-interpolate user input
- For pagination: cursor-based, not offset. Use `created_at + id` as cursor key.

## Component Patterns

### Server Components (default)
- Use by default. Fetch data directly in the component.
- Pattern:
  ```tsx
  // app/dashboard/page.tsx
  export default async function DashboardPage() {
    const user = await getCurrentUser();  // throws if not authenticated
    const projects = await db.query.projects.findMany({ where: ... });
    return <ProjectList projects={projects} />;
  }
  ```
- No `useEffect` for data fetching. No `useState` for server data.

### Client Components
- Only when you need: interactivity, browser APIs, React state, effects
- Mark with `"use client"` at the top
- Keep them small and leaf-shaped. Push data fetching up to Server Components.

### Forms
- Use Server Actions for mutations (not API routes for form submissions)
- Pattern:
  ```tsx
  // app/settings/form.tsx
  "use client";
  export function SettingsForm() {
    const [state, formAction] = useActionState(updateSettings, null);
    return <form action={formAction}>...</form>;
  }
  ```
- Validate with Zod in the Server Action, not in the component
- Return structured errors: `{ field: "email", message: "Invalid email" }[]`

### Loading & Error States
- Every route folder can have `loading.tsx` (auto suspense boundary)
- Every route folder can have `error.tsx` (must be client component)
- For inline loading: `Suspense` wrapper with skeleton component
- For mutations: optimistic updates via `useOptimistic` or loading buttons

## API Routes

- Use for: webhooks, third-party callbacks, SSE streams
- For internal CRUD: use Server Actions instead
- Always validate input with Zod before processing
- Return consistent error shape:
  ```ts
  { error: { message: string; code: string; status: number } }
  ```
- Use NextResponse, not raw Response

## Authentication

- NextAuth v5 with credentials provider
- Session strategy: JWT (not database sessions)
- Protect routes in `middleware.ts`:
  ```ts
  export default auth((req) => {
    if (!req.auth && req.nextUrl.pathname.startsWith("/dashboard")) {
      return NextResponse.redirect(new URL("/login", req.nextUrl));
    }
  });
  ```
- Get server-side user: `const user = await getCurrentUser()` (throws if not auth)
- Get client-side user: `const { data: session } = useSession()`

## Environment Variables

- Define all env vars in `.env.local` (gitignored)
- Validate at startup in `src/lib/env.ts` using Zod:
  ```typescript
  export const env = z.object({
    DATABASE_URL: z.string().url(),
    NEXTAUTH_SECRET: z.string().min(32),
    STRIPE_SECRET_KEY: z.string().startsWith("sk_"),
  }).parse(process.env);
  ```
- Access via `env.DATABASE_KEY`, not `process.env.DATABASE_URL`

## What We Don't Do (and Why)

- **No `any` types.** If you can't type it, use `unknown` and narrow. `any` defeats the purpose of TypeScript.
- **No `useEffect` for data fetching.** This is a Next.js 15 project. Fetch in Server Components.
- **No inline styles.** Use Tailwind classes. If you need dynamic values, use `cn()` helper.
- **No `console.log` in production code.** Use a proper logger. For dev, use `console.error` at minimum.
- **No editing shadcn/ui source files directly.** If you need to customize, copy the component to `components/shared/` and modify there.
- **No offset-based pagination at scale.** Cursor-based performs better and handles concurrent inserts.
- **No raw SQL without parameterized queries.** SQL injection is not an option.
- **No client-side auth checks as the only guard.** Always verify server-side too. Client-side checks are for UX, not security.
- **No committing `.env` files.** Ever. Not even `.env.example` with real secrets.
- **No `ts-ignore`.** Use `@ts-expect-error` with a comment explaining why.

## Dev Commands

```bash
npm run dev          # Start dev server (Turbopack)
npm run build        # Production build
npm run lint         # ESLint + TypeScript check
npm run db:generate  # Generate Drizzle migrations
npm run db:push      # Push schema to local DB
npm run db:studio    # Open Drizzle Studio (local DB viewer)
```
