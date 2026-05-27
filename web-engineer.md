# Frontend patterns and best practices — Web

> Conventions for **React on the web** (Next.js, plain React, Remotion). Drop this file into a project as context for Claude Code; it's self-contained.

---

## 0. Skills

This agent delegates specialized work to external skills via the Skill tool. If a skill is not installed in the user's environment, continue without it — never block on a missing skill.

### Install (one-time setup)

Run these once per project (or globally) to make the skills available:

```bash
pnpx skills add https://github.com/anthropics/skills --skill frontend-design
pnpx skills add https://github.com/nextlevelbuilder/ui-ux-pro-max-skill --skill ui-ux-pro-max
pnpx skills add https://github.com/vercel-labs/next-skills --skill next-best-practices
pnpx skills add https://github.com/vercel-labs/next-skills --skill next-cache-components
pnpx skills add https://github.com/vercel-labs/agent-skills --skill vercel-react-best-practices
pnpx skills add https://github.com/vercel-labs/agent-skills --skill vercel-composition-patterns
pnpx skills add https://github.com/vercel-labs/agent-skills --skill vercel-react-view-transitions
pnpx skills add https://github.com/vercel-labs/agent-browser --skill agent-browser
pnpx skills add https://github.com/remotion-dev/skills --skill remotion-best-practices
pnpx skills add supabase/agent-skills --skill supabase
pnpx skills add supabase/agent-skills --skill supabase-postgres-best-practices
pnpx skills add https://github.com/currents-dev/playwright-best-practices-skill --skill playwright-best-practices
pnpx skills add https://github.com/microsoft/playwright-cli --skill playwright-cli
pnpx skills add https://github.com/coreyhaines31/marketingskills --skill seo-audit
pnpx skills add https://github.com/pbakaus/impeccable --skill animate
pnpx skills add https://github.com/pbakaus/impeccable --skill colorize
pnpx skills add https://cli.sentry.dev
```

### When to invoke each skill

| Skill | Invoke when |
|---|---|
| `frontend-design` | Designing a new component or screen before writing JSX — variants, states, copy, layout. |
| `ui-ux-pro-max` | Polishing UX details: motion, focus order, error feedback, microinteractions. |
| `next-best-practices` | Any Next.js App Router work — routing, layouts, Server/Client component boundaries, data fetching, Server Actions (§5, §15). |
| `next-cache-components` | Caching strategy — `cache`, `revalidate`, `unstable_cache`, ISR, `dynamic`/`force-static` (§15). |
| `vercel-react-best-practices` | Writing React components — hooks, render correctness, common pitfalls. |
| `vercel-composition-patterns` | Component composition design — props, slots, render props, when to lift state (§5). |
| `vercel-react-view-transitions` | Animating route transitions, list reorders, optimistic UI with the View Transitions API. |
| `agent-browser` | Driving a real browser to verify a UI flow end-to-end before declaring the change complete. |
| `remotion-best-practices` | Working in a Remotion project — compositions, sequences, programmatic video, render config. |
| `supabase` | Any Supabase work — `@supabase/ssr` clients, auth, storage, realtime, Edge Functions (§17). |
| `supabase-postgres-best-practices` | Writing or reviewing migrations, RLS policies, indexes, triggers, or any raw SQL. |
| `playwright-best-practices` | Writing E2E specs in `features/<name>/tests/<scenario>.spec.ts` (§12). |
| `playwright-cli` | Recording a flow with `codegen`, debugging with `--ui`, or running specs from the command line. |
| `seo-audit` | Public pages with SEO needs — `generateMetadata`, sitemaps, structured data, Core Web Vitals (§15). |
| `animate` | Designing or implementing animations and motion — easing, timing, choreography, View Transitions, CSS/JS animation. |
| `colorize` | Choosing or refining color palettes, contrast, and theming decisions. |
| `sentry-cli` | Configuring Sentry, uploading source maps, tagging releases, scrubbing PII via `beforeSend` (§14, §18). |

When multiple skills apply to the same task, invoke them broadest-to-narrowest — e.g. `frontend-design` for the component shape, then `vercel-composition-patterns` for the API surface, then `vercel-react-best-practices` for the React-level details.

### Supabase MCP

In addition to the `supabase` and `supabase-postgres-best-practices` skills, this agent may use the **Supabase MCP server** when available. Prefer the MCP for live project introspection — listing tables, inspecting schemas, running read-only SQL, generating types, applying migrations, checking logs, and reading advisor notices. Skills provide patterns; the MCP provides ground truth against the running project. Use both together: consult the skill for *how* to do something, then use the MCP to verify the *current state* of the project before acting.

---

## 1. File and folder naming — kebab-case

- File names: `user-card.tsx`, `format-date.ts`, `orders.api.ts`, `user-card.test.ts`.
- Folder names: `features/order-history/`, `components/empty-state/`.
- The **exported symbol** keeps its natural casing — PascalCase for components, camelCase for functions. The _file_ is kebab-case regardless:
  - `user-card.tsx` exports `UserCard`
  - `format-date.ts` exports `formatDate`
  - `orders.utils.ts` exports `formatPrice`, `calculateTotal`, etc.
- Exceptions: framework-mandated filenames keep the framework's convention (`page.tsx`, `layout.tsx`, `_app.tsx`, `route.ts`, `middleware.ts`, etc.).
- Applies repo-wide, including documentation.

## 2. Alphabetical ordering

Sort alphabetically wherever order is not semantic:

- Object literal keys (props passed to a component, config objects, style objects when not order-dependent).
- Type / interface keys.
- Function parameter object destructuring (`function Foo({ a, b, c }: Props)`).
- Imports _within_ a group. Group order: React → external → `@/` aliases → relative; **within each group, alphabetical**.

Skip ordering when order carries meaning (animation step arrays, route precedence, CSS cascade overrides).

## 3. Named exports only

```ts
// component file (user-card.tsx)
export function UserCard(props: Props) {
  /* ... */
}

// consumer
import { UserCard } from "@/shared/components/user-card";
```

- Never `export default`. Never `import UserCard from '...'`.
- Applies to hooks, utils, and constants too.
- Default exports are reserved for cases mandated by frameworks (Next.js `page.tsx`, `layout.tsx`, `error.tsx`, `loading.tsx`, route handlers, middleware files).
- Supabase auth callback (`app/auth/callback/route.ts`) and `middleware.ts` are framework files — same exception applies (§17).

## 4. All feature functions live in utils

**No function definitions inside components — including event handlers.** Every named function for a feature lives in `<name>.utils.ts` (or `shared/utils/<name>.ts` per the shared/feature rule below). Components compose; utils do work.

This means:

- No `const handleClick = () => { ... }` inside a component.
- No `function onSubmit(values) { ... }` declared inside a component body.
- JSX props bind to a util directly or via a one-line arrow that just calls the util with closure values: `<Button onClick={() => addItemToCart({ cart, itemId: item.id, setCart })} />`. The arrow is a dispatch site, not a function definition — keep it to one expression.
- When a util needs values that live in component scope (state, refs, hooks, router), pass them in via the params object. Don't try to make the util a closure; that's what the inline JSX arrow is for.

### Function names say what the function does

Function names describe the **action**, not the call site or the trigger. You should know what a function does from its name alone, without reading its body or seeing where it's used.

- BAD: `handleClick`, `handlePress`, `handleAdd`, `onSubmit`, `doStuff`, `process`.
- GOOD: `addItemToCart`, `submitLoginForm`, `removeOrderById`, `navigateToOrderDetails`, `formatPrice`, `calculateInstallmentAmount`, `validateEmail`.

```ts
// features/orders/orders.utils.ts
export function calculateInstallmentAmount(total: number, n: number): number {
  /* ... */
}

export function formatPrice(value: number): string {
  /* ... */
}

export function submitOrderForm({ cart, customer }: SubmitOrderFormParams): Promise<Order> {
  /* ... */
}
```

### Feature vs. shared util

A util is shared (`shared/utils/<name>.ts`) only when its parameters and return value are not feature-specific shapes/types/schemas. If it receives or manipulates a type owned by a feature, it lives in that feature's `<name>.utils.ts`. `formatCurrency(value: number)` is shared. `summarizeOrder(order: Order)` is not — `Order` belongs to the orders feature.

## 5. Component file structure

**One component per file.** The file's name matches the component it exports (`user-card.tsx` exports `UserCard`, and only `UserCard`). No additional components — not even small "private" helpers — share the file. If a subcomponent emerges, give it its own file in the same directory. Utils, types, and constants live in their respective `<name>.utils.ts` / `<name>.types.ts` / `<name>.consts.ts` files (§4, §6, §8), never alongside the component.

Order inside a component file, separated by single blank lines:

```tsx
// 1. imports
import { useEffect } from "react";

import { Button } from "@/shared/components/button";
import { formatPrice, selectOrderById } from "@/features/orders/orders.utils";
import type { OrderCardProps } from "@/features/orders/orders.types";

// 2. component definition with props
export function OrderCard({ onSelect, order }: OrderCardProps) {
  // 3. variable declarations (const, let, destructuring)
  const formatted = formatPrice(order.total);

  // 4. useEffect / useMemo (only if present)
  useEffect(() => {
    /* ... */
  }, [order.id]);

  // 5. render — JSX dispatches to utils via inline arrows
  return (
    <div onClick={() => selectOrderById({ id: order.id, onSelect })}>
      <span>{formatted}</span>
    </div>
  );
}
```

One blank line between each numbered section. Skip a section entirely when there's nothing to put in it (don't leave a comment as a placeholder). **No `const handleX = ...` step** — see §4.

### Server vs. Client components (Next.js App Router)

- Default to **Server Components**. Only add `'use client'` when the component needs state, effects, browser APIs, or event handlers.
- Keep the `'use client'` directive at the top of the file, before imports.
- Push client boundaries as low as possible in the tree. Wrap interactive leaves, not whole pages.
- Server components can render client components, but not vice versa. Pass server-fetched data down as props.

---

## 6. Screaming Architecture

The project is organized by **feature**, not by technical concern. The directory tree communicates what the application does, not which framework it uses.

Why this layout:

- The structure communicates the application's purpose at a glance.
- Code is grouped by business domain, so a change to a feature touches one directory.
- Cross-feature dependencies are explicit and controlled — features don't reach into each other.

Top-level skeleton:

```
.
├── app/         # Next.js App Router routes
├── features/    # feature-sliced modules (per-feature tree below)
└── shared/      # cross-feature primitives
    ├── components/   # reusable UI
    ├── configs/      # global config (env, API clients, theme)
    ├── hooks/        # cross-feature hooks
    ├── icons/        # icon components
    ├── providers/    # third-party providers (React Query, theme, auth) — see below
    │   └── providers.tsx  # exports `Providers`, a single wrapper composing every provider
    ├── stores/       # global Zustand stores only — feature stores live in the feature (§16)
    └── utils/        # cross-feature pure utils (see §4)
```

Use a fixed file role set inside every feature directory:

```
features/<name>/
├── components/
├── tests/                  # unit/integration (.test.ts) + E2E Playwright flows (.spec.ts) — see §12
├── <name>.api.ts        # data access / fetchers (server actions, route handlers)
├── <name>.consts.ts     # static values, enums
├── <name>.hooks.ts      # React hooks
├── <name>.schemas.ts    # Zod schemas (forms, API boundaries)
├── <name>.store.ts      # Zustand store scoped to this feature (§16)
├── <name>.types.ts      # TS types (often derived from the database schema)
└── <name>.utils.ts      # pure functions (see §4)
```

Do not invent new file roles ad-hoc. If a new role is needed, decide it deliberately and apply it consistently across features.

### The `Providers` wrapper

`shared/providers/providers.tsx` exports a single `Providers` component that composes every app-wide client provider (React Query, theme, auth, etc.) in the correct order. The root `app/layout.tsx` imports it once and wraps `{children}` with it — no nested layout or page mounts providers. Because providers run on the client, `providers.tsx` carries the `'use client'` directive while `app/layout.tsx` stays a Server Component. Individual providers can still live in their own files inside `shared/providers/` (e.g. `query.provider.tsx`, `theme.provider.tsx`); `providers.tsx` is the composition root that consumes them.

```tsx
// shared/providers/providers.tsx
"use client";

import type { PropsWithChildren } from "react";

import { QueryProvider } from "@/shared/providers/query.provider";
import { ThemeProvider } from "@/shared/providers/theme.provider";

export function Providers({ children }: PropsWithChildren) {
  return (
    <QueryProvider>
      <ThemeProvider>{children}</ThemeProvider>
    </QueryProvider>
  );
}
```

```tsx
// app/layout.tsx
import { Providers } from "@/shared/providers/providers";

export default function RootLayout({ children }: { children: React.ReactNode }) {
  return (
    <html lang="en">
      <body>
        <Providers>{children}</Providers>
      </body>
    </html>
  );
}
```

## 7. Path aliases

- Configure in `tsconfig.json` `paths`. Aliases mirror the §6 top-level skeleton:
  - `@/app/*`        → `app/*`
  - `@/features/*`   → `features/*`
  - `@/shared/*`     → `shared/*`
- Inside `shared/`, drill in via the alias: `@/shared/components/button`, `@/shared/icons/users`, `@/shared/configs/supabase`.
- Use the alias whenever crossing a feature boundary; relative imports are reserved for siblings inside the same module.

## 8. TypeScript

### Compiler

- `strict: true` is mandatory. Plus: `noUncheckedIndexedAccess`, `noImplicitOverride`, `noFallthroughCasesInSwitch`.
- **No `any`, no `unknown`, no `as` casts.** Validate untyped data at system boundaries with Zod (`Schema.parse(...)` returns the typed value); from there on, every value already has a precise type. There is no legitimate escape hatch in feature code.
- The **only** permitted use of `as` is `as const` (literal narrowing of arrays / object literals — it makes types more precise, not less). Every other `as` form is banned.
- Function return types are inferred unless the function is exported across a feature boundary — in which case declare the return type explicitly.

### Naming conventions

- **Database row types**: bare singular noun, declared with `interface`. The shape mirrors a table row, so it's likely to be augmented or extended.

  ```ts
  // features/items/items.types.ts
  interface Item {
    id: string;
    name: string;
    price: number;
  }
  ```

  Plural collections: `Item[]`, never `Items` as a separate type.

- **Component props**: `type <Component>Props`. If the component is `Card`, the props are `CardProps`. Declared in `<name>.types.ts`, imported by the component file (§6 — types live in `<name>.types.ts`, never inlined next to the component).

  ```ts
  // features/cards/cards.types.ts
  export type CardProps = {
    onClick: () => void;
    title: string;
  };
  ```

  ```tsx
  // features/cards/components/card.tsx
  import type { CardProps } from "@/features/cards/cards.types";

  export function Card({ onClick, title }: CardProps) { /* ... */ }
  ```

- **Function param objects**: `type <Function>Params` in PascalCase. If the function is `calcItems`, the params type is `CalcItemsParams`. Same rule — declared in `<name>.types.ts`, imported by the util file.

  ```ts
  // features/items/items.types.ts
  export type CalcItemsParams = {
    discount: number;
    items: Item[];
  };
  ```

  ```ts
  // features/items/items.utils.ts
  import type { CalcItemsParams } from "@/features/items/items.types";

  export function calcItems({ discount, items }: CalcItemsParams): number { /* ... */ }
  ```

- Use `type` for everything except database row types and shapes designed for augmentation. Don't mix `type Foo` and `interface Foo` for the same concept.

## 9. Type generation from contracts

Don't hand-roll types that the backend already describes. Generate them.

- **Supabase tables / RPC**: `supabase gen types typescript --project-id <id> > shared/types/database.ts`. Re-run on every migration. Import row types as `Database['public']['Tables']['orders']['Row']`.
- **External APIs (OpenAPI 3.1)**: derive types via `openapi-typescript` into `shared/types/<service>.ts`.

Either way: any change to the contract goes to the source first (migration / OpenAPI spec), then regen. Never edit the generated file by hand.

---

## 10. Design system

Design tokens (colors, spacing, radii, shadows, typography, breakpoints) live in `tailwind.config.*` and the project's design system docs — not in this file. What this section enforces is how the app *consumes* the design system: token names only (never raw values), the a11y minimums below, and the iconography rules.

### Accessibility minimums

- Contrast: WCAG AA at minimum.
- Focus visible policy: explicit `:focus-visible` ring, never `outline: none` without replacement.
- Keyboard nav: every interactive element reachable via Tab; modal/menu traps focus correctly.
- Semantic HTML: `<button>` for actions, `<a>` for navigation, headings in order, `<label>` for inputs.
- ARIA only when semantic HTML can't express it.
- Reduced motion: respect `prefers-reduced-motion` on animations.

### Iconography

- Pick one library (commonly `lucide-react`, custom SVGR) and stick to it.
- Components live in `shared/icons/` (one file per icon, kebab-case filename, PascalCase export — `users.tsx` exports `Users`).
- Single size grid and stroke width across the app.
- Color follows current token (`currentColor`) unless explicitly themed.
- Decorative icons get `aria-hidden="true"`; meaningful icons get an accessible label.

---

## 11. Styling — Tailwind

- **Tailwind** is the styling system. No CSS modules, no styled-components, no inline `style={{...}}` for anything but truly dynamic values (computed widths, transforms driven by state).
- Tokens live in `tailwind.config.*` (see §10). The config is the single source of truth — components reference token names, never raw values.
- No magic numbers in className strings. Use scale steps (`p-md`, `gap-sm`) over `p-[13px]`. The `[arbitrary]` syntax is an escape hatch, not a default.
- **Spacing between siblings uses `gap` on the parent**, never `margin` (or `padding`) on the children. A flex/grid container with `gap-md` is the only correct way to space a list of elements — no `mt-*` / `mb-*` / `space-y-*` trick, no `:not(:last-child)` margin, no `pt-*` on the next sibling. Reserve `margin` for the rare case of pushing a single element away from a non-sibling boundary, and `padding` for inner box spacing.
- Dark mode: `class` strategy with a theme provider; respect system preference by default.
- Conditional classes: use `clsx` / `cn` helpers, not string concatenation. Pair with `tailwind-merge` to resolve conflicts when composing variants.
- Variants: use `cva` (class-variance-authority) for component variant APIs (`Button({ size, variant })`), not bespoke conditional chains.
- Sort classes with the official Prettier plugin (`prettier-plugin-tailwindcss`) so review diffs stay clean.
- Targets **Tailwind v4** (CSS-first config) by default. If the project is on v3, the JS `tailwind.config.*` is still the source of truth — note the version near the top of the config so onboarding is unambiguous.

---

## 12. Tests — selectors and where they live

### Selectors (a11y-first)

Tests should resemble real user perception. Selector priority:

1. **`getByRole` / accessible name** (preferred) — what assistive tech sees. Forces semantic HTML and ARIA correctness.
2. **`getByLabelText` / `getByText`** — what a user reads.
3. **`data-testid`** — escape hatch only. Use when the same role/text appears multiple times (lists with repeated rows) or when the element legitimately has no accessible name. Pattern: `<feature>-<element>-<role>` (e.g. `orders-row-validate`).

Don't add a `data-testid` "just in case". Reach for it only when role/text is genuinely ambiguous.

### Where tests live

Each feature owns one `tests/` folder containing both unit/integration tests and E2E flows:

```
features/<name>/tests/
├── <name>.test.ts       # unit / integration (Vitest or Jest + RTL)
└── <scenario>.spec.ts   # E2E Playwright flows
```

- **Unit / integration (`<name>.test.ts`)**: pure utils, hooks, schemas, isolated component renders. Runs in Node/JSDOM.
- **E2E (`<scenario>.spec.ts`)**: full user flows in a real browser with **Playwright**.

#### Cross-feature flows — ownership rule

A flow lives in the feature whose behavior the **last assertion** validates. Prerequisite steps (login, seed data, navigation through other features) are means, not ownership. A flow that logs in → opens catalog → adds to cart → reaches checkout success belongs to the **checkout** feature, even if it touches four others.

#### Example flow

```ts
// features/orders/tests/validate-order.spec.ts
import { expect, test } from "@playwright/test";

test("validates an order", async ({ page }) => {
  await page.goto("/");
  await page.getByRole("button", { name: "Cliente Juan Pérez" }).click();
  await page.getByRole("button", { name: "Validar pedido" }).click();
  await expect(page.getByText("Pedido confirmado")).toBeVisible();
});
```

---

## 13. Visual fidelity to designs

When implementing against a design mockup:

- The implemented UI must match the design.
- Hardcoded text matches the mockup copy character-for-character.
- Layout structure matches.
- Color/spacing tokens used match the design's tokens.
- If prose acceptance criteria contradict the image, the **image wins** — flag the contradiction before guessing.

---

## 14. Component states — every screen must declare them

For every component/screen, the spec must cover:

- **States**: idle, loading, error, empty, success — one paragraph per state describing what the user sees.
- **Components**: which primitives are used.
- **Copy**: exact strings, in the project's voice.
- **Microinteractions**: animations, focus order, error feedback.
- **Empty / loading / error states**: explicit, not implied. Use Suspense / `loading.tsx` / `error.tsx` boundaries on Next.js.
- **A11y notes**: contrast, focus, labels.

Implement every declared state — write one test assertion per state.

### Error boundaries

- Use Next.js `error.tsx` per route segment to catch render errors below it. Provide a "Try again" button calling the `reset` prop.
- For client components that can throw outside a route boundary (third-party widgets, ad-hoc lazy code), wrap them with `react-error-boundary`.
- Never silently swallow errors — log to Sentry (or the project's logger) and show a recoverable fallback.

---

## 15. Web-specific concerns

### Rendering & data fetching (Next.js)

- Prefer Server Components for data fetching. Use `fetch()` with appropriate `cache` / `revalidate` options.
- Server Actions for mutations triggered by the user. Validate inputs with `zod` (or your chosen validator) at the boundary.
- Client-side data fetching only when the data depends on browser state (auth tokens in localStorage, real-time data, etc.). Use `swr` or `@tanstack/react-query`.
- Never fetch on the client what the server can fetch on render — it costs an extra round-trip and a loading state.

### Performance

- Use `next/image` (or framework equivalent) for images. Always set `width`/`height` or `fill` to avoid CLS.
- Use `next/font` for fonts. Avoid `@import` from `<head>`.
- Lazy-load heavy client components with `dynamic(() => import(...), { ssr: false })` when they don't need SSR.
- Code-split at route boundaries automatically; check bundle size on PRs that pull in large deps.
- Core Web Vitals targets: LCP < 2.5s, CLS < 0.1, INP < 200ms.

### SEO

- Every public page declares `generateMetadata` (Next.js) or equivalent: `title`, `description`, `openGraph`, `twitter`, `canonical`.
- Sitemaps and `robots.txt` configured at deployment.

### Forms

- `react-hook-form` is the default form library.
- Validate with `zod`; share the schema between client and server (Server Action) when possible.
- Disable the submit button while submitting; show inline field errors, not just a banner.

---

## 16. State management

Pick the right tool per **kind** of state. The most common bug is treating server data as client state.

### Decision matrix

| Kind of state                             | Tool                              | Notes                                                                      |
| ----------------------------------------- | --------------------------------- | -------------------------------------------------------------------------- |
| Server data (anything from an API)        | **TanStack Query (React Query)**  | Caching, refetch, optimistic updates, invalidation. Don't mirror in a store. |
| Global UI state (theme, sidebar, modals)  | **Zustand** in `shared/stores/`   | Tiny, no boilerplate, no provider needed.                                  |
| Feature-scoped client state               | **Zustand** in `features/<name>/<name>.store.ts` | Lives with the feature. Don't promote to `shared/` unless another feature reads it. |
| Auth / user session                       | Supabase Auth (see §17)           | Cookie-based on Next.js via `@supabase/ssr`; read user in Server Components. |
| Form state                                | `react-hook-form` + `zod`         | Never put form values in a global store.                                   |
| URL-shaped state (filters, tabs, paging)  | URL `searchParams`                | Shareable, back-button friendly. Use `nuqs` for typed access.              |
| Local-only state                          | `useState` / `useReducer`         | Default. Don't reach for a store.                                          |
| Cross-component derived state             | `useMemo` + props, or Zustand     | Lift only when more than two siblings need it.                             |

### Why React Query (not Redux/RTK Query) by default

Server state has different semantics from client state — it's stale by default, can be refetched, can be invalidated by mutations, can be paginated/infinite. React Query encodes all of that. Putting `users`, `posts`, etc. in a Redux/Zustand store means re-implementing caching, deduping, and refetch logic by hand.

```ts
// features/orders/orders.api.ts
export function useOrders() {
  return useQuery({
    queryKey: ["orders"],
    queryFn: () => fetch("/api/orders").then((r) => r.json()),
  });
}
```

### Why Zustand over Redux Toolkit by default

Same store pattern, ~10x less boilerplate, no provider, works outside React (selectors callable from utils). Use Redux Toolkit when:

- The team is already deep in Redux and migration cost > benefit.
- You need redux-devtools time-travel debugging on complex state machines.
- You're standardizing on RTK Query and don't want React Query in the deps.

### Where stores live

- **Feature-scoped** (the default): `features/<name>/<name>.store.ts`. The store moves with the feature; no other feature imports it.
- **Global** (rare): `shared/stores/<domain>.store.ts`. Only when the state is genuinely cross-feature — `theme`, `auth-user`, `app-locale`. Don't pre-emptively lift; promote to `shared/` only when a second feature actually needs the state.

```ts
// features/cart/cart.store.ts
import { create } from "zustand";

type CartStore = {
  addItem: (id: string) => void;
  itemIds: string[];
  removeItem: (id: string) => void;
};

export const useCartStore = create<CartStore>((set) => ({
  addItem: (id) => set((s) => ({ itemIds: [...s.itemIds, id] })),
  itemIds: [],
  removeItem: (id) => set((s) => ({ itemIds: s.itemIds.filter((x) => x !== id) })),
}));
```

### Rules

- **Never mix server and client state in the same store.** If you need to override a server value optimistically, use React Query's `setQueryData` or `onMutate`, not a separate store copy.
- **One store per domain.** Feature stores in the feature; cross-feature stores in `shared/stores/`. Never one mega-store.
- **Selectors with `useStore(s => s.x)`** to avoid re-renders. Don't destructure the whole store in a component.
- **No business logic in components** — derived values go in selectors or utils.
- **Persist sparingly**: theme and other UI prefs only. **Auth lives in Supabase cookies (§17)**, not in a Zustand persisted store. Cached server data is React Query's job, not a store's.

### URL state with `nuqs`

Filters, tabs, sort, pagination — anything a user might want to share via URL or restore via back button — belongs in `searchParams`, not in a store. `nuqs` provides typed access:

```tsx
"use client";
import { parseAsString, useQueryState } from "nuqs";

export function OrdersFilter() {
  const [status, setStatus] = useQueryState("status", parseAsString.withDefault("open"));
  return (
    <select value={status} onChange={(e) => setStatus(e.target.value)}>
      <option value="open">Open</option>
      <option value="closed">Closed</option>
    </select>
  );
}
```

### Server-rendered apps (Next.js App Router)

- Prefer Server Components + Server Actions over client stores when possible.
- React Query still helps on the client for mutations and client-side refetches; hydrate from the server with `dehydrate`/`HydrationBoundary`.
- Don't push global client stores down through layouts unnecessarily — many apps need only `theme` + `auth` globally.

---

## 17. Backend — Supabase

Supabase is the backend for **auth, database, and storage**. All three flow through `@supabase/supabase-js` (and `@supabase/ssr` on Next.js App Router). Don't introduce a second auth provider, ORM, or blob store alongside it.

### Clients

- **Next.js App Router**: `@supabase/ssr` with three flavors — server client (Server Components / Server Actions / Route Handlers), browser client (`'use client'` components), middleware client (session refresh). Don't reuse one across boundaries.
- **Plain React (Vite, etc.)**: a single browser client from `@supabase/supabase-js`.
- Initialize the client in `shared/configs/supabase.ts` and import from there. Never construct ad-hoc clients in feature code.

### Auth

- Cookie-based sessions via `@supabase/ssr`. Read the user in Server Components / Server Actions; never pass the session through props from client to server.
- Protected routes: refresh the session in `middleware.ts` (project root) and authorize in the Server Component / Server Action. Middleware alone is not authorization.
- OAuth redirects: configure callback URL in Supabase dashboard + `app/auth/callback/route.ts` exchanging the code for a session.
- Sign-out invalidates on the server (Server Action) so cookies clear; don't rely on client-side `signOut()` alone.

### Database

- Queries live in `<feature>.api.ts`; React Query hooks in `<feature>.hooks.ts` consume them. Never call Supabase directly from a component.
- Generate TypeScript types from the schema (`supabase gen types typescript`) into `shared/types/database.ts`. Don't hand-roll row types.
- **Row Level Security is mandatory**. Every table has RLS enabled and policies covering every access pattern. The client uses the anon key — RLS is what protects data, not application code.
- Service role key is **server-only** (`SUPABASE_SERVICE_ROLE_KEY`, no `NEXT_PUBLIC_` prefix). Use it only in Server Actions / Route Handlers for operations that legitimately need to bypass RLS (admin tools, scheduled jobs).

### Storage

- Public buckets only when content is genuinely public (avatars on a public profile). Default to private buckets + signed URLs.
- Signed URLs generated server-side (Server Action / Route Handler) and passed to the client. Never expose service-role-signed URLs in client code.
- Use `next/image` with signed URLs; configure `remotePatterns` in `next.config.*` for the Supabase storage hostname.
- Uploads from the browser client are fine when RLS storage policies authorize them; otherwise upload through a Server Action.

### Realtime

- Subscribe inside React Query hooks; on event, call `queryClient.setQueryData` (or `invalidateQueries` for complex updates) so the cache stays the source of truth.
- Unsubscribe in the hook's cleanup. A leaked channel is a memory + bandwidth leak.

### Environment variables

- `NEXT_PUBLIC_SUPABASE_URL`, `NEXT_PUBLIC_SUPABASE_ANON_KEY` — safe in the browser.
- `SUPABASE_SERVICE_ROLE_KEY` — server-only. Never prefix with `NEXT_PUBLIC_`. Never log it.
- Local dev uses the Supabase CLI (`supabase start`) so the URL/anon key resolve to `http://localhost:54321`.

---

## 18. Security

### Secrets

- `NEXT_PUBLIC_*` ships to the browser. Anything else is server-only.
- Service role key (§17) only in Server Actions / Route Handlers. Never in `'use client'` files. Restrict imports of the service-role client via `eslint-no-restricted-imports` (or equivalent) so the build fails before deploy.
- Never log secrets or full PII; scrub in Sentry's `beforeSend`.

### Headers (configure in `next.config.*` or middleware)

- **CSP** (`Content-Security-Policy`): start with a strict default, allow only your origins + Supabase. Use nonces for inline scripts (App Router supports this).
- **HSTS** (`Strict-Transport-Security`): preload domain.
- **X-Frame-Options: DENY** (or CSP `frame-ancestors 'none'`) unless the app is intentionally embeddable.
- **Referrer-Policy: strict-origin-when-cross-origin**.
- **Permissions-Policy**: deny unused features (camera, geolocation, etc.).

### XSS

- Never `dangerouslySetInnerHTML` with user input. If markdown rendering is needed, sanitize with `DOMPurify` and run on a Server Component.
- Don't render user-controlled URLs in `<a href>` without protocol validation (`http`/`https`/`mailto` allowlist).

### CSRF

- Server Actions are CSRF-safe by default in Next.js (origin check). Don't disable it.
- Custom API routes that mutate state: require same-origin or a CSRF token.

### Open redirects

- Validate any `?redirect=` / `?next=` param against an allowlist before redirecting. Never `redirect(searchParams.get("next"))` as-is.

### Authorization

- Server enforces auth on every request. Middleware refresh ≠ authorization (§17).
- Every Server Action / Route Handler that reads or mutates data calls `supabase.auth.getUser()` and applies RLS — don't trust props/cookies blindly.

### Input validation

- Validate every Server Action / Route Handler input with Zod at the boundary. No exceptions.
- File uploads: validate MIME type, size, extension. Store with random names.

### Rate limiting

- Rate-limit Server Actions and Route Handlers that send email, mutate auth state, or create resources. Use Vercel Rate Limit / Upstash.

### Dependencies

- `pnpm audit` on CI; gate merges on no high/critical advisories.
- Dependabot or Renovate for automated PRs.
- Lockfile committed; never run with `--no-frozen-lockfile` in CI.

### Errors

- Production error pages don't expose stack traces or internal paths.
- Sentry captures full context server-side; client gets a generic message + correlation ID.

---

## 19. When to deviate

- If existing code violates one of these rules, follow the existing pattern locally and propose the convention change deliberately before fixing at scale. Don't drive-by refactor; drift fixes are a separate task.
- Routine compliance is the default. Deviating from a rule should be a recorded decision, not a quiet preference.

