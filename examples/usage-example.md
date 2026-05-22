# Example: Using the CLAUDE.md Template

This example shows what a project using this CLAUDE.md template looks like.

## Project: TaskFlow SaaS

A task management SaaS built with Next.js 15 + SQLite.

### File Structure

```
src/
鈹溾攢鈹€ app/
鈹?  鈹溾攢鈹€ (auth)/
鈹?  鈹?  鈹溾攢鈹€ login/page.tsx
鈹?  鈹?  鈹斺攢鈹€ register/page.tsx
鈹?  鈹溾攢鈹€ dashboard/
鈹?  鈹?  鈹溾攢鈹€ page.tsx          # Server Component - fetches tasks
鈹?  鈹?  鈹斺攢鈹€ loading.tsx       # Auto suspense boundary
鈹?  鈹溾攢鈹€ api/
鈹?  鈹?  鈹溾攢鈹€ auth/[...nextauth]/route.ts
鈹?  鈹?  鈹斺攢鈹€ webhooks/stripe/route.ts
鈹?  鈹溾攢鈹€ layout.tsx
鈹?  鈹斺攢鈹€ page.tsx
鈹溾攢鈹€ components/
鈹?  鈹溾攢鈹€ ui/                   # shadcn/ui primitives
鈹?  鈹溾攢鈹€ forms/
鈹?  鈹?  鈹斺攢鈹€ TaskForm.tsx      # Client Component with Server Action
鈹?  鈹斺攢鈹€ shared/
鈹?      鈹斺攢鈹€ Pagination.tsx
鈹溾攢鈹€ lib/
鈹?  鈹溾攢鈹€ db/
鈹?  鈹?  鈹溾攢鈹€ schema.ts         # Drizzle schema
鈹?  鈹?  鈹溾攢鈹€ migrations/
鈹?  鈹?  鈹斺攢鈹€ index.ts          # DB connection singleton
鈹?  鈹溾攢鈹€ auth.ts               # NextAuth config
鈹?  鈹斺攢鈹€ utils.ts              # cn(), formatDate()
鈹溾攢鈹€ hooks/
鈹?  鈹斺攢鈹€ usePagination.ts
鈹溾攢鈹€ types/
鈹?  鈹斺攢鈹€ index.ts
鈹斺攢鈹€ middleware.ts
```

### Example: Server Component (default)

```tsx
// app/dashboard/page.tsx
export default async function DashboardPage() {
  const user = await getCurrentUser();
  const tasks = await db.query.tasks.findMany({
    where: eq(tasks.user_id, user.id),
    orderBy: desc(tasks.created_at),
  });
  return <TaskList tasks={tasks} />;
}
```

### Example: Client Component (form with Server Action)

```tsx
// app/dashboard/form.tsx
"use client";
export function TaskForm() {
  const [state, formAction] = useActionState(createTask, null);
  return <form action={formAction}>...</form>;
}
```

### Example: Drizzle Schema

```typescript
// src/lib/db/schema.ts
export const tasks = sqliteTable("tasks", {
  id: text("id").primaryKey().$defaultFn(() => crypto.randomUUID()),
  user_id: text("user_id").notNull().references(() => users.id, { onDelete: "cascade" }),
  title: text("title").notNull(),
  created_at: integer("created_at", { mode: "timestamp" }).notNull().$defaultFn(() => new Date()),
  updated_at: integer("updated_at", { mode: "timestamp" }).notNull().$defaultFn(() => new Date()),
});
```

### What Claude Code Does With This Template

When this CLAUDE.md is present, Claude Code will:
- Default to Server Components for new pages
- Use Drizzle query builder instead of raw SQL
- Follow the naming conventions automatically
- Know the folder structure without asking
- Understand the auth flow and protect routes correctly
- Use cursor-based pagination for list queries
- Validate all API inputs with Zod
