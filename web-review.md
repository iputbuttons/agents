# Frontend review checklist — Web

> Use this to audit React web code (Next.js, plain React, Remotion) against `frontend-web.md`. Implementers use that file to *build*; this file tells you how to *check* compliance and produce a review report. Section refs (§N) point at `frontend-web.md` unless stated otherwise.

---

## How to use this file

1. **Read the change description first.** What is this diff supposed to achieve? What's in scope and what isn't? If acceptance criteria exist (Gherkin format or otherwise), note them — every criterion needs at least one test assertion.
2. **Sanity-check the diff.** If the diff doesn't match the description, **stop and report a scope mismatch** instead of filing per-line findings. Don't review off-spec work.
3. **Walk the §1–§18 checks below in order.** Each check carries: default severity, rule source, what to verify, how to find violations, common false positives, and a resolution-hint template.
4. **Produce a review report** with a findings table (format at the bottom) and a one-paragraph summary of the verdict.

## Severity scale

| Severity | Meaning | Examples |
|---|---|---|
| `blocker` | Must fix before merge. Breaks a contract, security, runtime correctness, or compounds architectural debt. | Missing RLS, `any` cast, server data mirrored in store, no test for an acceptance criterion, `dangerouslySetInnerHTML` of user input. |
| `non-blocker` | Should fix; doesn't gate merge. Convention drift or quality nit. | Imports not alphabetical, missing `aria-hidden` on a decorative icon, `[arbitrary]` Tailwind value. |
| `info` | Observation, not a defect. Use sparingly. | "Consider lazy-loading this client component if it grows." |

For §18 (Security), use the security severity scale (`critical | high | medium | low | info`) — they map roughly: `critical` ≈ `blocker`+, `medium` ≈ `non-blocker`.

## Stop conditions (report instead of line-by-line review)

Surface the issue in your summary and stop the §-by-§ walk when any apply:
- The diff doesn't address the stated change description.
- Acceptance criteria are missing, vague, or contradictory and the implementer guessed.
- The implementation is so far off-spec that line-by-line review wastes effort — recommend a redo with a tighter scope.

## Quick scope sanity-check (before any §-by-§ review)

- **AC coverage.** For every stated acceptance criterion, there should be at least one test assertion (unit or E2E). Each missing → one `blocker` finding.
- **Touched feature directories.** Cross-feature changes are a smell unless the change description calls for them. Note in summary if the diff touches >2 features without justification.
- **Generated files.** If the diff modifies `shared/types/database.ts` *by hand* (not via `supabase gen types`) — `blocker` immediately (§9).
- **Diff size.** Note in the summary if the diff is unusually large for the stated change — call out the cost-guardrail concern even if the code itself is clean.
- **Server/Client component boundaries.** If a previously-Server component became `'use client'`, verify it had to (state, effects, browser API, event handler). Otherwise a likely §5 regression.

---

## §1 — File and folder naming

- **Default severity:** `non-blocker` (drift fix). Escalate to `blocker` if casing breaks an import on a case-sensitive deploy target.
- **Rule:** §1 of `frontend-web.md`.

**Verify**
- New/renamed files in `app/`, `features/`, `shared/` are kebab-case.
- Exported symbols keep their natural casing (PascalCase components, camelCase utils).
- Framework filenames keep their convention: `page.tsx`, `layout.tsx`, `error.tsx`, `loading.tsx`, `not-found.tsx`, `route.ts`, `middleware.ts`, `_app.tsx` (pages router).

**How to find**
- Pattern: `git diff --name-only main...HEAD | grep -E '/[A-Z][^/]*\.(ts|tsx)$' | grep -v 'app/'`. Hits outside framework files are violations.

**False positives**
- Next.js framework files in `app/`.
- `shared/icons/<icon-name>.tsx` (kebab file, PascalCase export — file is fine).
- Auto-generated types.

**Finding template**
```
| F-NNN | non-blocker | <path> | §1 | rename to kebab-case (<old> → <new>); export keeps its casing |
```

## §2 — Alphabetical ordering

- **Default severity:** `non-blocker`.
- **Rule:** §2.

**Verify on every changed file**
- Object literal keys (props, configs).
- Type / interface members.
- Destructured params: `function Foo({ a, b, c }: Props)`.
- Imports within each group. Group order: React → external → `@/` aliases → relative.

**Skip when order is semantic**
- Animation step arrays.
- Route precedence (e.g. middleware match order).
- CSS cascade overrides where order matters.

**Finding template**
```
| F-NNN | non-blocker | <file>:<line> | §2 | sort <symbol> alphabetically (or annotate the semantic reason in a one-line comment) |
```

## §3 — Named exports only

- **Default severity:** `blocker` (changes downstream import shape).
- **Rule:** §3.

**Verify**
- No `export default` outside Next.js framework files (`page.tsx`, `layout.tsx`, `error.tsx`, `loading.tsx`, `not-found.tsx`, route handlers, `middleware.ts`, `app/auth/callback/route.ts`).
- No default imports of project files (`import X from "@/features/..."`).
- Constants, hooks, utils, types — all named-exported.

**How to find**
- Pattern: `git diff main...HEAD -- 'features/**' 'shared/**' | grep -E '^\+.*export default'`. Any hit outside framework paths is a violation.
- Pattern: `git diff main...HEAD | grep -E '^\+import [A-Z][a-zA-Z]+ from "@/'`. Any hit is a default-import violation.

**False positives**
- All `app/**/page.tsx`, `app/**/layout.tsx`, `app/**/error.tsx`, `app/**/loading.tsx`, `app/**/route.ts`, `middleware.ts`.

**Finding template**
```
| F-NNN | blocker | <file>:<line> | §3 | switch to `export function X` and update consumers to `import { X }` |
```

## §4 — Feature functions live in utils

- **Default severity:** `blocker` (architectural).
- **Rule:** §4.

**Verify in every component file**
- No `const handleX = () => {...}` declarations.
- No `function onSubmit(values) {...}` inside a component body.
- Inline JSX arrows are dispatch sites — one expression that calls a util.
- Util names describe the **action** (`addItemToCart`, `submitOrderForm`), not the trigger (`handleClick`, `onClick`, `doStuff`).

**Anti-patterns to flag immediately**
- Identifiers starting with `handle`, `on` (when not a JSX prop), `do`, `process`.
- Closure-style helpers inside components capturing state — should be a util taking that state via params.
- Inline arrows that contain >1 expression (multi-line bodies, conditionals, awaits).
- Server Action handlers defined inline in a Server Component — should live in `<feature>.api.ts`.

**How to find**
- Pattern: `git diff main...HEAD -- '**/*.tsx' | grep -E '^\+.*\bconst (handle|on|do)[A-Z]\w* ?='`.
- Pattern: `git diff main...HEAD -- '**/*.tsx' | grep -E '^\+.*function (handle|on|do)[A-Z]\w*\b'`.

**False positives**
- JSX prop bindings: `onClick={...}`, `onChange={...}` are React conventions, not function names. Don't flag the prop *binding*; flag the *function definition* it points to.

**Cross-checks**
- A `handleX` violation in a component usually means §5 is also off (extra step before JSX).
- Likely also means a missing entry in `<feature>.utils.ts` (or `<feature>.api.ts` for Server Actions).

**Finding template**
```
| F-NNN | blocker | <file>:<line> | §4 | extract `handleSubmit` → `submitLoginForm({ email, password, router })` in <feature>.utils.ts; bind via inline arrow at JSX site |
```

## §5 — Component file structure

- **Default severity:** `non-blocker` (unless §4 violations are stacked; escalate together).
- **Rule:** §5.

**Verify the 5-section order with one blank line between**
1. imports (`'use client'` directive — if present — sits *above* imports).
2. component definition + props destructuring.
3. variable declarations (`const`, `let`, destructuring).
4. `useEffect` / `useMemo` (only if present).
5. JSX render.

**Server / Client component boundary checks**
- `'use client'` only on components that genuinely need state, effects, browser APIs, or event handlers.
- Push client boundaries down to interactive leaves; don't wrap whole pages.
- Server components must not be imported by client components (Next.js compile error). Verify the data flow goes Server → Client via props.
- A page that became `'use client'` to "fix" a small interaction is a likely architecture regression.

**How to find**
- Pattern: `git diff main...HEAD -- 'app/**/page.tsx' | grep -E '^\+.*"use client"'` — every new `'use client'` page is a smell; verify justification.

**Finding template**
```
| F-NNN | blocker | <file> | §5 | move `'use client'` from `<page>.tsx` to the leaf interactive component; keep the page a Server Component to allow server fetching |
```

## §6 — Feature module file roles

- **Default severity:** `blocker` for new files violating roles; `non-blocker` for stylistic refactors.
- **Rule:** §6.

**Verify**
- Every new file in `features/<name>/` matches the fixed role set:
  - `components/`, `tests/`, `<name>.api.ts`, `<name>.consts.ts`, `<name>.hooks.ts`, `<name>.schemas.ts`, `<name>.store.ts`, `<name>.types.ts`, `<name>.utils.ts`.
- No invented roles (`<name>.helpers.ts`, `<name>.lib.ts`, `<name>.service.ts` — all violations).
- Tests live in `features/<name>/tests/`. Unit tests as `<name>.test.ts`; E2E Playwright flows as `<scenario>.spec.ts`.
- Cross-feature E2E flows live in the feature whose **last assertion** validates the behavior (per §12). Login/seed steps don't determine ownership.
- Feature stores (`<name>.store.ts`) are owned by the feature; no other feature imports them.
- Global stores live in `shared/stores/<domain>.store.ts` — only when genuinely cross-feature (theme, auth, locale).

**How to find**
- New files in feature dirs: `git diff --name-only --diff-filter=A main...HEAD -- 'features/**'`.
- Any file matching `features/<name>/<name>.<role>.ts` where role isn't in the allowlist → violation.

**Finding template**
```
| F-NNN | blocker | <file> | §6 | merge `<feature>.helpers.ts` content into `<feature>.utils.ts` (or split into the appropriate role file); delete the helper file |
```

## §7 — Path aliases

- **Default severity:** `non-blocker`.
- **Rule:** §7.

**Verify**
- Cross-feature imports use `@/features/...`, `@/shared/...`, `@/app/...`.
- Relative imports (`./`, `../`) only inside the same module.
- No deep relative paths across features (`../../other-feature/...`).

**How to find**
- Pattern: `git diff main...HEAD | grep -E '^\+import .* from "(\\.\\./){2,}'`.

**Finding template**
```
| F-NNN | non-blocker | <file>:<line> | §7 | rewrite `../../<other>/x` as `@/features/<other>/x` |
```

## §8 — TypeScript strict + escape-hatch ban

- **Default severity:** `blocker`.
- **Rule:** §8.

**Verify**
- `tsconfig.json`: `strict: true`, `noUncheckedIndexedAccess`, `noImplicitOverride`, `noFallthroughCasesInSwitch` all on.
- **No `any`, no `unknown`, no `as` casts** in feature code. Only `as const` permitted.
- Untyped data validated at boundaries with Zod (`Schema.parse(...)` returns the typed value). No manual narrowing via `as`.
- Functions exported across feature boundaries declare explicit return types.
- Types live in `<name>.types.ts` and are imported (per §6); not inlined next to the component.

**How to find**
- Pattern: `git diff main...HEAD -- '*.ts' '*.tsx' | grep -E '^\+.*\b(any|unknown)\b'` (filter out comments/strings).
- Pattern: `git diff main...HEAD | grep -E '^\+.* as [A-Z]'` (then exclude `as const`).
- Pattern: `git diff main...HEAD | grep -E '^\+.*: any\b'`.

**False positives**
- `any` inside a string literal or comment.
- `as const` (allowed).
- Generated files in `shared/types/`.
- Library type imports.

**Finding template**
```
| F-NNN | blocker | <file>:<line> | §8 | replace `as <Type>` with a Zod parse at the boundary, or narrow with a `typeof`/`in` guard. `any`/`unknown`/`as` not permitted in feature code (only `as const`) |
```

## §9 — Generated types

- **Default severity:** `blocker` if the diff causes type errors; `non-blocker` if the generated file lags an unrelated migration.
- **Rule:** §9.

**Verify**
- Migrations under `supabase/migrations/` in the diff trigger a regen of `shared/types/database.ts` — and that regen is in the same diff.
- No hand-edits to generated files.
- OpenAPI source changes have a corresponding `openapi-typescript` regen.

**Finding template**
```
| F-NNN | blocker | shared/types/database.ts | §9 | run `supabase gen types typescript --project-id <id> > shared/types/database.ts`, commit the regen, revert any hand-edits |
```

## §10 — Design system & a11y

- **Default severity:** `blocker` for a11y minimums; `non-blocker` for token consumption nits.
- **Rule:** §10.

**Verify (`blocker` items)**
- Semantic HTML: `<button>` for actions, `<a>` for navigation, headings in order, `<label>` for inputs.
- Every interactive element is keyboard-reachable via Tab; modal/menu traps focus correctly.
- Explicit `:focus-visible` style — never `outline: none` without replacement.
- Decorative icons get `aria-hidden="true"`; meaningful icons get an accessible label.

**Verify (`non-blocker` items)**
- Contrast WCAG AA at minimum (verify against design tokens).
- ARIA only when semantic HTML can't express it.
- Animations respect `prefers-reduced-motion`.
- Iconography: one library (e.g. `lucide-react`); files in `shared/icons/` (kebab file, PascalCase export); color via `currentColor` unless explicitly themed.
- No raw color/spacing values in JSX/className — tokens from `tailwind.config.*`.

**How to find**
- Pattern: `git diff main...HEAD | grep -E 'outline: none|outline-none'` — flag if no `:focus-visible` replacement nearby.
- Pattern: `git diff main...HEAD | grep -E '#[0-9a-fA-F]{3,8}\b'` for raw hex literals.
- Pattern: `git diff main...HEAD -- '*.tsx' | grep -E '<div [^>]*onClick='` — divs with click handlers should usually be `<button>`.

**Finding template (a11y)**
```
| F-NNN | blocker | <file>:<line> | §10 | replace `<div onClick={...}>` with `<button type="button" onClick={...}>` and add `aria-label="<copy>"` if the visible text is icon-only |
```

## §11 — Tailwind

- **Default severity:** `non-blocker`.
- **Rule:** §11.

**Verify**
- No CSS modules, no styled-components, no inline `style={{...}}` (except truly dynamic computed values).
- No `[arbitrary]` magic numbers in className strings — `p-md`, not `p-[13px]`.
- Conditional classes via `clsx`/`cn` + `tailwind-merge`, not string concat.
- Component variants via `cva`, not bespoke conditional chains.
- Classes sorted (Prettier plugin: confirm `prettier-plugin-tailwindcss` in `.prettierrc`).
- Dark mode: `class` strategy with theme provider; respects system preference by default.
- Tailwind v4 (CSS-first config) by default; v3 only if the project explicitly opts in (note at top of config).

**How to find**
- Pattern: `git diff main...HEAD | grep -E 'styled-components\\|@emotion\\|.module\\.css'` — flag any.
- Pattern: `git diff main...HEAD | grep -E '\\[\\d+px\\]'` (arbitrary numeric values).

**Finding template**
```
| F-NNN | non-blocker | <file>:<line> | §11 | replace `p-[13px]` with the closest scale step (`p-sm` or `p-md`); document the magic number only if a token genuinely doesn't fit |
```

## §12 — Tests

- **Default severity:** `blocker` for missing AC coverage; `non-blocker` for selector quality.
- **Rule:** §12.

**Verify**
- Each acceptance criterion has at least one test assertion. Map them: AC #1 → test name X. Missing coverage → one `blocker` finding per missing AC.
- Tests live in `features/<name>/tests/`. `<name>.test.ts` for unit/integration (Vitest/Jest + RTL); `<scenario>.spec.ts` for E2E (Playwright).
- Selector priority: `getByRole` → `getByLabelText`/`getByText` → `data-testid`. `data-testid` only when role/text is genuinely ambiguous (lists with repeated rows, elements without an accessible name).
- E2E flow ownership: a flow lives in the feature whose **last assertion** validates behavior, regardless of preconditions.
- Server Action / Route Handler endpoints have at least one integration test asserting auth + Zod validation behavior (404/401/422 paths).

**How to find**
- Pattern: `git diff main...HEAD | grep -E '^\+.*data-testid='` — confirm each is justified vs role/text.
- Pattern: `git diff main...HEAD -- '**/tests/*.spec.ts' | grep -E 'getByTestId'` — same check at the test side.

**Finding template (missing AC coverage)**
```
| F-NNN | blocker | features/<name>/tests/ | §12 + AC #N | add a Playwright spec asserting "Pedido confirmado" appears after a valid `Validar pedido` click |
```

**Finding template (selector quality)**
```
| F-NNN | non-blocker | features/<name>/tests/<flow>.spec.ts:<line> | §12 | replace `getByTestId('validate-button')` with `getByRole('button', { name: 'Validar pedido' })` since the role+name uniquely identifies the element |
```

## §13 — Visual fidelity

- **Default severity:** `blocker` for layout/copy mismatch; `non-blocker` for minor spacing.
- **Rule:** §13.

**Verify**
- Hardcoded text matches mockup copy character-for-character.
- Layout structure matches the design.
- Tokens used match design tokens.
- Responsive breakpoints implemented per spec.
- If prose acceptance criteria contradict the image, the image wins (§13). Flag the contradiction in your summary, don't silently choose.

**How to find**
- Read screenshots vs design. If neither is available, file an `info` finding requesting screenshot evidence before approving.

**Finding template**
```
| F-NNN | blocker | <component>.tsx | §13 | copy reads "Validar pedido" in mockup but "Validar" in code; update string to match mockup char-for-char |
```

## §14 — Component states

- **Default severity:** `blocker` if a declared state is missing.
- **Rule:** §14.

**Verify**
- For every screen/component in the diff: idle, loading, error, empty, success implemented as declared in the spec.
- One test assertion per state.
- Each route segment has an `error.tsx` boundary with a "Try again" button calling the `reset` prop.
- Suspense / `loading.tsx` boundaries used appropriately on Server Components.
- For client components that can throw outside a route boundary (third-party widgets, lazy code), `react-error-boundary` wrapping.
- Errors are logged to Sentry (or project logger) — never silently swallowed.

**How to find**
- Cross-reference the spec's "Microinteractions / States" with the diff. Missing state → finding.
- Pattern: every new route directory (`app/<route>/`) should have an `error.tsx` and `loading.tsx` (or inherit from layout).

**Finding template**
```
| F-NNN | blocker | app/<route>/ | §14 | add `error.tsx` with a "Try again" button calling `reset()`; implement the declared `empty` state with corresponding test assertion |
```

## §15 — Web platform concerns

- **Default severity:** `blocker` for perf/SSR regressions; `non-blocker` for SEO meta nits.
- **Rule:** §15.

**Verify**
- Data fetching: Server Components by default with `fetch()` + `cache`/`revalidate`. Client-side fetching only when data depends on browser state.
- Server Actions for mutations; inputs validated with Zod at the boundary.
- Never fetch on the client what the server can fetch on render.
- Images: `next/image` with explicit `width`/`height` or `fill` (no CLS).
- Fonts: `next/font` (no `@import` in `<head>`).
- Heavy client components lazy-loaded with `dynamic(..., { ssr: false })` when SSR isn't needed.
- Core Web Vitals targets respected: LCP < 2.5s, CLS < 0.1, INP < 200ms.
- Public pages declare `generateMetadata` (title, description, openGraph, twitter, canonical).
- Forms use `react-hook-form` + Zod; submit button disabled while submitting; inline field errors.

**How to find**
- Pattern: `git diff main...HEAD -- '*.tsx' | grep -E '<img\\b'` — should be `<Image />` from `next/image`.
- Pattern: `git diff main...HEAD | grep -E '@import url'` — flag `@import` in CSS/HTML.
- Pattern: `git diff main...HEAD -- 'app/**/page.tsx' | grep -E 'use client.*useEffect.*fetch'` — client-side fetching that could be server-side.

**Finding template**
```
| F-NNN | blocker | <file>:<line> | §15 | replace `<img src={url} />` with `<Image src={url} width={400} height={300} alt="..." />` from `next/image`; configure `remotePatterns` if the host is external |
```

## §16 — State management

- **Default severity:** `blocker`.
- **Rule:** §16.

**Verify**
- Server data goes through React Query (`useQuery` / `useInfiniteQuery`) or Server Components — never mirrored into Zustand/Redux.
- Auth user read via `@supabase/ssr` server client in Server Components / Server Actions; never passed via props from client to server.
- Form state in `react-hook-form` + Zod, never in a global store.
- URL-shaped state (filters, tabs, paging) lives in `searchParams` (typed access via `nuqs`) — not in a store.
- Persisted client state limited to UI prefs (theme). Auth lives in Supabase cookies (§17), not in a Zustand persisted store. Cached server data is React Query's job.
- Selectors used (`useStore(s => s.x)`); no destructure-the-whole-store.
- Server-rendered apps prefer Server Components + Server Actions over client stores; React Query hydrates from server with `dehydrate`/`HydrationBoundary`.

**How to find**
- Pattern: `git diff main...HEAD -- '**/*.store.ts' | grep -E '\\b(orders|users|posts|<server-entity>)\\b'` — server entities in a store is a smell.
- Pattern: `git diff main...HEAD | grep -E 'useState.*searchParam'` — likely should be `nuqs`.

**Finding template**
```
| F-NNN | blocker | features/<name>/<name>.store.ts | §16 | remove `orders` from the Zustand store; consume `useOrders()` from `<feature>.hooks.ts`. For optimistic updates, use React Query `setQueryData`/`onMutate` |
```

## §17 — Backend (Supabase)

- **Default severity:** `blocker` (security and data integrity).
- **Rule:** §17.

**Verify**
- Every table touched by the diff has RLS enabled and policies covering every access pattern. Migrations include `enable row level security` and `create policy ...`.
- Service role key (`SUPABASE_SERVICE_ROLE_KEY`) is **server-only** — never `NEXT_PUBLIC_` prefixed, never logged, never imported by `'use client'` files.
- `@supabase/ssr` clients used correctly: server client (Server Components / Server Actions / Route Handlers), browser client (`'use client'` components), middleware client (session refresh). No reuse across boundaries.
- Auth: cookie-based via `@supabase/ssr`. User read in Server Components; never passed via props from client to server.
- `middleware.ts` refreshes the session; **authorization** still happens in the Server Component / Server Action — middleware refresh is not authorization.
- OAuth callback at `app/auth/callback/route.ts` exchanges the code for a session.
- Sign-out invalidates server-side (Server Action), so cookies clear.
- Storage: private buckets default; signed URLs server-side; `next/image` `remotePatterns` configured for the Supabase storage host.
- Realtime subscriptions cleaned up in hook cleanup.
- Local dev: `supabase start` → `http://localhost:54321`.

**How to find**
- Pattern: `git diff main...HEAD -- 'supabase/migrations/*.sql' | grep -E 'create table'` — every new table must have RLS enabled in the same migration.
- Pattern: `git diff main...HEAD -- 'features/**/*.tsx' | grep -E 'SUPABASE_SERVICE_ROLE_KEY'` — any client-component import is `critical`.
- Pattern: `git diff main...HEAD | grep -E 'createServerClient|createBrowserClient|createClient'` — verify the right factory at the right boundary.

**Finding template (RLS)**
```
| F-NNN | blocker | supabase/migrations/<file>.sql | §17 | add `alter table orders enable row level security;` and a `create policy "users_read_own_orders" on orders for select using (auth.uid() = user_id);` |
```

**Finding template (service role)**
```
| F-NNN | critical | <file>:<line> | §17 + §18 | service role key imported in a `'use client'` file. Move all service-role usage to a Server Action / Route Handler; restrict imports via `eslint-no-restricted-imports` |
```

## §18 — Security

- **Default severity:** use security severity scale (`critical | high | medium | low | info`).
- **Rule:** §18.

**Secrets**
- `NEXT_PUBLIC_*` ships to browser; everything else is server-only.
- Service role key (§17) only in Server Actions / Route Handlers. Never in `'use client'` files. Restricted via `eslint-no-restricted-imports` (or equivalent).
- No secrets or full PII in logs (Sentry `beforeSend` scrubs).

**Headers (in `next.config.*` or middleware)**
- **CSP** with strict default; nonces for inline scripts.
- **HSTS** with preload.
- **X-Frame-Options: DENY** (or CSP `frame-ancestors 'none'`) unless intentionally embeddable.
- **Referrer-Policy: strict-origin-when-cross-origin**.
- **Permissions-Policy**: deny unused features.

**XSS**
- Never `dangerouslySetInnerHTML` with user input. If markdown is needed, sanitize with `DOMPurify` and run on a Server Component.
- User-controlled URLs in `<a href>` validated against an allowlist (`http`/`https`/`mailto`).

**CSRF**
- Server Actions are CSRF-safe by default (origin check). Don't disable.
- Custom mutating routes require same-origin or CSRF token.

**Open redirects**
- `?redirect=` / `?next=` validated against an allowlist. Never `redirect(searchParams.get("next"))` raw.

**Authorization**
- Every Server Action / Route Handler that reads or mutates data calls `supabase.auth.getUser()` and applies RLS. Don't trust props/cookies blindly.

**Input validation**
- Every Server Action / Route Handler input validated with Zod at the boundary.
- File uploads: validate MIME, size, extension; store with random names.

**Rate limiting**
- Sensitive endpoints (email, auth mutations, resource creation) rate-limited (Vercel Rate Limit / Upstash).

**Errors**
- Production error pages don't expose stack traces or internal paths.
- Sentry server-side; client gets generic message + correlation ID.

**Dependencies**
- `pnpm audit` on CI; gate on no high/critical advisories.
- Dependabot/Renovate active.
- Lockfile committed; never `--no-frozen-lockfile` in CI.

**How to find**
- Pattern: `git diff main...HEAD | grep -E 'dangerouslySetInnerHTML'` — every hit needs justification + sanitization.
- Pattern: `git diff main...HEAD | grep -E 'redirect\\(.*searchParams'` — flag if not allowlist-validated.
- Pattern: `git diff main...HEAD -- 'app/**/route.ts' 'app/**/actions.ts'` — every handler should `getUser()` + parse body with Zod.
- Pattern: `git diff main...HEAD | grep -E 'console\\.(log|warn|error)'` and inspect each — token / PII in any log is `high`.
- Pattern: `git diff main...HEAD -- 'next.config.*' 'middleware.ts'` — verify CSP/HSTS/etc. headers present.

**Finding template (XSS)**
```
| F-NNN | high | <file>:<line> | §18 | drop `dangerouslySetInnerHTML={{ __html: userInput }}`. If markdown rendering is required, sanitize with `DOMPurify` in a Server Component, or render via `react-markdown` with the default sanitizer |
```

**Finding template (open redirect)**
```
| F-NNN | high | <handler>.ts:<line> | §18 | validate `next` against an allowlist before `redirect(next)`. Reject any value not matching `^/<allowed-prefix>/`. Never trust `searchParams` raw |
```

**Finding template (missing auth check)**
```
| F-NNN | critical | <handler>.ts:<line> | §18 | add `const { data: { user } } = await supabase.auth.getUser(); if (!user) return new Response("Unauthorized", { status: 401 });` before any data access. Middleware refresh is not authorization |
```

---

## Cross-cutting smells (look for these spanning multiple §s)

- **`handleX` in component → §4 violation** + likely §5 reorder needed + missing entry in `<feature>.utils.ts` (or `<feature>.api.ts` for Server Actions).
- **`any` cast at a Supabase response** → §8 + the boundary parse is missing → §17 (untyped data flowing into RLS-relevant code).
- **`data-testid` everywhere** → §12 (a11y-first selector violation) + likely §10 (missing accessible name on the element).
- **New `'use client'` page** → §5 boundary regression + likely §15 (server-side fetch turned into client-side).
- **Service role key in any `'use client'` file** → §17 + §18 (`critical`).
- **Server Action without `getUser()` + Zod parse** → §17 + §18 (`critical` if the action mutates).
- **`redirect(searchParams.get("next"))` without allowlist** → §18 (`high`) open-redirect.
- **`dangerouslySetInnerHTML` with anything not provably static** → §18 (`high`) XSS risk.

---

## Producing the review report

After walking the checks, produce a single markdown report containing two sections:

### 1. Summary (one paragraph)

State plainly:
- What was reviewed (scope, file count, lines).
- Verdict: `clean` / `N findings open` / `scope mismatch — recommend redo`.
- Top blockers if any (by ID).
- Any cross-cutting concern from the stop conditions / scope sanity-check.

### 2. Findings table

If no findings, write: *"No findings — change is ready to merge against `frontend-web.md`."*

Otherwise, use this format:

```
| ID | severity | location | rule | fix |
|---|---|---|---|---|
| F-001 | blocker | features/orders/components/order-card.tsx:23 | §4 | extract `handleSubmit` to orders.utils.ts as `submitOrderForm({ values, router })`; bind via inline JSX arrow |
| F-002 | blocker | features/orders/orders.api.ts:14 | §8 | replace `as Order[]` with `OrderSchema.array().parse(data)` at the Supabase boundary |
| F-003 | non-blocker | features/orders/tests/validate-order.spec.ts:12 | §12 | replace `getByTestId('validate-button')` with `getByRole('button', { name: 'Validar pedido' })` |
| F-004 | critical (security) | app/api/orders/route.ts:8 | §18 | add `getUser()` auth check + `OrderInputSchema.parse(body)` before any DB access; middleware refresh is not authorization |
| F-005 | high (security) | app/dashboard/page.tsx:42 | §18 | drop `dangerouslySetInnerHTML={{ __html: comment.body }}`; render via `react-markdown` with default sanitization |
```

IDs are stable per review (`F-001`, `F-002`, …); never reuse a retired ID across re-reviews of the same change.

The implementer (or whoever owns the next iteration) reads the report, fixes the blockers, and resubmits. The reviewer's job ends when the report is delivered.
