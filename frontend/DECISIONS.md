# Architecture Decisions — Next.js Frontend Template

This document explains **why** the frontend template is built the way it is. `CLAUDE.md` is the *spec* (the rules to follow); this is the *reasoning* (why those rules exist, what else was considered, and what tradeoff each choice accepts).

It started as an adaptation of an earlier Vite + React SPA template. Roughly half of that template's ideas survived unchanged (vertical slices, Zod at the boundary, typed errors, TanStack Query, the testing conventions). The other half had to be rethought for Next.js App Router, because server rendering changes where data is fetched, where secrets can live, and how errors surface.

## Context that drives everything

This repo is **one frontend half of a mix-and-match template library**. The plan is to keep several frontend templates and several backend templates, and assemble a full-stack app by picking one of each as sibling `frontend/` + `backend/` folders. Two consequences shape almost every decision below:

1. **Backend-agnostic in code, backend-present at runtime.** The frontend never hard-codes a specific backend, but you *do* develop against a real one you run yourself (typically a dotnet backend). The only thing the two halves share is a wire contract — REST resource URLs, RFC 7807 Problem Details errors, and paged response envelopes. That contract is the seam that makes the halves swappable.
2. **It's a template others copy.** "Idiomatic and familiar" sometimes beats "my personal favorite," because the value is in being recognizable to whoever adopts it.

---

## 1. Rendering model: Hybrid (not RSC-purist, not client-SPA)

**Decision:** Server Components handle the initial data fetch and render; Client Components handle interactivity. TanStack Query remains the client-side cache.

**Alternatives considered:**
- *RSC-first* — Server Components fetch everything; TanStack Query mostly removed. Maximum use of the platform, but it deletes the strongest, most battle-tested part of the old template and makes testing harder (you can't easily render async Server Components in tests).
- *Client-SPA inside Next.js* — keep everything `"use client"` and use Next.js only as a bundler/router. Least rework, but it throws away the reason to use Next.js at all (streaming, server fetch, smaller bundles, SEO).

**Why hybrid:** It keeps the proven client patterns (TanStack Query, the service layer, typed errors) while still getting the real Next.js wins on first paint. It's the middle path that doesn't burn down either side.

**Tradeoff accepted:** TanStack Query's role narrows — it no longer owns the *initial* fetch on a server-rendered page (the Server Component does that). It owns everything after hydration. That's a deliberate, documented boundary, not an accident.

---

## 2. Backend relationship: agnostic in code

**Decision:** The template defines the client-side contract (Zod schemas, fetch wrapper, error shape, `API_URL`) but prescribes nothing about what's behind `/api`. The reference Todos feature talks to whatever backend honours the contract.

**Alternatives considered:**
- *Tie to a specific dotnet backend* — what the original Vite template did. Convenient for one project, but it's exactly the coupling the mix-and-match library is trying to avoid.
- *Make Next.js itself the backend* (Server Actions + a database) — turns this into a full-stack template, not a frontend half. Wrong shape for the library.

**Why agnostic:** The whole point of the template library is to swap backend halves freely. The frontend can only be swappable if it depends on a *contract*, not on a particular server.

**Tradeoff accepted:** There's no runnable backend bundled in this repo. You bring your own. That's intentional — the backend is a separate template you assemble alongside.

---

## 3. Server/Client boundary: server page by default, client islands by opt-in

**Decision:** A route's `page.tsx`/`layout.tsx` is a Server Component by default. `"use client"` is an explicit opt-in only where interactivity needs it. The standard data flow is: the Server Component prefetches into a query cache, dehydrates it, and wraps the tree in `<HydrationBoundary>`; Client Components then read that data via `useQuery`.

**Why this pattern specifically:** It solves the "prop drilling" problem cleanly. The Server Component does the fetch once, and any Client Component deep in the tree reads the data straight from the hydrated cache — no passing data through five layers of props, and no "loading…" flash on first render because the cache is already warm. It's also the pattern TanStack Query officially recommends for RSC.

**On sharing data between server components:** React's `cache()` function deduplicates server-side reads within a single render, so multiple Server Components can call the same service function and share one result. But `cache()` is server-only — it does **not** reach into Client Components. Crossing from server to client always goes through the prefetch + hydration boundary, never through `cache()`.

**Tradeoff accepted:** Slightly more ceremony on each server page (prefetch + dehydrate + boundary). In exchange you get no client waterfall, no prop drilling, and a single consistent read path (`useQuery`) everywhere on the client.

---

## 4. Mutations: fetch for data, Server Actions for framework concerns only

**Decision:** Data create/update/delete go through the fetch wrapper from Client Components (via TanStack Query mutation hooks). Server Actions are used *only* for framework concerns — auth, redirects, and cache revalidation.

**Alternatives considered:**
- *Server Actions for all mutations* — the trendy Next.js approach. But Server Actions can't be intercepted by MSW (so they break the testing/mocking story) and they don't return HTTP responses, so the typed `ProblemError` contract falls apart.
- *Everything through fetch, no Server Actions at all* — fully consistent, but you give up genuinely useful Server Action features like `revalidateTag` and clean auth redirects.

**Why the split:** It keeps the data path consistent and testable (fetch + `ProblemError` + MSW all intact) while still using Server Actions where they're actually the right tool.

**Tradeoff accepted:** Two mechanisms exist instead of one, so the rule must be stated clearly: *Server Actions never do data CRUD.* The boundary is "framework concern vs. data concern."

---

## 5. Folder model: `app/` owns routes, features own everything below

**Decision:** `app/` owns route composition and the Server-Component fetch orchestration (the page, its prefetch, Suspense/error boundaries). Features own the reusable pieces below the page — components, hooks, the API/service layer, mocks, and test support. Features no longer ship a `pages/` folder or a `routes.tsx`.

**Alternatives considered:**
- *Thin re-export shims in `app/`* — make every `app/.../page.tsx` a one-liner that re-exports a page from the feature. This fights Next.js's file-system routing and scatters the server fetch logic away from where the route actually lives. Pure noise.
- *Route groups mirroring features* — possible, but it forces awkward folder gymnastics to satisfy both the routing system and the slice structure.

**Why this split:** Next.js routing is file-location-based; fighting it loses. The page is *where the server fetch should happen*, so the page belongs in `app/`. Everything reusable still lives in the feature and is reached only through its barrel.

**Tradeoff accepted:** This changes one rule from the old template — features used to own `pages/`. Now route-level composition lives in `app/`. The feature still owns all the substance; only the thin route shell moved.

---

## 6. Environment & secrets: two tiers, because there's a server now

**Decision:** Two validated env modules — `env.client.ts` for `NEXT_PUBLIC_*` (public, in the bundle) and `env.server.ts` for server-only vars (guarded by the `server-only` package). Both are parsed through Zod at boot, so a missing or malformed variable fails immediately rather than mid-request.

**Why this is a real change:** The old SPA template held *zero* secrets — everything in a browser bundle is public, so secrets had to live behind a separate backend. Next.js has a server runtime, so secrets *can* now safely live server-side (in Server Components, route handlers, the server fetch path). This is one of the biggest upgrades the move to Next.js unlocks.

**Alternatives considered:**
- *Keep "zero secrets"* — simplest, but wastes the server runtime and forces a separate service for every privileged call.
- *One env module split at runtime* — fewer files, but a single mistake can leak a server variable into the client bundle. The two-file + `server-only` guard makes that a build error instead.

**Tradeoff accepted:** Two files and a discipline rule (never give a secret a `NEXT_PUBLIC_` prefix, never import the server module into a client component). The safety is worth the small extra structure.

---

## 7. Test runner: Jest (plus Playwright), chosen for familiarity

**Decision:** Jest via `next/jest` + React Testing Library for unit/component/hook tests, and Playwright for end-to-end (which is the only way to test Server Components, routing, and the hydration handoff).

**Alternatives considered:**
- *Vitest* — what the old template used, and arguably nicer DX. The entire testing section (RTL conventions, render helper, builders, MSW) would have transferred verbatim, saving a rewrite.

**Why Jest won here:** This is a template others adopt, and "idiomatic Next.js" carried the decision. A developer opening a Next.js template is more likely to expect `next/jest` and Jest. Neither Jest nor Vitest can render async Server Components anyway, so that limitation wasn't a tiebreaker.

**The Server-Component testing rule:** You *cannot* unit-test an async Server Component today — there's no stable renderer. So the rule is: extract the logic into plain functions (which already live in the service layer) and unit-test those; cover the Server-Component shell, routing, and hydration with Playwright. This keeps unit coverage high without fighting an unsolved problem.

**Tradeoff accepted:** Playwright is a whole new testing pillar the SPA template didn't need, and it requires a running backend (or a seeded one) to exercise real states.

---

## 8. MSW (mocking): test-only, no dev-runtime mocking

**Decision:** MSW runs in tests only (Node `setupServer`). The dev app always talks to a real backend. The browser service worker, the `VITE_ENABLE_MOCKS` flag, and the "backend-optional dev" feature from the old template are all removed.

**Why the old approach didn't survive:** The SPA template let you run the whole app with no backend by faking the network in the browser. That worked because *all* fetches happened in the browser. In the hybrid Next.js model, the initial fetch happens on the *server*, where a browser service worker can't see it. Keeping backend-optional dev would have required intercepting on the server too (via Next.js instrumentation) — which is fiddly, fragile, and version-sensitive.

**What made it an easy cut:** The actual dev workflow is "I have my dotnet backend running when I build a feature." So zero-backend dev wasn't needed. And error states (500/409/network) don't need a live browser mock either — those are tested in Jest with MSW, not faked by hand in the running app.

**Tradeoff accepted:** To see a *specific* visual state (say, fifty todos with one overdue), you seed the real backend rather than flipping a mock. Minor, and the test builders already construct those states for tests.

---

## 9. Caching: TanStack Query owns it; server fetches are uncached first-paint only

**Decision:** Server-Component service fetches run uncached (`no-store`, also Next 15's default). They exist only to produce first-paint data, which is handed to TanStack Query. After hydration, TanStack Query is the single cache — its `staleTime`, invalidation, and refetch rules govern freshness. The Next.js Data Cache and `revalidateTag` are *not* used for the normal data path.

**Why one cache instead of two:** Next.js and TanStack Query each have their own caching layer, and coordinating both by hand is where hybrid apps quietly rot — stale data bugs that nobody can trace. Picking a single owner (TanStack) gives a mental model a person can actually hold.

**Alternatives considered:**
- *Use the Next.js Data Cache with tags + `revalidateTag`* — maximum performance (CDN-cacheable server content), but maximum complexity, and only worth it for genuinely cacheable public pages. A backend-driven app behind auth isn't that.
- *Per-route choice* — possible, but inconsistency is itself a cost in a template meant to teach a pattern.

**Tradeoff accepted:** You don't get Next.js's CDN-level caching of server-rendered data. For an interactive, backend-driven app that's the right call; the simplicity is worth more than the cache hit.

---

## 10. Errors: split by where the fetch happened

**Decision:** The fetch wrapper always throws a typed `ProblemError` (parsed from the RFC 7807 body). Where it surfaces depends on the runtime:
- **Server-Component fetch (initial load):** a thrown error propagates to the segment `error.tsx`; a 404 calls `notFound()` and renders `not-found.tsx`; `loading.tsx` covers the wait.
- **Client fetch (after hydration):** unchanged from the old template — 422 maps into form fields, 404 shows a "not found" state, 409/5xx/network show component-level error UI plus a toast for failed mutations.

**Why split this way:** App Router gives you dedicated files (`error.tsx`, `not-found.tsx`, `loading.tsx`) that are the *idiomatic* place for first-paint failures — using them is cleaner than hand-rolling boundaries. But they're useless for an interaction like a form submit, so client-side errors keep the old, proven component-level handling.

**Rule of thumb:** first-paint errors use App Router files; interaction errors use component UI. The typed error contract is the same on both sides — only the rendering differs.

**Tradeoff accepted:** There are two error-rendering paths to understand. The mapping table in `CLAUDE.md` keeps it concrete.

---

## 11. Fetch client: one isomorphic wrapper

**Decision:** Services import a single `fetchClient`. Internally it resolves the right base URL and auth strategy by runtime: on the server it uses the internal `API_URL` and forwards cookies/headers; in the browser it uses `NEXT_PUBLIC_API_URL` and relies on automatic cookie sending. The server path is isolated in a `server-only` module so it never leaks into the browser bundle.

**Alternatives considered:**
- *Two explicit clients* — clearer about which runtime you're in, but services stop being runtime-neutral (they'd have to pick a client or receive one), causing duplication.
- *Inject the fetcher into every service call* — pure and testable, but every call site has to thread the fetcher through. Too much ceremony.

**Why one wrapper:** The old template's single, pure service layer was one of its best features. Keeping one entry point preserves that — services and hooks look identical to before; only the wrapper grew a runtime branch.

**Tradeoff accepted:** The wrapper has to be carefully split internally (`.server.ts` / `.browser.ts` behind one entry) so the browser build doesn't pull in server-only code. That's a one-time setup cost paid in `shared/api/`.

---

## 12. Auth: a documented seam, not an implementation

**Decision:** The template ships no auth. The Todos reference feature is unauthenticated. Two seams are marked for adopters to wire their own solution: `middleware.ts` (route protection) and the server path of the fetch wrapper (where to forward a session cookie).

**Why not pick a library:** A template's job is the architectural spine, not your auth vendor. Wiring in Auth.js or a specific session scheme would force a choice on every adopter and add a large surface that most would have to rip out.

**Why a seam and not nothing:** The server fetch path already needed a place to attach credentials, so leaving a clearly labeled slot (rather than silence) tells adopters exactly where their auth plugs in.

**Tradeoff accepted:** The template isn't "batteries-included" for auth. Given the mix-and-match goal — where the backend half owns the real session model — that's the right boundary.

---

## 13. Repo layout: keep the `frontend/` folder

**Decision:** The Next.js app lives under `frontend/`, with its spec at `frontend/CLAUDE.md`, assuming a sibling `backend/` at runtime and an optional thin root `CLAUDE.md` above both.

**Why not put the app at the repo root:** Because of the mix-and-match goal, this repo is specifically the *frontend half*. Keeping the `frontend/` nesting means it drops straight into the assembled `frontend/` + `backend/` layout without rearranging anything. The folder isn't vestigial — it's the slot this half occupies in every assembled app.

**Tradeoff accepted:** A little extra nesting in a repo that, viewed alone, contains only a frontend. Justified entirely by how the halves get assembled.

---

## 14. Next.js primitives and `src/`

**Decision:** Adopt `next/link`, `next/image`, `next/font`, and the Metadata API as mandatory conventions. Keep the `src/` directory with the `@/*` → `src/*` alias.

**Why the primitives are mandatory:** They're the idiomatic-Next.js bar (the same reasoning that picked Jest). `next/image` and `next/font` are real performance wins (optimization, no layout shift), and the Metadata API is the *only* correct way to set head tags in App Router. Raw `<a>`/`<img>` for internal nav/images is now a "don't."

**Why keep `src/`:** Next.js fully supports `src/app`. Keeping it preserves the old template's tree almost verbatim, keeps config files out of the code directories, and keeps the `@/*` alias working. Only the *contents* of `app/` change (to file-based routing), not its location.

**Tradeoff accepted:** None significant — these are low-controversy, high-familiarity choices.

---

## What carried over essentially unchanged

These were good ideas in the SPA template and remain good ideas here:

- **Vertical slices** with atomic design as a principle, not a folder taxonomy.
- **Three-layer types:** Zod wire schemas (the API contract), view/form types, and a mapping step in TanStack Query's `select`.
- **Zod validation at the boundary** — the wire schema is the single source of truth for the wire type.
- **Typed errors** — one `ProblemError` from RFC 7807 Problem Details, shared by every feature.
- **Query-key factory** to prevent key drift.
- **Testing conventions** — accessible queries first, `user-event` over `fireEvent`, assert on visible behavior, render only through the providers helper.
- **Test data builders** — overrides factory plus named recipes, kept out of feature barrels.
- **Dependency direction** — `app → features (via barrels) → shared`, and "promote to `shared/` only on the second use."
