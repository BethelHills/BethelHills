### Polling App — Project Rules for AI (Cursor)

These rules guide AI-assisted scaffolding and refactors for our Polling App. Follow them strictly when generating, moving, or editing code.

### Golden Rules
- **TypeScript-first**: All code is TypeScript. Export explicit types for public APIs. Use `zod` schemas as the single source of truth and derive TS types via `z.infer`.
- **App Router**: Use Next.js App Router under `app/`. Prefer Server Components by default; use Client Components for interactive UI (forms, toasts, client Supabase).
- **Form stack**: Use `react-hook-form` with `@hookform/resolvers/zod`. UI components come from `shadcn/ui` (`Button`, `Input`, `Form`, `FormField`, `Textarea`, `Switch`, etc.).
- **Supabase**: Use Supabase for auth and database. Never expose the service role key client-side. Use server-client split (`lib/supabase/server.ts` and `lib/supabase/client.ts`).
- **Validation at the edge**: Validate inputs with `zod` both in UI (client) and API/server actions (server). Reject invalid data early. Return typed, structured errors.
- **Client/server boundaries**: Never import server-only modules into Client Components. Route handlers and server utilities must stay on the server.
- **File placement contract**: Place files according to structure below. Do not create ad-hoc folders or duplicate patterns.
- **Accessible UI**: Always provide `label` associations and `aria-*` where relevant. Use `FormMessage` for errors.

### Folder Structure Contract
```text
app/
  layout.tsx
  page.tsx                     # Landing or redirect to /polls
  polls/
    page.tsx                   # List polls
    new/
      page.tsx                 # New poll page (renders PollForm)
    [pollId]/
      page.tsx                 # Poll detail (results + actions)
      vote/
        page.tsx               # Optional dedicated vote page or embed VoteForm
  api/
    polls/
      route.ts                 # GET (list), POST (create)
    polls/[pollId]/
      route.ts                 # GET (detail), DELETE (owner only)
    polls/[pollId]/vote/
      route.ts                 # POST (vote)

components/
  polls/
    PollForm.tsx
    VoteForm.tsx
    PollCard.tsx
    PollList.tsx
  ui/                          # shadcn/ui components

lib/
  supabase/
    server.ts                  # createServerClient (cookies-based)
    client.ts                  # createBrowserClient
  validation/
    poll.ts                    # zod schemas for poll create/vote
  types.ts                     # shared domain types (derived from zod)
  utils.ts

styles/
  globals.css
```

Environment variables required (configure in `.env.local`, never hardcode):
- `NEXT_PUBLIC_SUPABASE_URL`
- `NEXT_PUBLIC_SUPABASE_ANON_KEY`

### Domain Models and Validation
- **Tables**: `polls` (id, question, user_id, created_at), `poll_options` (id, poll_id, label), `votes` (id, poll_id, option_id, user_id/null, created_at, fingerprint optional).
- **IDs**: Use UUID strings. Timestamps as ISO strings in types, `timestamptz` in DB.
- **Schemas**:
  - `newPollSchema`: `{ question: string (3–200), options: string[] (2–10 unique, 1–80 chars) }`
  - `voteSchema`: `{ pollId: uuid, optionId: uuid }`
  - Derive TS types via `z.infer<typeof newPollSchema>` and export from `lib/types.ts`.

### Supabase Integration Rules
- **Server (route handlers / server actions)**
  - Create client via `createServerClient` in `lib/supabase/server.ts` using `cookies()`.
  - Read session on the server for privileged actions (create/delete poll). Return 401 if unauthenticated where required.
  - Prefer row-level security (RLS) to enforce ownership.
- **Client (interactive components)**
  - Use `createBrowserClient` from `lib/supabase/client.ts` for client-only interactions (e.g., optimistic UI, subscriptions if added).
  - Do not import `server.ts` into client components.
- **Security**
  - Never use service role key in the browser. Restrict all writes via RLS + auth.
  - Check `auth.getUser()` server-side before inserting owner-scoped data.

### Forms & UI Rules (react-hook-form + shadcn/ui)
- Always use `react-hook-form` with `zodResolver(schema)`.
- Use shadcn `<Form>` wrapper and `<FormField>` with `control` and `name`.
- Display errors via `<FormMessage>`. Disable submit while pending. Provide success/failure feedback (e.g., toast).
- Keep components small: `PollForm` handles create; `VoteForm` handles vote. Lift data fetching to the parent page or a server component.

### API & Server Actions Rules
- **API route handlers** in `app/api/.../route.ts`:
  - Validate `request.json()` via `zod`.
  - Return `NextResponse.json({ data, error }, { status })` with accurate status codes: 200/201, 400, 401, 403, 404, 409, 422, 500.
  - Use server Supabase client. Never trust client-provided `user_id`.
- **Server actions** (optional): If used for form submits, still validate with `zod` and return typed results. Keep server actions colocated with pages/components only if ergonomics improve.

### Naming & Conventions
- Files: `PascalCase` for components, `kebab-case` for routes, `camelCase` for functions.
- Functions: verbs; components: nouns. Avoid 1–2 letter names. Prefer descriptive identifiers.
- Early returns and guard clauses. Avoid deep nesting. Handle errors meaningfully — never swallow exceptions.

### Scaffold Playbook — “Create a form to submit a new poll”
Use this when the AI is asked to scaffold a new poll form.

1) Create schema and types
```ts
// lib/validation/poll.ts
import { z } from "zod";

export const newPollSchema = z.object({
  question: z.string().min(3).max(200),
  options: z.array(z.string().min(1).max(80)).min(2).max(10).refine(
    (opts) => new Set(opts.map((o) => o.trim().toLowerCase())).size === opts.length,
    { message: "Options must be unique" }
  )
});

export type NewPollInput = z.infer<typeof newPollSchema>;
```

2) Build UI component
```tsx
// components/polls/PollForm.tsx (Client Component)
"use client";
import { useForm } from "react-hook-form";
import { zodResolver } from "@hookform/resolvers/zod";
import { newPollSchema, type NewPollInput } from "@/lib/validation/poll";
import { Form, FormField, FormItem, FormLabel, FormControl, FormMessage } from "@/components/ui/form";
import { Button } from "@/components/ui/button";
import { Input } from "@/components/ui/input";
import { useTransition, useState } from "react";

export function PollForm({ onCreated }: { onCreated?: (pollId: string) => void }) {
  const form = useForm<NewPollInput>({ resolver: zodResolver(newPollSchema), defaultValues: { question: "", options: ["", ""] } });
  const [isPending, startTransition] = useTransition();

  async function onSubmit(values: NewPollInput) {
    const res = await fetch("/api/polls", { method: "POST", body: JSON.stringify(values) });
    if (!res.ok) {
      const { error } = await res.json();
      form.setError("question", { message: error ?? "Failed to create poll" });
      return;
    }
    const { data } = await res.json();
    onCreated?.(data.id);
  }

  return (
    <Form {...form}>
      <form onSubmit={form.handleSubmit((v) => startTransition(() => onSubmit(v)))} className="space-y-4">
        <FormField name="question" control={form.control} render={({ field }) => (
          <FormItem>
            <FormLabel>Question</FormLabel>
            <FormControl><Input placeholder="What should we do?" {...field} /></FormControl>
            <FormMessage />
          </FormItem>
        )} />
        {/* Render dynamic option inputs (at least two) */}
        <Button type="submit" disabled={isPending}>Create Poll</Button>
      </form>
    </Form>
  );
}
```

3) Add API route
```ts
// app/api/polls/route.ts
import { NextResponse } from "next/server";
import { newPollSchema } from "@/lib/validation/poll";
import { createServerClient } from "@/lib/supabase/server";

export async function POST(req: Request) {
  const supabase = createServerClient();
  const {
    data: { user }
  } = await supabase.auth.getUser();
  if (!user) return NextResponse.json({ error: "Unauthorized" }, { status: 401 });

  const json = await req.json();
  const parsed = newPollSchema.safeParse(json);
  if (!parsed.success) {
    return NextResponse.json({ error: parsed.error.flatten() }, { status: 422 });
  }

  const { question, options } = parsed.data;
  const { data, error } = await supabase.rpc("create_poll_with_options", { p_question: question, p_options: options });
  if (error) return NextResponse.json({ error: error.message }, { status: 500 });
  return NextResponse.json({ data }, { status: 201 });
}
```

4) Use the form on `app/polls/new/page.tsx`, redirect to `app/polls/[pollId]` on success.

### Refactor Playbook — “Move voting into a dedicated VoteForm”
- Extract vote UI to `components/polls/VoteForm.tsx` (client component).
- Keep data fetching (poll and options) server-side in `app/polls/[pollId]/page.tsx`.
- Submit to `POST /api/polls/[pollId]/vote` with `voteSchema`.
- Ensure optimistic UI only updates after server confirms or reconcile on error.

### Review Checklist for AI Outputs
- **Placement**: Files are placed exactly per the Folder Structure Contract.
- **Validation**: Inputs use `zod` with resolver on client and parse on server.
- **Supabase**: No server modules in client; no service role exposed; `auth.getUser()` checked.
- **Accessibility**: Labels, messages, and focus management present.
- **Types**: All exports typed; schemas derive types; no `any` in public APIs.
- **HTTP**: Correct status codes and structured JSON response.

### How to Update These Rules
- When AI-generated code violates a rule, add a short rule or tweak wording here to prevent the pattern from recurring.
- Prefer adding examples near the relevant rule (small, minimal snippets).
- Keep this file succinct; link to source files via paths (e.g., `app/polls/new/page.tsx`).

### Example Prompts for AI
- **Scaffold new poll form**: “Create a form to submit a new poll using `react-hook-form` + `zod` and `shadcn/ui`. Place UI in `components/polls/PollForm.tsx`, page at `app/polls/new/page.tsx`, schema in `lib/validation/poll.ts`, and API handler at `app/api/polls/route.ts` with Supabase server client auth and proper status codes.”
- **Add delete poll endpoint**: “Add `DELETE /api/polls/[pollId]` that verifies auth server-side with Supabase, enforces owner-only delete (RLS), returns 204 on success, 401/403/404 appropriately. Do not import server code into client.”
- **Add voting UI**: “Create `components/polls/VoteForm.tsx` to submit to `POST /api/polls/[pollId]/vote`. Validate with `voteSchema` on client and server. Do not expose `user_id` in the request; read from server session.”
- **Refactor to App Router pattern**: “If any route is under `pages/api`, migrate it to `app/api/.../route.ts` and align responses to `{ data, error }` JSON shape with accurate status codes.”

### Validation Checklist (quick)
- zod schemas exist in `lib/validation/*` and are imported on both client and server.
- Client forms use `zodResolver(schema)`; server parses via `schema.safeParse()`.
- No `any` types in public exports; types derived via `z.infer`.
- All write operations check auth on the server (`auth.getUser()`).
- API responses use `NextResponse.json({ data, error }, { status })` with correct status codes.

### Observations and Rule Tweaks
- Found cases where routes were placed under `pages/api`. Fixed and clarified: All APIs must live in `app/api/.../route.ts`.
- Occasionally the service role key was referenced in client code. Strengthened rule: Never expose service role keys; only use anon on client.
- Client components imported `lib/supabase/server`. Clarified split and added explicit ban.
- Missing form error messages and labels. Added Accessibility rule and checklist reminder.
- Type gaps with `any` in shared APIs. Reinforced TypeScript-first rule and zod-derived types.

