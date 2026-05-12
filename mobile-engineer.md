# Frontend patterns and best practices — Mobile

> Conventions for **React Native (Expo or bare RN)**. Drop this file into a project as context for Claude Code; it's self-contained.

---

## 0. Skills

This agent delegates specialized work to external skills via the Skill tool. If a skill is not installed in the user's environment, continue without it — never block on a missing skill.

### Install (one-time setup)

Run these once per project (or globally) to make the skills available:

```bash
pnpx skills add expo/skills
pnpx skills add https://github.com/anthropics/skills --skill frontend-design
pnpx skills add https://github.com/nextlevelbuilder/ui-ux-pro-max-skill --skill ui-ux-pro-max
pnpx skills add https://github.com/supabase/agent-skills --skill supabase
pnpx skills add https://github.com/supabase/agent-skills --skill supabase-postgres-best-practices
pnpx skills add https://github.com/vercel-labs/agent-skills --skill vercel-react-native-skills
pnpx skills add https://github.com/sleekdotdesign/agent-skills --skill sleek-design-mobile-apps
pnpx skills add https://github.com/pbakaus/impeccable --skill animate
pnpx skills add https://github.com/pbakaus/impeccable --skill colorize
pnpx skills add https://cli.sentry.dev
```

### When to invoke each skill

| Skill | Invoke when |
|---|---|
| `expo` | Touching any Expo SDK module — `expo-image`, `expo-secure-store`, `expo-linking`, `expo-local-authentication`, `expo-screen-capture`, `expo-haptics`, `expo-router`, EAS Build profiles, `app.json` / `app.config.ts`. |
| `frontend-design` | Designing a new component or screen before writing JSX — variants, states, copy, layout. |
| `ui-ux-pro-max` | Polishing UX details: motion, focus order, error feedback, microinteractions. |
| `sleek-design-mobile-apps` | Mobile-first visual design — spacing, typography rhythm, native-feel patterns (iOS vs Android). |
| `supabase` | Any Supabase work — client setup (`shared/configs/supabase.ts`), auth flows, storage, realtime, Edge Functions (§17). |
| `supabase-postgres-best-practices` | Writing or reviewing migrations, RLS policies, indexes, triggers, or any raw SQL. |
| `vercel-react-native-skills` | RN-specific implementation: `FlashList`, Reanimated 3, `react-native-gesture-handler`, performance profiling, EAS Build (§15). |
| `animate` | Designing or implementing animations and motion — easing, timing, choreography, Reanimated worklets. |
| `colorize` | Choosing or refining color palettes, contrast, and theming decisions. |
| `sentry-cli` | Configuring `sentry-expo`, uploading source maps, tagging releases, scrubbing PII via `beforeSend` (§14, §18). |

When multiple skills apply to the same task, invoke them broadest-to-narrowest — e.g. `frontend-design` for the component shape, then `sleek-design-mobile-apps` for mobile polish, then `vercel-react-native-skills` for RN-specific implementation details.

---

## 1. File and folder naming — kebab-case

- File names: `user-card.tsx`, `format-date.ts`, `orders.api.ts`, `user-card.test.ts`.
- Folder names: `features/order-history/`, `components/empty-state/`.
- The **exported symbol** keeps its natural casing — PascalCase for components, camelCase for functions. The _file_ is kebab-case regardless:
  - `user-card.tsx` exports `UserCard`
  - `format-date.ts` exports `formatDate`
  - `orders.utils.ts` exports `formatPrice`, `calculateTotal`, etc.
- Exceptions: framework-mandated filenames keep the framework's convention. For Expo Router: `_layout.tsx`, `+not-found.tsx`, `[id].tsx`, `(tabs)/`, etc.
- Applies repo-wide, including documentation.

## 2. Alphabetical ordering

Sort alphabetically wherever order is not semantic:

- Object literal keys (props passed to a component, config objects, style objects when not order-dependent).
- Type / interface keys.
- Function parameter object destructuring (`function Foo({ a, b, c }: Props)`).
- Imports _within_ a group. Group order: React → React Native → external → `@/` aliases → relative; **within each group, alphabetical**.

Skip ordering when order carries meaning (animation step arrays, route precedence, `StyleSheet` ordering when intentional).

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
- Default exports are reserved for cases mandated by the framework: Expo Router files (`app/index.tsx`, `app/_layout.tsx`, `app/[id].tsx`) require a default export.

## 4. All feature functions live in utils

**No function definitions inside components — including event handlers.** Every named function for a feature lives in `<name>.utils.ts` (or `shared/utils/<name>.ts` per the shared/feature rule below). Components compose; utils do work.

This means:

- No `const handlePress = () => { ... }` inside a component.
- No `function onSubmit(values) { ... }` declared inside a component body.
- JSX props bind to a util directly or via a one-line arrow that just calls the util with closure values: `<Button onPress={() => addItemToCart({ cart, itemId: item.id, setCart })} />`. The arrow is a dispatch site, not a function definition — keep it to one expression.
- When a util needs values that live in component scope (state, refs, hooks, navigation), pass them in via the params object. Don't try to make the util a closure; that's what the inline JSX arrow is for.

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

Order inside a component file, separated by single blank lines:

```tsx
// 1. imports
import { useEffect } from "react";
import { Pressable, View } from "react-native";

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
    <Pressable onPress={() => selectOrderById({ id: order.id, onSelect })}>
      <View>{/* ... */}</View>
    </Pressable>
  );
}
```

One blank line between each numbered section. Skip a section entirely when there's nothing to put in it (don't leave a comment as a placeholder). **No `const handleX = ...` step** — see §4.

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
├── app/         # Expo Router routes
├── features/    # feature-sliced modules (per-feature tree below)
└── shared/      # cross-feature primitives
    ├── components/   # reusable UI
    ├── configs/      # global config (env, API clients, theme)
    ├── hooks/        # cross-feature hooks
    ├── icons/        # icon components
    ├── providers/    # third-party providers (React Query, theme, auth)
    ├── stores/       # global Zustand stores only — feature stores live in the feature (§16)
    └── utils/        # cross-feature pure utils (see §4)
```

Use a fixed file role set inside every feature directory:

```
features/<name>/
├── components/
├── tests/                  # unit/integration (.test.ts) + E2E Maestro flows (.yaml) — see §12
├── <name>.api.ts        # data access / fetchers
├── <name>.consts.ts     # static values, enums
├── <name>.hooks.ts      # React hooks
├── <name>.schemas.ts    # Zod schemas (forms, API boundaries)
├── <name>.store.ts      # Zustand store scoped to this feature (§16)
├── <name>.types.ts      # TS types (often derived from the database schema)
└── <name>.utils.ts      # pure functions (see §4)
```

Do not invent new file roles ad-hoc. If a new role is needed, decide it deliberately and apply it consistently across features.

## 7. Path aliases

- Configure in `tsconfig.json` `paths` and (when needed) `babel.config.js` `module-resolver`. Aliases mirror the §6 top-level skeleton:
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
    onPress: () => void;
    title: string;
  };
  ```

  ```tsx
  // features/cards/components/card.tsx
  import type { CardProps } from "@/features/cards/cards.types";

  export function Card({ onPress, title }: CardProps) { /* ... */ }
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

Design tokens (colors, spacing, radii, elevation, typography, breakpoints) live in `tailwind.config.*` and the project's design system docs — not in this file. What this section enforces is how the app *consumes* the design system: token names only (never raw values), the a11y minimums below, and the iconography rules.

### Accessibility minimums

- Contrast: WCAG AA at minimum.
- Every `Pressable` / interactive element gets `accessibilityRole`, `accessibilityLabel`, and `accessibilityState` when state-bearing (selected, disabled, busy).
- Touch target minimum: 44×44 pt iOS, 48×48 dp Android. Use `hitSlop` for visually smaller targets.
- `accessibilityHint` for non-obvious actions.
- VoiceOver / TalkBack: focus order matches visual order; avoid trapping focus inside non-modal views.
- Reduced motion: respect `AccessibilityInfo.isReduceMotionEnabled()` on animations.
- Dynamic type: respect system font scale unless the screen explicitly opts out via `allowFontScaling={false}` (rare — needs justification).

### Iconography

- Use **`lucide-react-native`** as the icon library.
- Components live in `shared/icons/` (one file per icon, kebab-case filename, PascalCase export — `users.tsx` exports `Users`).
- Single size grid and stroke width across the app.
- Color follows the current token, never a hex literal.
- Decorative icons get `accessibilityElementsHidden` (iOS) / `importantForAccessibility="no"` (Android); meaningful icons get an `accessibilityLabel`.

---

## 11. Styling — NativeWind

- **NativeWind** (Tailwind for React Native) is the styling system. No `StyleSheet.create`, no `react-native-unistyles`, no inline `style={{...}}` for anything but truly dynamic values (animated values from Reanimated, computed transforms).
- Tokens live in `tailwind.config.*` (see §10). The config is the single source of truth — components reference token names, never raw values.
- No magic numbers. Use scale steps (`p-md`, `gap-sm`) over `p-[13px]`. The `[arbitrary]` syntax is an escape hatch, not a default.
- Dark mode: NativeWind's `dark:` prefix wired to `useColorScheme()` + theme provider.
- Conditional classes: use `clsx` / `cn` helpers, not string concatenation. Pair with `tailwind-merge` to resolve conflicts when composing variants.
- Variants: use `cva` (class-variance-authority) for component variant APIs (`Button({ size, variant })`), not bespoke conditional chains.
- Animated styles must come from Reanimated's `useAnimatedStyle` — those are the one place inline `style` is correct (you can't animate className values).
- Platform branches: `Platform.select({ ios: ..., android: ... })` only when there's a real platform difference (haptics, blur view, status bar). Don't branch by default.
- Safe areas: every screen's root uses `SafeAreaView` (or `useSafeAreaInsets()` from `react-native-safe-area-context`) — never hardcoded top/bottom padding.
- Sort classes with the official Prettier plugin (`prettier-plugin-tailwindcss`) so review diffs stay clean.
- Targets **Tailwind v3 + NativeWind v4**. Tailwind v4's CSS-first config is not yet supported by NativeWind — stay on v3 until that lands.

---

## 12. Tests — selectors and where they live

### Selectors (a11y-first)

Tests should resemble real user perception. Selector priority:

1. **`accessibilityLabel`** (preferred) — what a VoiceOver/TalkBack user hears. Forcing every interactive element to have a meaningful label improves a11y and gives tests a stable, semantic handle.
2. **`text`** — what the user reads. Use for assertions on visible content.
3. **`testID`** — escape hatch only. Use when two elements share the same label (lists with repeated rows), when the element legitimately can't carry an `accessibilityLabel`, or when copy/i18n churn would make the test brittle without one. Pattern: `<feature>-<element>-<role>` (e.g. `orders-row-validate`).

Rules:

- Every interactive element gets a meaningful `accessibilityLabel`. This is a §10 a11y requirement that doubles as the test selector.
- Don't add a `testID` "just in case". Reach for it only when the a11y/text path is genuinely ambiguous.
- On Android, `testID` is exposed via the accessibility tree — when you do use one, pair it with an `accessibilityLabel` so screen readers still get a real string.

### Where tests live

Each feature owns one `tests/` folder containing both unit/integration tests and E2E flows:

```
features/<name>/tests/
├── <name>.test.ts       # unit / integration (Jest or Vitest + RNTL)
└── <scenario>.yaml      # E2E Maestro flows
```

- **Unit / integration (`<name>.test.ts`)**: pure utils, hooks, schemas, isolated component renders. Runs in Node/JSDOM, no device.
- **E2E (`<scenario>.yaml`)**: full user flows on simulator/device with **Maestro**.

#### Cross-feature flows — ownership rule

A flow lives in the feature whose behavior the **last assertion** validates. Prerequisite steps (login, seed data, navigation through other features) are means, not ownership. A flow that logs in → opens catalog → adds to cart → reaches checkout success belongs to the **checkout** feature, even if it touches four others.

#### Example flow

```yaml
# features/orders/tests/validate-order.yaml
appId: com.example.app
---
- launchApp
- tapOn:
    accessibilityText: "Cliente Juan Pérez"
- tapOn:
    accessibilityText: "Validar pedido"
- assertVisible:
    text: "Pedido confirmado"
```

---

## 13. Visual fidelity to designs

When implementing against a design mockup:

- The implemented UI must match the design.
- Hardcoded text matches the mockup copy character-for-character.
- Layout structure matches.
- Color/spacing tokens used match the design's tokens.
- If prose acceptance criteria contradict the image, the **image wins** — flag the contradiction before guessing.
- Verify on **both iOS and Android** — RN diverges (shadows, fonts, ripple, status bar).

---

## 14. Component states — every screen must declare them

For every component/screen, the spec must cover:

- **States**: idle, loading, error, empty, success — one paragraph per state describing what the user sees.
- **Components**: which primitives are used.
- **Copy**: exact strings, in the project's voice.
- **Microinteractions**: animations (Reanimated), haptics (`expo-haptics`), focus/error feedback.
- **Empty / loading / error states**: explicit, not implied. Skeletons or spinners declared per screen.
- **A11y notes**: roles, labels, hints, focus order.

Implement every declared state — write one test assertion per state.

### Error boundaries

- Wrap the app root and each tab/stack root with `react-native-error-boundary` (or a custom one). A render error inside a screen should fall back to a recovery UI, not crash the whole app.
- Pair with Sentry / `sentry-expo` so production crashes have a stack trace.
- Mutations and network errors are handled by React Query (`onError`) — boundaries only catch render-time exceptions.

---

## 15. Mobile-specific concerns

### Navigation

- Default: **Expo Router** (file-based). Routes live in `app/`. Group routes with `(group)/`, dynamic with `[id].tsx`.
- Use typed routes (`expo-router/typed-routes`) when supported.
- Each stack/tab declares its `_layout.tsx`. Headers, tab bars, and modals configured there, not per-screen.
- Bare RN projects use `@react-navigation/native` with the same conventions adapted.

### Lists

- Use `FlashList` (Shopify) for any list with > 20 items. `FlatList` for small lists. Never `.map()` a long list inside a `ScrollView`.
- Set `keyExtractor` and stable `getItemType` (FlashList) to avoid re-renders.
- Pagination via `@tanstack/react-query` `useInfiniteQuery`.

### Animations & gestures

- Animations: **Reanimated 3** (`useSharedValue`, `useAnimatedStyle`). Avoid the legacy `Animated` API.
- Gestures: `react-native-gesture-handler`. Always wrap the app root in `GestureHandlerRootView`.
- Honor `prefers-reduced-motion` via `useReducedMotion()`.

### Native modules & Expo

- Prefer Expo SDK modules over third-party native modules when both exist (`expo-image`, `expo-av`, `expo-camera`, etc.) — they're maintained against the current Expo version.
- New native dependencies require an EAS Build (no Expo Go). Document the build profile in deployment docs.
- Permissions: declare in `app.json` / `app.config.ts`. Request at the moment of need with a clear pre-prompt explaining why (see §18 for the security side of permissions).

### Performance

- Images: `expo-image` over `Image` (better caching, faster decode). Always set `contentFit` and explicit dimensions.
- Memoize list item components (`React.memo`) and stable callbacks (`useCallback`) — list re-renders are the most common perf bug.
- Avoid running heavy work on the JS thread during gestures/animations; use Reanimated's worklets.
- Profile with the Expo Dev Tools / Flipper / React DevTools Profiler before optimizing.

### Offline & state persistence

- Persistent client state via `@tanstack/react-query` + `AsyncStorage` persister, or `mmkv` (faster) when needed.
- Network-aware UI: handle offline state explicitly — don't assume requests succeed.

---

## 16. State management

Pick the right tool per **kind** of state. The most common bug is treating server data as client state.

### Decision matrix

| Kind of state                             | Tool                              | Notes                                                                      |
| ----------------------------------------- | --------------------------------- | -------------------------------------------------------------------------- |
| Server data (anything from an API)        | **TanStack Query (React Query)**  | Caching, refetch, optimistic updates, invalidation. Don't mirror in a store. |
| Global UI state (theme, modals, drawers)  | **Zustand** in `shared/stores/`   | Tiny, no boilerplate, no provider needed. Works outside React.             |
| Feature-scoped client state               | **Zustand** in `features/<name>/<name>.store.ts` | Lives with the feature. Don't promote to `shared/` unless another feature reads it. |
| Auth / user session                       | Supabase Auth (see §17)           | Session in `expo-secure-store`; never `AsyncStorage`. Mirror user into Zustand. |
| Form state                                | `react-hook-form` + `zod`         | Never put form values in a global store.                                   |
| Navigation state                          | Expo Router / React Navigation    | Don't duplicate route params in a store.                                   |
| Local-only state                          | `useState` / `useReducer`         | Default. Don't reach for a store.                                          |
| Cross-component derived state             | `useMemo` + props, or Zustand     | Lift only when more than two siblings need it.                             |

### Why React Query (not Redux/RTK Query) by default

Server state has different semantics from client state — it's stale by default, can be refetched, can be invalidated by mutations, can be paginated/infinite. React Query encodes all of that. Putting `users`, `posts`, etc. in a Redux/Zustand store means re-implementing caching, deduping, and refetch logic by hand. On mobile this matters more: app focus, network reconnection, and offline are all built into React Query.

```ts
// features/orders/orders.api.ts
import { supabase } from "@/shared/configs/supabase";

export function useOrders() {
  return useQuery({
    queryKey: ["orders"],
    queryFn: async () => {
      const { data, error } = await supabase.from("orders").select();
      if (error) throw error;
      return data;
    },
  });
}
```

Pair with `@tanstack/query-async-storage-persister` (or `mmkv` persister) for offline cache survival across app restarts.

### Wire React Query to React Native lifecycles

By default React Query refetches on browser focus and network reconnect — neither fires on RN. Wire them once in the root provider:

```ts
// shared/providers/query.provider.tsx
import { focusManager, onlineManager } from "@tanstack/react-query";
import NetInfo from "@react-native-community/netinfo";
import { AppState } from "react-native";

onlineManager.setEventListener((setOnline) =>
  NetInfo.addEventListener((state) => setOnline(!!state.isConnected))
);

AppState.addEventListener("change", (status) => {
  focusManager.setFocused(status === "active");
});
```

Without this, queries go stale when the app backgrounds and never recover.

### Why Zustand over Redux Toolkit by default

Same store pattern, ~10x less boilerplate, no provider, works outside React (selectors callable from utils, useful for non-component code like notification handlers, deep links). Use Redux Toolkit when:

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

### Persistence (mobile-specific)

- **Tokens / secrets**: `expo-secure-store` — never `AsyncStorage`, never Zustand persist for these.
- **UI preferences (theme, onboarding flag)**: Zustand `persist` middleware backed by `mmkv` (faster) or `AsyncStorage`.
- **Server cache**: React Query persister (`AsyncStorage` or `mmkv`). Set a `maxAge` so stale data eventually evicts.

### Rules

- **Never mix server and client state in the same store.** If you need to override a server value optimistically, use React Query's `setQueryData` or `onMutate`, not a separate store copy.
- **One store per domain.** Feature stores in the feature; cross-feature stores in `shared/stores/`. Never one mega-store.
- **Selectors with `useStore(s => s.x)`** to avoid re-renders. Don't destructure the whole store in a component — list re-renders are the most common mobile perf bug.
- **No business logic in components** — derived values go in selectors or utils.
- **Hydrate before render** when the persisted state must be correct on first paint (auth, theme). Show a splash/skeleton until the store has rehydrated.

---

## 17. Backend — Supabase

Supabase is the backend for **auth, database, and storage**. All three flow through `@supabase/supabase-js`. Don't introduce a second auth provider, ORM, or blob store alongside it.

### Client

- Initialize once in `shared/configs/supabase.ts` and import from there. Never construct ad-hoc clients in feature code.
- Configure the auth storage adapter with **`expo-secure-store`** for the session — never `AsyncStorage` (sessions contain refresh tokens).
- Set `detectSessionInUrl: false` and `flowType: 'pkce'`. Provide a custom `storage` adapter wrapping `expo-secure-store`.

### Auth

- Restore the session on app start before rendering authenticated routes. Show a splash until `getSession()` resolves; don't flash the signed-out UI.
- Subscribe to `onAuthStateChange` once in the root provider; mirror the user into the global auth store (`shared/stores/auth.store.ts`) so screens read from one place.
- OAuth / magic links: configure deep links in `app.json` (`scheme`) and a Supabase redirect URL pointing at the scheme. Handle the callback with `expo-linking` → `supabase.auth.exchangeCodeForSession`.
- Sign-out: `supabase.auth.signOut()` clears the secure-store session; reset client caches (`queryClient.clear()`, Zustand reset) on the same path.

### Database

- Queries live in `<feature>.api.ts`; React Query hooks in `<feature>.hooks.ts` consume them. Never call Supabase directly from a component.
- Generate TypeScript types from the schema (`supabase gen types typescript`) into `shared/types/database.ts`. Don't hand-roll row types.
- **Row Level Security is mandatory**. Every table has RLS enabled and policies covering every access pattern. The client uses the anon key — RLS is what protects data, not application code.
- The **service role key never ships to the device**. If a flow needs it (admin tools, server-only mutations), proxy through a Supabase Edge Function the app calls.

### Storage

- Public buckets only when content is genuinely public. Default to private buckets + signed URLs.
- Generate signed URLs from an Edge Function or trusted backend, not the device. Pass the signed URL down to the app.
- Display images with `expo-image` using the signed URL; cache appropriately (`cachePolicy="memory-disk"`).
- Uploads from the device are fine when RLS storage policies authorize them; for sensitive paths, upload via Edge Function.

### Realtime

- Subscribe inside React Query hooks tied to **screen focus** (`useFocusEffect` from `@react-navigation/native`). Unsubscribe on blur to save battery and bandwidth.
- On event, call `queryClient.setQueryData` (or `invalidateQueries` for complex updates) so the cache stays the source of truth.
- A leaked channel is a memory + bandwidth + battery leak — always clean up.

### Environment variables

- `EXPO_PUBLIC_SUPABASE_URL`, `EXPO_PUBLIC_SUPABASE_ANON_KEY` — safe to ship in the bundle.
- **Service role key never lives on the device.** If you need it, it lives on the Edge Function side.
- Local dev uses the Supabase CLI (`supabase start`); point `EXPO_PUBLIC_SUPABASE_URL` at the LAN IP, not `localhost` (the simulator/device can't reach `localhost` on the host).

### Offline

- React Query persister + `mmkv` (or `AsyncStorage`) survives app restarts; pair with `setOnline(false)` when `@react-native-community/netinfo` reports offline so mutations queue rather than fail.
- Supabase auth tokens auto-refresh when online; on cold start while offline, the cached session still authorizes reads of cached data (subject to RLS once back online).

---

## 18. Security

### Secrets and data at rest

- Tokens / refresh tokens / biometric keys → `expo-secure-store` (§17). Never `AsyncStorage` / Zustand persist.
- API keys in code: only `EXPO_PUBLIC_*`. Anything else lives server-side. Service role keys never on device.
- PII / payment data: don't persist locally unless absolutely required; if you do, encrypt with `expo-secure-store`.
- Logging: never log tokens, full headers, request bodies with credentials, or full PII. Configure Sentry's `beforeSend` to scrub.

### Network

- HTTPS only. Reject plaintext connections in `app.json` (`NSAppTransportSecurity` iOS / `usesCleartextTraffic: false` Android).
- Trust the server: client-side validation is UX, not security. Server (Supabase RLS / Edge Functions) is the source of truth.
- Certificate pinning is optional — only for apps handling regulated data (banking, health). Adds ops cost.

### Auth & sensitive flows

- Biometric gate on sensitive screens via `expo-local-authentication` (Face ID / Touch ID / fingerprint).
- Re-authenticate before destructive actions (delete account, change email, large transfers).
- Sign-out clears: secure-store session + React Query cache + Zustand stores.

### Deep links

- Validate every deep link parameter at the route handler. Treat them as untrusted user input — parse with Zod.
- Never auto-execute actions from a deep link without user confirmation (e.g. "open URL in WebView" is a phishing vector).

### WebView hardening (when used)

- `originWhitelist` to your domains only.
- `javaScriptEnabled: false` unless required.
- Never inject user-controlled HTML/JS.

### Permissions

- Request minimum necessary; explain why with a pre-prompt before the OS dialog (§15).
- Revoke access paths: settings deep link to OS permission page.

### Sensitive screens

- Block screenshots / screen recording on sensitive screens via `expo-screen-capture` `preventScreenCaptureAsync`.
- Mask app contents on backgrounding (PCI / banking apps).

### Dependencies

- `pnpm audit` on CI; gate merges on no high/critical advisories.
- Dependabot or Renovate for automated PRs.
- Lockfile committed; never run with `--no-frozen-lockfile` in CI.

### Threat-model defaults

- Treat the device as compromised: jailbreak/root detection only adds friction for legitimate users in most cases — not worth it unless regulated.
- Treat any input from the network or another app as untrusted.

---

## 19. When to deviate

- If existing code violates one of these rules, follow the existing pattern locally and propose the convention change deliberately before fixing at scale. Don't drive-by refactor; drift fixes are a separate task.
- Routine compliance is the default. Deviating from a rule should be a recorded decision, not a quiet preference.

