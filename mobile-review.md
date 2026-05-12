# Frontend review checklist — Mobile

> Use this to audit React Native code against `frontend-mobile.md`. Implementers use that file to *build*; this file tells you how to *check* compliance and produce a review report. Section refs (§N) point at `frontend-mobile.md` unless stated otherwise.

---

## How to use this file

1. **Read the change description first.** What is this diff supposed to achieve? What's in scope and what isn't? If acceptance criteria exist (Gherkin format or otherwise), note them — every criterion needs at least one test assertion.
2. **Sanity-check the diff.** If the diff doesn't match the description, **stop and report a scope mismatch** instead of filing per-line findings. Don't review off-spec work.
3. **Walk the §1–§18 checks below in order.** Each check carries: default severity, rule source, what to verify, how to find violations, common false positives, and a resolution-hint template.
4. **Produce a review report** with a findings table (format at the bottom) and a one-paragraph summary of the verdict.

## Severity scale

| Severity | Meaning | Examples |
|---|---|---|
| `blocker` | Must fix before merge. Breaks a contract, security, runtime correctness, or compounds architectural debt. | Missing RLS, `any` cast, server data mirrored in store, no test for an acceptance criterion. |
| `non-blocker` | Should fix; doesn't gate merge. Convention drift or quality nit. | Imports not alphabetical, `accessibilityHint` missing, `[arbitrary]` Tailwind value. |
| `info` | Observation, not a defect. Use sparingly. | "Consider memoizing this list item if it grows." |

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

---

## §1 — File and folder naming

- **Default severity:** `non-blocker` (drift fix). Escalate to `blocker` if casing breaks an import path on a case-sensitive filesystem.
- **Rule:** §1.

**Verify**
- New/renamed files in `app/`, `features/`, `shared/` are kebab-case.
- Exported symbols keep their natural casing (PascalCase components, camelCase utils).
- Framework filenames keep their convention: `_layout.tsx`, `+not-found.tsx`, `[id].tsx`, `(tabs)/`, etc.

**How to find**
- Skim the file list of the diff. Anything ending in `.ts`/`.tsx` with PascalCase or camelCase outside `app/` and not in `shared/icons/` is suspect.
- Pattern: `git diff --name-only main...HEAD | grep -E '/[A-Z][^/]*\.(ts|tsx)$'`.

**False positives**
- Expo Router framework files in `app/`.
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
- Object literal keys (props, configs, style objects).
- Type / interface members.
- Destructured params: `function Foo({ a, b, c }: Props)`.
- Imports within each group. Group order: React → React Native → external → `@/` aliases → relative.

**Skip when order is semantic**
- Animation step arrays.
- Route precedence arrays.
- Intentional `StyleSheet` / className composition where overrides matter.

**How to find**
- Read the diff visually. Linters catch some cases, but mixed-case sort orders and rebase reorderings often slip through.

**Finding template**
```
| F-NNN | non-blocker | <file>:<line> | §2 | sort <symbol> alphabetically (or annotate the semantic reason in a one-line comment) |
```

## §3 — Named exports only

- **Default severity:** `blocker` (changes downstream import shape).
- **Rule:** §3.

**Verify**
- No `export default` outside Expo Router framework files.
- No default imports of project files (`import X from "@/features/..."`).
- Constants, hooks, utils, types — all named-exported.

**How to find**
- Pattern: `git diff main...HEAD -- 'features/**' 'shared/**' | grep -E '^\+.*export default'`. Any hit outside framework paths is a violation.
- Pattern: `git diff main...HEAD | grep -E '^\+import [A-Z][a-zA-Z]+ from "@/'`. Any hit is a default-import violation.

**False positives**
- `app/index.tsx`, `app/_layout.tsx`, `app/[id].tsx`, `app/+not-found.tsx`, `app/**/_layout.tsx`.

**Finding template**
```
| F-NNN | blocker | <file>:<line> | §3 | switch to `export function X` and update consumers to `import { X }` |
```

## §4 — Feature functions live in utils

- **Default severity:** `blocker` (architectural — propagates through the rest of the file shape).
- **Rule:** §4.

**Verify in every component file**
- No `const handleX = () => {...}` declarations.
- No `function onSubmit(values) {...}` inside a component body.
- Inline JSX arrows are dispatch sites — one expression that calls a util.
- Util names describe the **action** (`addItemToCart`, `submitOrderForm`), not the trigger (`handlePress`, `onClick`, `doStuff`).

**Anti-patterns to flag immediately**
- Identifiers starting with `handle`, `on` (when not a JSX prop), `do`, `process`.
- Closure-style helpers inside components capturing state — should be a util taking that state via params.
- Inline arrows that contain >1 expression (multi-line bodies, conditionals, awaits).

**How to find**
- Pattern: `git diff main...HEAD -- '**/*.tsx' | grep -E '^\+.*\bconst (handle|on|do)[A-Z]\w* ?='`.
- Pattern: `git diff main...HEAD -- '**/*.tsx' | grep -E '^\+.*function (handle|on|do)[A-Z]\w*\b'`.

**False positives**
- JSX prop bindings: `onPress={...}`, `onChangeText={...}` are React/RN conventions, not function names. Don't flag the prop *binding*; flag the *function definition* it points to.

**Cross-checks**
- A `handleX` violation in a component usually means §5 is also off (extra step before JSX).
- Likely also means a missing entry in `<feature>.utils.ts`.

**Finding template**
```
| F-NNN | blocker | <file>:<line> | §4 | extract `handleValidate` → `validateOrderById({ id, onSelect })` in <feature>.utils.ts; bind via inline arrow at JSX site |
```

## §5 — Component file structure

- **Default severity:** `non-blocker` (unless §4 violations are stacked, then escalate together).
- **Rule:** §5.

**Verify the 5-section order with one blank line between**
1. imports
2. component definition + props destructuring
3. variable declarations (`const`, `let`, destructuring)
4. `useEffect` / `useMemo` (only if present)
5. JSX render

**Common violations**
- Effect declared above variable declarations.
- Local function definition between component opening and JSX (covered by §4 too).
- Placeholder comments for empty sections (skip the section instead).
- `'use client'` is a Next.js concept and **does not apply** on mobile — flag if you see it.

**Finding template**
```
| F-NNN | non-blocker | <file> | §5 | reorder per §5: move <X> from section <N> to section <M>; remove placeholder comment |
```

## §6 — Feature module file roles

- **Default severity:** `blocker` for new files violating roles; `non-blocker` for stylistic refactors.
- **Rule:** §6.

**Verify**
- Every new file in `features/<name>/` matches the fixed role set:
  - `components/`, `tests/`, `<name>.api.ts`, `<name>.consts.ts`, `<name>.hooks.ts`, `<name>.schemas.ts`, `<name>.store.ts`, `<name>.types.ts`, `<name>.utils.ts`.
- No invented roles (`<name>.helpers.ts`, `<name>.lib.ts`, `<name>.service.ts` — all violations).
- Tests live in `features/<name>/tests/`. Unit tests as `<name>.test.ts`; E2E Maestro flows as `<scenario>.yaml`.
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

- **Default severity:** `blocker` if the diff causes type errors against the current schema; `non-blocker` if the generated file just lags an unrelated migration.
- **Rule:** §9.

**Verify**
- Migrations under `supabase/migrations/` in the diff trigger a regen of `shared/types/database.ts` — and that regen is in the same diff.
- No hand-edits to `shared/types/database.ts` or other generated files.
- OpenAPI source changes have a corresponding `openapi-typescript` regen.

**How to find**
- Cross-check `git diff --name-only main...HEAD` — if `supabase/migrations/*.sql` appears, `shared/types/database.ts` should also appear.

**Finding template**
```
| F-NNN | blocker | shared/types/database.ts | §9 | run `supabase gen types typescript --project-id <id> > shared/types/database.ts`, commit the regen, revert any hand-edits |
```

## §10 — Design system & a11y

- **Default severity:** `blocker` for a11y minimums; `non-blocker` for token consumption nits.
- **Rule:** §10.

**Verify (`blocker` items)**
- Every `Pressable` / interactive element has `accessibilityRole`, `accessibilityLabel`, and `accessibilityState` (when state-bearing).
- Touch target ≥ 44×44 pt iOS / 48×48 dp Android — use `hitSlop` for visually smaller targets.
- Decorative icons hidden from a11y (`accessibilityElementsHidden` iOS / `importantForAccessibility="no"` Android); meaningful icons have an `accessibilityLabel`.

**Verify (`non-blocker` items)**
- `accessibilityHint` for non-obvious actions.
- Animations check `AccessibilityInfo.isReduceMotionEnabled()`.
- Dynamic type respected unless explicit `allowFontScaling={false}` justified.
- Iconography: only `lucide-react-native`; files in `shared/icons/` (kebab file, PascalCase export); color from token, never hex.
- No raw color/spacing values in JSX — tokens from `tailwind.config.*`.

**How to find**
- Pattern: `git diff main...HEAD -- '*.tsx' | grep -E '^\+\s*<Pressable\b'` and check the surrounding lines for `accessibilityRole`/`accessibilityLabel`.
- Pattern: `git diff main...HEAD | grep -E '#[0-9a-fA-F]{3,8}\b'` for raw hex literals.

**Finding template (a11y)**
```
| F-NNN | blocker | <file>:<line> | §10 | add `accessibilityLabel="<copy>"` and `accessibilityRole="button"` (and `accessibilityState={{ disabled }}` when disabled) |
```

## §11 — NativeWind

- **Default severity:** `non-blocker`.
- **Rule:** §11.

**Verify**
- No `StyleSheet.create`, no `react-native-unistyles`, no inline `style={{...}}` (except Reanimated `useAnimatedStyle` and computed transforms).
- No `[arbitrary]` magic numbers in className strings — `p-md`, not `p-[13px]`.
- Conditional classes via `clsx`/`cn` + `tailwind-merge`, not string concat.
- Component variants via `cva`, not bespoke conditional chains.
- Classes sorted (Prettier plugin: confirm `prettier-plugin-tailwindcss` in `.prettierrc`).
- `SafeAreaView` / `useSafeAreaInsets()` on every screen root; no hardcoded `paddingTop: 44`.
- `Platform.select` only when there's a real platform difference.

**How to find**
- Pattern: `git diff main...HEAD | grep -E 'StyleSheet\.create'`.
- Pattern: `git diff main...HEAD | grep -E '\\[\\d+px\\]'` (arbitrary numeric values).
- Pattern: `git diff main...HEAD | grep -E 'paddingTop: \\d+'` for hardcoded safe-area padding.

**Finding template**
```
| F-NNN | non-blocker | <file>:<line> | §11 | replace `p-[13px]` with the closest scale step (`p-sm` or `p-md`); document the magic number only if a token genuinely doesn't fit |
```

## §12 — Tests

- **Default severity:** `blocker` for missing AC coverage; `non-blocker` for selector quality.
- **Rule:** §12.

**Verify**
- Each acceptance criterion has at least one test assertion. Map them: AC #1 → test name X. Missing coverage → one `blocker` finding per missing AC.
- Tests live in `features/<name>/tests/`. `<name>.test.ts` for unit/integration (Jest/Vitest + RNTL); `<scenario>.yaml` for E2E (Maestro).
- Selector priority: `accessibilityLabel` → `text` → `testID`. `testID` only when label/text is genuinely ambiguous (lists with repeated rows, elements without a meaningful label).
- E2E flow ownership: a flow lives in the feature whose **last assertion** validates behavior, regardless of preconditions (login, seed data).
- Every interactive element used in tests has a meaningful `accessibilityLabel` (also a §10 a11y requirement).
- On Android, `testID`-based selectors should be paired with `accessibilityLabel` (so screen readers still get a real string).

**How to find**
- Pattern: `git diff main...HEAD | grep -E '^\+.*testID="'` — confirm each is justified vs label/text.
- Pattern: `git diff main...HEAD -- '**/tests/*.yaml' | grep -E '^\+.*id: "'` — Maestro `id:` selector means `testID` was used; check whether `accessibilityText:` would have worked.

**Finding template (missing AC coverage)**
```
| F-NNN | blocker | features/<name>/tests/ | §12 + AC #N | add a Maestro flow asserting "Pedido confirmado" appears after a valid `Validar pedido` tap |
```

**Finding template (selector quality)**
```
| F-NNN | non-blocker | features/<name>/tests/<flow>.yaml:<line> | §12 | replace `id: "validate-button"` with `accessibilityText: "Validar pedido"` since the label is unique on this screen |
```

## §13 — Visual fidelity

- **Default severity:** `blocker` for layout/copy mismatch; `non-blocker` for minor spacing.
- **Rule:** §13.

**Verify**
- Hardcoded text matches mockup copy character-for-character.
- Layout structure matches the design.
- Tokens used match design tokens.
- Both **iOS and Android** verified — RN diverges (shadows, fonts, ripple, status bar, safe areas).
- If prose acceptance criteria contradict the image, the image wins (§13). Flag the contradiction in your summary, don't silently choose.

**How to find**
- This requires reading screenshots vs the design. If neither is available, file an `info` finding requesting screenshot evidence before approving.

**Finding template**
```
| F-NNN | blocker | <screen>.tsx | §13 | copy reads "Validar pedido" in mockup but "Validar" in code; update string to match mockup char-for-char |
```

## §14 — Component states

- **Default severity:** `blocker` if a declared state is missing.
- **Rule:** §14.

**Verify**
- For every screen/component in the diff: idle, loading, error, empty, success states implemented as declared in the spec.
- One test assertion per state.
- App root and each tab/stack root wrapped with `react-native-error-boundary` (or custom). Sentry/`sentry-expo` configured.
- Mutations and network errors handled via React Query `onError`, not the boundary.

**How to find**
- Cross-reference the spec's "Microinteractions / States" with the diff. Missing state → finding.

**Finding template**
```
| F-NNN | blocker | <screen>.tsx | §14 | implement the declared `empty` state ("No tienes pedidos todavía") with the corresponding test assertion |
```

## §15 — Mobile platform concerns

- **Default severity:** `blocker` (these are runtime correctness or perf regressions).
- **Rule:** §15.

**Verify**
- Lists > 20 items use `FlashList` with stable `keyExtractor` and `getItemType`. `FlatList` for small lists. **Never** `.map()` a long list inside a `ScrollView`.
- Animations on Reanimated 3 (`useSharedValue`, `useAnimatedStyle`); legacy `Animated` API banned.
- `react-native-gesture-handler` for gestures; `GestureHandlerRootView` wraps the app root.
- Images via `expo-image` (with `contentFit` + explicit dimensions), not `Image`.
- Permissions declared in `app.json` / `app.config.ts`; pre-prompt before the OS dialog.
- New native deps require an EAS Build profile documented in deployment docs.
- Memoize list item components (`React.memo`) and stable callbacks (`useCallback`) — list re-renders are the most common perf bug.
- Heavy gesture/animation work uses Reanimated worklets, not the JS thread.

**How to find**
- Pattern: `git diff main...HEAD | grep -E 'ScrollView'` and inspect surroundings — flag `.map(` inside.
- Pattern: `git diff main...HEAD | grep -E 'from "react-native".*Animated'` — flag legacy Animated imports.
- Pattern: `git diff main...HEAD -- '*.tsx' | grep -E '<Image\\b'` and confirm it's `expo-image`.

**Finding template**
```
| F-NNN | blocker | <file>:<line> | §15 | convert `<ScrollView>{items.map(...)}</ScrollView>` to `<FlashList data={items} keyExtractor={(o) => o.id} estimatedItemSize={64} renderItem={...} />` |
```

## §16 — State management

- **Default severity:** `blocker` (mixing layers is the most expensive mistake to undo later).
- **Rule:** §16.

**Verify**
- Server data goes through React Query (`useQuery` / `useInfiniteQuery`). Never mirrored into Zustand/Redux.
- React Query wired to `AppState` + `NetInfo` in the root provider — `focusManager.setFocused` and `onlineManager.setEventListener`. Without this, queries go stale on background and never recover.
- Auth user mirrored to `shared/stores/auth.store.ts` from `onAuthStateChange`.
- Form state in `react-hook-form` + Zod, never global.
- Persistence layered correctly: `expo-secure-store` for secrets/tokens, `mmkv`/`AsyncStorage` for UI prefs only, React Query persister with `maxAge` for server cache.
- Selectors used (`useStore(s => s.x)`); no destructure-the-whole-store anti-pattern.
- Hydrate before render for persisted state required on first paint (auth, theme) — splash/skeleton until rehydrated.

**How to find**
- Pattern: `git diff main...HEAD -- '**/*.store.ts' | grep -E '\\b(orders|users|posts|<server-entity>)\\b'` — server entities living in a store is a smell; verify whether they're truly client-state.
- Pattern: `git diff main...HEAD | grep -E 'AsyncStorage'` — flag any usage for tokens/secrets.

**Finding template**
```
| F-NNN | blocker | features/<name>/<name>.store.ts | §16 | remove `orders` from the Zustand store; consume `useOrders()` from `<feature>.hooks.ts`. If you need optimistic updates, use React Query `setQueryData`/`onMutate` |
```

## §17 — Backend (Supabase)

- **Default severity:** `blocker` (security and data integrity).
- **Rule:** §17.

**Verify**
- Every table touched by the diff has RLS enabled (`alter table <name> enable row level security;`) and policies covering every access pattern. Migrations in the diff include `enable row level security` and `create policy ...`.
- Service role key never imported by app code; only Edge Functions / server-side use it.
- Storage: private buckets default; signed URLs generated server-side (Edge Function), passed to the client.
- Realtime subscriptions tied to `useFocusEffect` and cleaned up on blur. A leaked channel = battery drain.
- Sign-out clears: `supabase.auth.signOut()` + `queryClient.clear()` + Zustand reset, on the same path.
- Auth storage adapter is `expo-secure-store` — never `AsyncStorage` (sessions contain refresh tokens).
- `detectSessionInUrl: false`, `flowType: 'pkce'` set on the client.
- Local dev `EXPO_PUBLIC_SUPABASE_URL` points at LAN IP, not `localhost`.

**How to find**
- Pattern: `git diff main...HEAD -- 'supabase/migrations/*.sql' | grep -E 'create table'` — every new table must have RLS enabled in the same migration.
- Pattern: `git diff main...HEAD -- 'supabase/migrations/*.sql' | grep -i 'service_role'` — flag any policy granting service_role to authenticated/anon contexts.
- Pattern: `git diff main...HEAD | grep -E 'SUPABASE_SERVICE_ROLE_KEY'` — must only appear in Edge Function code or server-only files.

**Finding template (RLS)**
```
| F-NNN | blocker | supabase/migrations/<file>.sql | §17 | add `alter table orders enable row level security;` and a `create policy "users_read_own_orders" on orders for select using (auth.uid() = user_id);` |
```

## §18 — Security

- **Default severity:** use security severity scale (`critical | high | medium | low | info`). Map: `critical` → must hotfix; `high` → blocks merge; `medium` → fix in same release; `low` → backlog; `info` → observation.
- **Rule:** §18.

**Verify**
- HTTPS only. `app.json` rejects cleartext: `NSAppTransportSecurity` (iOS), `usesCleartextTraffic: false` (Android).
- API keys: only `EXPO_PUBLIC_*` shipped. Service role key never bundled.
- Logging scrubs tokens, full headers, request bodies with credentials, full PII. Sentry `beforeSend` configured.
- Deep link params validated with Zod at the route handler. No auto-execution of actions from a deep link without user confirmation.
- Biometric gate (`expo-local-authentication`) on destructive flows (delete account, change email, large transfers). Re-auth before submit.
- Sensitive screens block screenshots/screen recording (`expo-screen-capture` `preventScreenCaptureAsync`); mask app contents on backgrounding for PCI/banking.
- WebViews: `originWhitelist` to your domains; `javaScriptEnabled: false` unless required; never inject user-controlled HTML/JS.
- Permissions: minimum necessary; pre-prompt explains why.
- Dependencies: `pnpm audit` on CI gates merges on no high/critical advisories. Lockfile committed; never `--no-frozen-lockfile` in CI. Dependabot/Renovate active.

**How to find**
- Pattern: `git diff main...HEAD -- 'app.json' 'app.config.ts'` and check for cleartext + permission declarations.
- Pattern: `git diff main...HEAD | grep -E 'console\\.(log|warn|error)'` and inspect each — token / PII in any log is `high`.
- Pattern: `git diff main...HEAD | grep -E 'Linking\\.openURL|expo-linking'` and confirm parsed params.
- Pattern: `git diff main...HEAD | grep -E 'WebView\\b'` and audit each WebView config.

**Finding template (deep link)**
```
| F-NNN | high | <handler>.ts:<line> | §18 | parse params with `LinkParamsSchema.parse(params)` (Zod) before use; reject malformed input. Never call `Linking.openURL(params.target)` without validation |
```

**Finding template (logging PII)**
```
| F-NNN | high | <file>:<line> | §18 | drop `console.log(user)` (logs PII). Use Sentry `beforeSend` to scrub if telemetry is needed |
```

---

## Cross-cutting smells (look for these spanning multiple §s)

- **`handleX` in component → §4 violation** + likely §5 reorder needed + missing entry in `<feature>.utils.ts`.
- **`any` cast at a Supabase response** → §8 + the boundary parse is missing → §17 (untyped data flowing into RLS-relevant code).
- **`testID` everywhere** → §12 (a11y-first selector violation) + likely §10 (missing `accessibilityLabel`).
- **New file in `features/<name>/` not in role allowlist** → §6 + check whether content belongs in `utils`/`hooks`.
- **List with `.map()` inside a `ScrollView`** → §15 perf + likely no `keyExtractor`.
- **Token mirrored to `AsyncStorage`** → §16 + §17 + §18 (security risk).

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

If no findings, write: *"No findings — change is ready to merge against `frontend-mobile.md`."*

Otherwise, use this format:

```
| ID | severity | location | rule | fix |
|---|---|---|---|---|
| F-001 | blocker | features/orders/components/order-card.tsx:23 | §4 | extract `handleValidate` to orders.utils.ts as `validateOrderById({ id, onSelect })`; bind via inline JSX arrow |
| F-002 | blocker | features/orders/orders.api.ts:14 | §8 | replace `as Order[]` with `OrderSchema.array().parse(data)` at the Supabase boundary |
| F-003 | non-blocker | features/orders/tests/validate-order.yaml:9 | §12 | replace `id: "validate-button"` with `accessibilityText: "Validar pedido"` since the label is unique |
| F-004 | high (security) | features/auth/auth.utils.ts:32 | §18 | drop `console.log({ session })` (logs refresh token); use Sentry `beforeSend` to scrub if telemetry is needed |
```

IDs are stable per review (`F-001`, `F-002`, …); never reuse a retired ID across re-reviews of the same change.

The implementer (or whoever owns the next iteration) reads the report, fixes the blockers, and resubmits. The reviewer's job ends when the report is delivered.
