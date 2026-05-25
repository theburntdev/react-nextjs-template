# Frontend — Next.js (App Router)

This is the **frontend half** of a mix-and-match full-stack template library. When assembled into a real app it occupies the `frontend/` slot beside a `backend/` half (typically a dotnet template). The two halves are swappable because they meet only at a defined wire contract — resource URLs, Problem Details errors, paged envelopes — never at code. A thin root `CLAUDE.md` may sit above both halves with shared project vocabulary.

The template is **backend-agnostic in code** but **backend-present at runtime**: you develop features against a real backend you run yourself. The integration seam is the Zod wire schema + `API_URL`, so any backend half that honours the contract drops in.

The reference feature is **Todos**, backed by a backend `TodoItems` resource.

## Tech choices (decided)
- **Framework / language**: Next.js 15 (App Router) + React 19 + TypeScript (`strict`)
- **Rendering model**: **Hybrid** — Server Components own initial data fetch; Client Components own interactivity. Not RSC-purist, not client-SPA.
- **Package manager**: pnpm (run `corepack enable` once). All commands run from `frontend/`.
- **Routing**: App Router file-system routing (`app/`). No React Router.
- **Server state (client)**: TanStack Query (React Query) — the single client-side cache. Hydrated from Server Components.
- **HTTP**: native `fetch` via one isomorphic shared wrapper (no axios)
- **Validation / schemas**: Zod — validates API responses at the boundary, infers wire types
- **Forms**: react-hook-form + `@hookform/resolvers/zod`
- **Styling**: Tailwind CSS
- **UI primitives**: `next/link`, `next/image`, `next/font`, and the Metadata API are mandatory (see *Components & styling*)
- **Test runner**: Jest via `next/jest` + React Testing Library + `@testing-library/user-event`
- **E2E / Server-Component tests**: Playwright
- **Network mocking**: MSW — **test-only** (Node). No dev-runtime mocking; dev runs against a real backend.
- **Lint / format**: ESLint + Prettier

Not used: React Router / TanStack Router, axios, Storybook, Redux/Zustand for server state, MSW browser worker, Server Actions for data mutations (see *Mutations*).

## Architecture
Vertical slices. Each feature under `src/features/` is a self-contained unit owning its data layer (`api/`), TanStack Query hooks (`hooks/`), components, types, and test support. Components are built small and composable and promoted to `shared/` only when a **second** feature needs them. There is no `atoms/molecules/organisms` nesting.

App Router changes one ownership rule from a typical slice layout: **`app/` owns route composition and Server-Component data orchestration; the feature owns everything reusable below the page.** A feature no longer ships `pages/` or a `routes.tsx` — routes are declared by file location in `app/`. The route segment (`app/todos/page.tsx`) is a Server Component that prefetches data and composes the feature's Client Components.

A feature's public surface is its barrel (`index.ts`): components, hooks, services, schemas, and query keys that `app/` (or another feature) is allowed to import. Other code imports only through the barrel — never internal paths.

`shared/` holds domain-agnostic code only (primitives, the fetch wrapper, env config). Domain knowledge always stays in a feature; a reusable domain widget stays owned by its feature and is shared via its barrel, never moved to `shared/`.

`app/` is the composition root: providers, route segments, layouts, the App Router special files, and SC fetch orchestration. It imports from features; features never import from `app/`.

## Source structure (target — built at scaffold time)
```
frontend/
  src/
    app/                          # composition root + file-system routes
      layout.tsx                  # root layout (SC): <html>, fonts, <Providers/>
      providers.tsx               # "use client": QueryClientProvider (browser singleton)
      get-query-client.ts         # per-request server client / browser singleton
      page.tsx                    # "/" route (SC)
      global-error.tsx            # root render-crash fallback
      todos/                      # /todos route segment
        page.tsx                  # SC: prefetch + dehydrate + <HydrationBoundary>
        loading.tsx               # Suspense fallback for the segment
        error.tsx                 # "use client": segment error boundary
        not-found.tsx             # rendered by notFound()
        [id]/
          page.tsx                # SC: prefetch detail, hydrate
    features/
      todos/
        api/                      # PROD: data layer (framework-free, pure)
          todoService.ts          # listTodos, getTodo, createTodo, updateTodo, deleteTodo
          todoService.test.ts     # colocated, MSW-backed (Node)
          todoSchema.ts           # Zod wire schema + inferred type
          todoKeys.ts             # query-key factory
        hooks/                    # PROD: TanStack Query wrappers ("use client")
          useTodos.ts / .test.ts
          useTodo.ts
          useCreateTodo.ts
          useUpdateTodo.ts
          useDeleteTodo.ts
        components/               # PROD: flat, PascalCase folders
          TodoList/   { TodoList.tsx, TodoList.test.tsx }      # "use client"
          TodoItem/
          TodoForm/
        mocks/                    # NON-PROD: MSW handlers (tests only)
          handlers.ts             # fake /todo-items endpoints, built from builders
        testing/                  # NON-PROD: test fixtures
          todo.builder.ts
        __tests__/                # feature-level integration tests (RTL)
          todoFlow.test.tsx
        types.ts                  # view / form types (NOT wire types)
        index.ts                  # BARREL — public API
    shared/                       # domain-agnostic only
      api/
        fetchClient.ts            # isomorphic entry -> throws ProblemError
        fetchClient.server.ts     # server path ("server-only"): internal URL + header forwarding
        fetchClient.browser.ts    # browser path: public URL + credentials
        problemSchema.ts          # RFC 7807 Zod schema + ProblemDetails type
      components/                 # reusable primitives
      hooks/
      utils/
      config/
        env.client.ts             # Zod-validated NEXT_PUBLIC_* (in bundle)
        env.server.ts             # Zod-validated server vars ("server-only")
      testing/
        builders/                 # builders for shared types
    test/                         # GLOBAL test infra (test-only)
      setup.ts                    # Jest setupFiles (RTL matchers, MSW lifecycle)
      server.ts                   # MSW node server, composes feature handlers
      renderWithProviders.tsx     # the only render unit/component tests call
      test-query-client.ts        # fresh client per test, retry off
  e2e/                            # Playwright specs (SC / routing / hydration)
  middleware.ts                   # auth/route-guard SEAM (documented, unimplemented)
  instrumentation.ts              # optional (telemetry); NOT used for MSW
  .env.example
  next.config.ts                  # rewrites /api -> backend (dev)
  jest.config.ts
  playwright.config.ts
  tsconfig.json                   # "@/*" -> "src/*"
  eslint.config.js
  .prettierrc
  tailwind.config.ts
  package.json
```

## Dependency direction
```
app -> features (via barrels) -> shared
```
- `app` imports features; features never import `app`.
- Features import each other **only through barrels** (`features/x/index.ts`), never internal paths.
- `shared` imports nothing from `features` or `app`.

## Essential commands
```powershell
# Install
pnpm install

# Dev server (Next.js; rewrites /api -> backend). Assumes the backend half is running.
pnpm dev

# Production build (typecheck gates the build)
pnpm build

# Start the production build
pnpm start

# Unit / component / hook tests (Jest)
pnpm test
pnpm test:watch

# E2E (Playwright)
pnpm e2e

# Lint / format / typecheck
pnpm lint
pnpm format
pnpm typecheck
```

## The Server-Component / Client-Component boundary
**Rule: a route segment (`page.tsx`, `layout.tsx`) is a Server Component by default. `"use client"` is an explicit, deliberate opt-in for interactivity** (forms, toggles, optimistic UI, anything using hooks/state/effects).

### Standard data-flow pattern (memorize this)
Server Component fetches for first paint, hands the cache to Client Components, which read via TanStack Query. No prop drilling, no first-paint loading flash.

```tsx
// app/todos/page.tsx — Server Component
import { dehydrate, HydrationBoundary } from "@tanstack/react-query";
import { getQueryClient } from "@/app/get-query-client";
import { listTodos, todoKeys, TodoList } from "@/features/todos";

export default async function TodosPage() {
  const queryClient = getQueryClient();
  await queryClient.prefetchQuery({
    queryKey: todoKeys.list({}),
    queryFn: () => listTodos({}),
  });
  return (
    <HydrationBoundary state={dehydrate(queryClient)}>
      <TodoList />
    </HydrationBoundary>
  );
}
```

```tsx
// features/todos/components/TodoList/TodoList.tsx — Client Component
"use client";
import { useTodos } from "@/features/todos";

export function TodoList() {
  const { data } = useTodos({}); // hits hydrated cache immediately; no fetch on mount
  // ...
}
```

### Deduping server fetches without prop drilling
Within a single server render, wrap pure service reads in React's `cache()` so any Server Component in the tree calls the same function and shares one result (also dedupes native `fetch`). Use this when multiple SCs need the same data; `cache()` is **server-only** and does not cross into Client Components — the SC→CC handoff is always the prefetch + `HydrationBoundary` pattern above.

```ts
// api/todoService.ts
import { cache } from "react";
export const getTodos = cache(async (params: ListTodosParams) => { /* fetch + parse */ });
```

### Do not
- Do not pass server-fetched data down as props to read it in Client Components — hydrate it and read via `useQuery`. Props from SC are for static, non-server-state values only.
- Do not put `"use client"` on a route segment by reflex. Default is server; justify the opt-in.

## Feature slice structure
A feature owns everything for its domain *below the route*. File presence rules:
- `api/` — always: at least one `*Service.ts`, its `*.test.ts`, the Zod schema, the key factory.
- `hooks/` — one TanStack Query hook per file, colocated `.test.ts`. All `"use client"`.
- `components/` — flat, one PascalCase folder per component, colocated test. Mark `"use client"` only where interactivity requires it.
- `types.ts` — view/form types only (see *Type layering*). Omit if none.
- `mocks/` — MSW handlers. Test-only.
- `testing/` — builders / fixtures. Test-only.
- `__tests__/` — feature-level integration tests (multi-component RTL flows).
- `index.ts` — the only barrel. Exports the feature's public API: components, hooks, services, schemas, query keys. **No routes** (file-based now). **Never exports test support.**

### Data layer
Split the raw API from the TanStack Query wrappers.

`api/todoService.ts` — pure async functions. Call the fetch wrapper, validate with Zod, return parsed types. No React (except an optional `cache()` wrapper for SC dedup).
```ts
export async function listTodos(params: ListTodosParams): Promise<Page<Todo>> {
  const data = await fetchClient.get(`/todo-items`, { query: params });
  return pageSchema(todoSchema).parse(data);
}
```

`api/todoSchema.ts` — the Zod wire schema is the **single source of truth** for the wire type. Mirror the backend DTO exactly; normalize at the boundary.
```ts
export const todoSchema = z.object({
  id: z.string().uuid(),
  title: z.string(),
  status: z.enum(["Pending", "InProgress", "Done"]),
  createdAt: z.string().datetime().transform((s) => new Date(s)), // wire string -> Date
});
export type Todo = z.infer<typeof todoSchema>;
```

`api/todoKeys.ts` — query-key factory; prevents key drift. Used by both the SC prefetch and the client hooks, so they share keys.
```ts
export const todoKeys = {
  all: ["todos"] as const,
  list: (params: ListTodosParams) => [...todoKeys.all, "list", params] as const,
  detail: (id: string) => [...todoKeys.all, "detail", id] as const,
};
```

`hooks/` — TanStack Query wrappers own keys, caching, invalidation, and the wire->view mapping via `select`.
```ts
export function useTodos(params: ListTodosParams) {
  return useQuery({
    queryKey: todoKeys.list(params),
    queryFn: () => listTodos(params),
    select: (page) => ({ ...page, items: page.items.map(toTodoView) }), // wire -> view
  });
}
```

### Type layering
Three distinct kinds of type — do not conflate:
1. **Wire schemas** (`api/`, Zod) — exact backend contract. Response (`todoSchema`) and request bodies (`createTodoSchema` = `{ title }`, `updateTodoSchema` = partial for JSON Merge Patch). Pure mirror of the API; no UI concerns.
2. **View / form types** (`types.ts`) — derived/UI-enriched: view models with computed fields (`isOverdue`, `displayLabel`), optimistic shapes, react-hook-form values.
3. **Mapping** — wire -> view happens in the TanStack Query `select` option. The service returns wire types; `select` produces the view model; components consume the view model.

```
wire Todo (todoSchema)  --select-->  TodoView (types.ts)  -->  component
TodoFormValues (RHF)    --maps to-->  createTodoSchema     -->  POST body
```
Never add view concerns to the Zod wire schema — compute them downstream.

### Routing
File-system routing under `app/`. A route segment's `page.tsx` is a Server Component that prefetches and hydrates (pattern above). `layout.tsx` provides shared shells. Dynamic segments use `[id]`. The feature provides the components/hooks/services the page composes; `app/` provides the route + fetch orchestration.

Route-level concerns map to App Router special files:
- `loading.tsx` — Suspense fallback while the SC fetch resolves.
- `error.tsx` (`"use client"`) — segment error boundary; catches render errors and errors thrown in the SC fetch.
- `not-found.tsx` — rendered when a SC calls `notFound()`.
- `global-error.tsx` — root layout crash fallback.

### Mutations
**Data mutations go through the fetch wrapper from Client Components** (via TanStack Query mutation hooks) — this preserves the `ProblemError` contract and keeps them MSW-testable. **Server Actions are reserved for framework concerns only** — auth/session, redirects, and cache revalidation (`revalidateTag`/`revalidatePath`) — never for data CRUD against the backend.

```ts
// CC mutation — fetch-based, ProblemError-throwing, MSW-testable
const { mutate } = useCreateTodo(); // POST /todo-items via fetchClient

// Server Action — framework concern only
"use server";
export async function revalidateTodos() { revalidateTag("todos"); }
```

### Components & styling
- Flat structure, one PascalCase folder per component, colocated test.
- Tailwind utility classes; design tokens in `tailwind.config.ts`.
- **Next.js primitives are mandatory**: `next/link` for internal navigation (never raw `<a>`), `next/image` for images (never raw `<img>`), `next/font` for fonts, and the Metadata API (`export const metadata` / `generateMetadata`) for `<title>`/SEO in SC pages and layouts.
- Build small, composable pieces; promote to `shared/components/` only when a second feature needs it. A domain-rich reusable component stays in its feature and is shared via the barrel.

### Forms
react-hook-form with the Zod resolver. The form's Zod schema feeds both client validation and the request body shape. Server-side validation errors (HTTP 422) are mapped back into the form via `setError` and shown inline per field. Forms submit through fetch-based mutation hooks, not Server Actions.

## State / TanStack Query defaults
Configured once in `app/get-query-client.ts`. Overriding a default per-query is a deliberate choice.
- `staleTime: 30_000` — reuse cache for 30s before considering data stale. **Set a non-zero `staleTime`** so hydrated data isn't immediately refetched on mount.
- `retry: 1`, and **never retry 4xx** — inspect `ProblemError.status`; only retry network/5xx. Tests use `retry: 0`.
- `refetchOnWindowFocus: true` (guarded by `staleTime`).
- `gcTime`: default (5 min).
- **Mutations**: default pattern is **invalidate-on-success** (`onSuccess` invalidates the relevant `todoKeys`). A global mutation `onError` toasts `ProblemError.detail`. Optimistic updates (snapshot -> optimistic write -> rollback on error) are opt-in for specific UX, not the default.

### QueryClient lifecycle (App Router footgun)
`get-query-client.ts` returns a **new client per request on the server** and a **singleton in the browser**. A module-level server singleton would leak cache across requests/users — do not do it.
```ts
import { isServer, QueryClient } from "@tanstack/react-query";
function makeClient() { return new QueryClient({ defaultOptions: { queries: { staleTime: 30_000 } } }); }
let browserClient: QueryClient | undefined;
export function getQueryClient() {
  if (isServer) return makeClient();          // fresh per request
  return (browserClient ??= makeClient());    // singleton in browser
}
```

## Caching — TanStack owns it
There are two cache systems in play; the rule is **TanStack Query owns all caching, full stop**.
- Server-Component service fetches run **uncached** (`cache: "no-store"`, also Next 15's default) — they exist only to produce first-paint data, which is dehydrated into TanStack.
- After hydration, TanStack is the single source of truth. `staleTime`, invalidation, and refetch govern freshness.
- Mutations invalidate `todoKeys`; the corresponding `useQuery` refetches. **No `revalidateTag` for data** — that's reserved for genuine framework revalidation, not the normal data path.
- Anything that must react to mutations lives in TanStack (a Client Component + `useQuery`). Do not render live, mutation-sensitive data directly in a Server Component outside a hydration boundary.

## Error handling
The shared fetch wrapper owns wire -> error mapping. On a non-2xx response it parses and Zod-validates the RFC 7807 Problem Details body and throws a typed `ProblemError`:
```ts
export class ProblemError extends Error {
  constructor(readonly problem: ProblemDetails, readonly status: number) {
    super(problem.detail ?? problem.title);
  }
}
```

Errors fire in two runtimes; route each accordingly:

**Server-Component fetch (initial load)**
- A `ProblemError` thrown during the SC fetch propagates to the segment `error.tsx`.
- A 404 is special-cased: the SC calls `notFound()` → renders `not-found.tsx`.
- `loading.tsx` covers the Suspense window.

**Client fetch (post-hydration query/mutation)** — unchanged contract:
- **422 validation** → mapped into react-hook-form fields (`setError`), inline per field.
- **404** on a client read → component "not found" state.
- **409 / 5xx / network** → component-level error UI (`query.isError`) and a toast for failed mutations.

Rule of thumb: **first-paint errors use App Router files; interaction errors use component-level UI.** SC services throw `ProblemError`; the SC decides `notFound()` (404) vs rethrow (everything else → `error.tsx`). `ProblemDetails`, `problemSchema`, and the wrapper live in `shared/api/`.

## Fetch client — one isomorphic wrapper
A single `fetchClient` entry is used by services regardless of runtime; it resolves base URL and auth internally so services stay runtime-neutral and pure.

| | Server path (`fetchClient.server.ts`) | Browser path (`fetchClient.browser.ts`) |
|---|---|---|
| Base URL | internal `API_URL` (server-to-server) | `NEXT_PUBLIC_API_URL` |
| Auth | forwards cookies/headers via `next/headers` | browser sends cookies (`credentials: "include"`) |
| Caching | `cache: "no-store"` | n/a |

The server path imports `next/headers` and is marked `import "server-only"`; the browser path carries no server imports. `shared/api/fetchClient.ts` re-exports the correct path so the browser bundle never pulls server code. Services and hooks import only the single entry.

## Validation / environment
Two tiers, each parsed once through a Zod schema; a missing/malformed var throws at boot, not mid-request.

```ts
// shared/config/env.client.ts — in the bundle, PUBLIC
export const clientEnv = z.object({
  NEXT_PUBLIC_API_URL: z.string().default("/api"),
}).parse({ NEXT_PUBLIC_API_URL: process.env.NEXT_PUBLIC_API_URL });

// shared/config/env.server.ts
import "server-only";
export const serverEnv = z.object({
  API_URL: z.string().url(),            // internal server-to-server URL
}).parse(process.env);
```
- Dev uses a Next.js rewrite: `next.config.ts` maps `/api/*` -> the running backend, so the browser calls same-origin `/api/...` and avoids CORS. Server-side fetches use `API_URL` directly.
- App code imports `clientEnv` / `serverEnv`, never raw `process.env`.
- Commit `.env.example` (documents required vars). Gitignore `.env.local`.

### Secrets
Unlike a pure SPA, Next.js has a server runtime, so **secrets may live server-side** — in Server Components, API routes, Server Actions, and the server fetch path. Rules:
- `NEXT_PUBLIC_*` is the literal opt-in to client exposure — **never** put a secret behind that prefix. Anything `NEXT_PUBLIC_` is in the public bundle.
- Server-only secrets are read through `serverEnv` (guarded by `import "server-only"`), never imported into a Client Component.
- Third-party keyed APIs: call them from the server tier (route handler / SC), never from the browser.
- Rule of thumb: **if it must stay secret, it never gets a `NEXT_PUBLIC_` prefix and never crosses into a Client Component.**

### Auth (seam only — not implemented)
The template ships no auth; Todos is unauthenticated. Two seams are left for adopters to wire their own (Auth.js, a dotnet session cookie, etc.):
- `middleware.ts` — the slot for route protection / redirects.
- The server path of the fetch wrapper — the place to forward the session cookie/header on server-to-server calls (`fetchClient.server.ts`).

## Testing

### Strategy
- **Jest + RTL** covers the bulk: Client Components, hooks, services, Zod schemas, mappers.
- **Playwright** covers what RTL/Jest cannot: Server Components, routing, streaming, and the SC→CC hydration handoff.
- **Never unit-test an async Server Component** — there is no stable renderer for it. Extract its logic into plain functions (they already live in `api/` services) and unit-test those; cover the SC shell with Playwright.

### Placement & naming
- **Unit / component / hook tests**: colocated next to the file under test (`todoService.test.ts`, `useTodos.test.ts`, `TodoList.test.tsx`).
- **Feature integration tests** (multi-component RTL flows): `features/<feature>/__tests__/`.
- **E2E**: `e2e/` at the project root.
- **Test support** (builders in `testing/`, MSW handlers in `mocks/`) holds reusable fixtures/fakes, not the tests themselves.
- Naming: `describe(<unit>)` + `it("does X behavior")` in plain English. One behavior per test (exception: a parameterized `it.each`).

### React Testing Library conventions
- Query by accessible role/label first: `getByRole`, `getByLabelText`, `getByText`. `getByTestId` is a last resort — needing it signals an accessibility gap; fix the component, not the test.
- Use `@testing-library/user-event`, never `fireEvent`.
- Assert on user-visible behavior, never class names or component internals.
- Render only via `renderWithProviders` (`src/test/`), which supplies a fresh `QueryClient`. Never call bare RTL `render`.

### MSW (network in tests — test-only)
MSW is the fake *other side* of the wire; it intercepts HTTP and returns canned responses. Real fetch + Zod + error mapping in `todoService.ts` stay under test — only the network is faked. **MSW runs in tests only**; the dev app talks to a real backend.
- Handlers live in the feature's `mocks/` folder and mirror **API resources/endpoints**, built from builders, returning paged envelopes / Problem Details as the backend would.
- `src/test/server.ts` (Node `setupServer`) composes all features' handlers as defaults; individual tests override per case with `server.use(...)`.

### Test data builders
Overrides factory + named recipes. Returns the parsed (clean) type.
```ts
// features/todos/testing/todo.builder.ts
export function buildTodo(overrides?: Partial<Todo>): Todo {
  return { id: "todo-1", title: "Buy milk", status: "Pending", createdAt: new Date("2026-01-01"), ...overrides };
}
export const buildInProgressTodo = (o?: Partial<Todo>) => buildTodo({ status: "InProgress", ...o });
export const buildDoneTodo = (o?: Partial<Todo>) => buildTodo({ status: "Done", ...o });
```
Builders are `*.builder.ts`, live in a `testing/` folder, imported directly by tests, and are **never** exported from a feature barrel. Shared-type builders live in `shared/testing/builders/`.

### Playwright (SC / routing / hydration)
- Covers route navigation, `loading.tsx`/`error.tsx`/`not-found.tsx` behavior, and that a hydrated page shows data without a loading flash.
- Runs against the dev/preview server with a real (or seeded) backend. To exercise a specific visual state, seed the backend rather than mocking.

### Test client
`src/test/test-query-client.ts` creates a fresh `QueryClient` per test with `retry: false` — prevents cache bleed and stops retry timers from hanging tests.

## What NOT to do
1. Do not import across feature internals — only through the feature barrel (`features/x/index.ts`).
2. Do not put domain logic in `shared/` — domain stays in its feature; `shared/` is domain-agnostic only.
3. Do not import `testing/`, `mocks/`, or `src/test/` from production code (forbidden by ESLint `no-restricted-imports`).
4. Do not read `process.env` directly — use `clientEnv` / `serverEnv`.
5. Do not give a secret a `NEXT_PUBLIC_` prefix, and never import `serverEnv`/server-only modules into a Client Component.
6. Do not fetch outside the `api/` service layer — components and hooks never call `fetch` directly.
7. Do not hold server/async state in `useState`/`useEffect` — it goes through TanStack Query.
8. Do not pollute the Zod wire schema with view concerns — compute view types via `select`.
9. Do not use a bare RTL `render` — use `renderWithProviders`.
10. Do not throw or handle a raw `Response` in components — the fetch wrapper throws a typed `ProblemError`.
11. Do not pass server-fetched server-state down as props to read in Client Components — hydrate it and read via `useQuery`.
12. Do not use Server Actions for data CRUD — those go through the fetch wrapper. Server Actions are for framework concerns (auth, redirect, `revalidateTag`) only.
13. Do not render live, mutation-sensitive data directly in a Server Component outside a hydration boundary.
14. Do not add `"use client"` to a route segment by default — server is the default; justify every opt-in.
15. Do not introduce a second cache for the same data — TanStack owns caching; SC fetches are uncached first-paint only.
16. Do not unit-test an async Server Component — extract logic and unit-test it; cover the shell with Playwright.
17. Do not add per-component barrel files — the feature-level barrel is the only one.
18. Do not export test support from a feature barrel.
19. Do not select elements by `data-testid` when an accessible role/label query works.
20. Do not use `fireEvent` — use `user-event`.
21. Do not use raw `<a>`/`<img>` for internal nav/images — use `next/link` / `next/image`.
22. Do not couple feature folder names to REST paths — feature folders are product concepts (`todos`); the exact API path (`/todo-items`) stays in the service.
