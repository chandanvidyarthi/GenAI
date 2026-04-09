# Phase 1 Deliverable — Prompt Strategy (OTT Watchlist Feature)

**Feature:** Logged-in users add/remove titles on a Watchlist, browse a dedicated screen, and see an on-card indicator when a title is already saved.

---

## 1. Feature breakdown — six AI-sized tasks (build order)

1. **Types and API layer** — `WatchlistItem` shapes and `watchlistClient.get/add/remove` in `src/api/watchlistClient.ts`, matching the backend contract.

2. **`useWatchlist` hook** — Fetch on mount when authenticated; `isInWatchlist(titleId)`, `addToWatchlist`, `removeFromWatchlist`; loading and error states; no UI logic inside the hook.

3. **`WatchlistIndicator`** — Presentational bookmark icon (saved vs unsaved) with `accessibilityLabel` and `testID`; receives `isSaved` and `onToggle` as props.

4. **Card integration** — Wire indicator + toggle in `src/components/ContentCard.tsx` via hook or thin wrapper; keep list rows performant; avoid hook calls inside `renderItem`.

5. **`WatchlistScreen` + navigation** — `FlatList` with items, empty state, pull-to-refresh; register on authenticated stack only (`src/navigation/AuthenticatedStack.tsx`); navigation to title detail on tap.

6. **Auth guard and polish** — Hide watchlist entry points when logged out; handle 401 errors; optional optimistic mutations with rollback on failure.

---

## 2. Two full prompts (CDIR)

### Prompt A — Task 2: `useWatchlist` Hook

**Context**
React Native OTT app. Authentication token from `useAuth()` (`src/hooks/useAuth.ts`). Backend endpoints:
- `GET /me/watchlist` → List saved titles
- `POST /me/watchlist` → Add title (body: `{ titleId }`)
- `DELETE /me/watchlist/:titleId` → Remove title

API client: `src/api/client.ts` (axios-based, handles auth headers). TypeScript strict mode enabled.

**Decomposition**
- Load watchlist array when user is authenticated
- Provide O(1) membership check (`isInWatchlist(titleId)`)
- Expose mutations: `addToWatchlist(titleId)`, `removeFromWatchlist(titleId)`
- Return state: `isLoading`, `error`, `refetch`
- No-op silently when logged out (no error)

**Instructions**
- Create `src/hooks/useWatchlist.ts` with the hook definition
- Use `useCallback` and `useMemo` to prevent unnecessary re-renders
- Errors as `Error | null`, never thrown from handler functions
- Mirror patterns from nearest existing user-data hook (e.g., `useProfile`)
- Export from `src/hooks/index.ts` if project uses barrel exports
- JSDoc comments on all public functions (parameters, return types)

**Reference**
@-mention `src/api/client.ts`, `src/hooks/useAuth.ts`

**Result**
Hook file: `src/hooks/useWatchlist.ts` (80–150 lines)

---

### Prompt B — Task 5: `WatchlistScreen` + Navigation

**Context**
React Navigation 6+; authenticated stack in `src/navigation/AuthenticatedStack.tsx`. Theme colors in `src/theme/colors.ts`. Existing list screens: `src/screens/ProfileScreen.tsx`, `src/screens/SearchResultsScreen.tsx`.

**Decomposition**
- Display watchlist titles in `FlatList` with poster image and title
- Tap item → navigate to `TitleDetail` screen, passing `{ titleId }`
- Empty state with message and CTA button ("Explore catalogue")
- Row action (swipe or button) to remove item from watchlist
- Pull-to-refresh to reload list

**Instructions**
- Implement `src/screens/WatchlistScreen.tsx`
- Use `useWatchlist()` hook from Task 2
- Configure `FlatList`: proper `keyExtractor`, `ListEmptyComponent`, `RefreshControl`
- Register route in `AuthenticatedStack.tsx`: `<Stack.Screen name="Watchlist" component={WatchlistScreen} />`
- Header style and safe area: match tab bar and profile screen conventions
- Respect project `.cursor/rules` (e.g., no hardcoded English, use theme colors)

**Reference**
@-mention `useWatchlist`, `AuthenticatedStack.tsx`, `ProfileScreen.tsx`

**Result**
Screen file: `src/screens/WatchlistScreen.tsx` + navigation registration (120–180 lines total)

---

## 3. Plan Mode outline — Task 2 (`useWatchlist`)

**Questions**
- Is watchlist paginated, or single response with all items?
- Optimistic updates expected (show/hide immediately), or wait for server?
- What is `titleId` type: `string`, `number`, or UUID?
- Are there rate limits on add/remove requests?
- What error codes should trigger a user-facing message vs. silent retry?

**Files to review**
- `src/api/client.ts` — HTTP client setup, error handling
- `src/hooks/useAuth.ts` — How to get userId, logout signal
- Backend OpenAPI spec or contract docs
- Existing hook: `src/hooks/useProfile.ts` or similar (for pattern reference)

**Steps**
1. Lock request/response shapes and error codes with backend team
2. Define hook state shape (`Array<WatchlistItem>` vs. `Map<titleId, timestamp>`)
3. Document hook API in JSDoc: parameter types, return object, error scenarios
4. Implement fetch on mount (when `userId` is available)
5. Implement mutations (`add`, `remove`) with refetch or optimistic UI
6. Handle edge cases: logout during request, double-tap add, 401 response
7. Manual test checklist: authenticated, logged out, offline, network error

---

## 4. `.cursor/rules` additions

| Rule | Reasoning |
|------|-----------|
| **Mutations only through `useWatchlist` (no scattered fetch calls)** | Prevents inconsistent state across screens; all changes flow through one hook. |
| **`WatchlistIndicator` is presentational; parent passes `isSaved` + `onToggle`** | Keeps `FlatList` rows cheap; avoids hook misuse (Rules of Hooks) inside `renderItem`. |
| **Watchlist copy in `src/i18n/watchlist.ts` or feature namespace** | Reduces hardcoded English strings; centralizes labels and error messages. |
| **Auth guard: check `userId` before rendering watchlist entry points** | Prevents logged-out users from seeing save buttons or watchlist tab. |

---

## 5. AI failure anticipation

| Failure | Catch | Follow-up prompt |
|---------|-------|------------------|
| **`useWatchlist` called inside `renderItem`** → N hook calls, Rules of Hooks violation, or stale closure bugs. | Grep `useWatchlist` in list/card files; scroll with React DevTools Profiler open. | "Move hook call to screen level; pass `isSaved` boolean to card. Show WatchlistScreen + ContentCard diff only." |
| **Wrong REST paths** (e.g., `/watchlist/add`, `/save-title`) | Diff against backend OpenAPI spec or curl test. | "Use ONLY these three paths: [paste spec]. Update `watchlistClient` methods and hook; do not invent endpoints." |
| **Missing error handling for 401 (expired token)** | Watchlist mutations fail silently; network tab shows auth error. | "On 401, clear auth state (logout). Show error handler in mutation `catch` block only." |
| **No empty state** → Blank screen when watchlist is empty. | Delete all items from watchlist on device. | "Add `ListEmptyComponent` with icon + message + button to browse catalogue." |
| **Optimistic update not rolled back on failure** → Item removed from UI but still on server. | Add item, toggle airplane mode, tap remove. | "On mutation error, revert UI state immediately. Show only the `catch` block with rollback logic." |

---

## 6. One thing that changed how I work

Asking the agent for **hook API + edge cases first, no UI** turns the first pass into a cheap design review. One big "build the whole watchlist" prompt invites wrong endpoints and tangled list performance; short, ordered prompts (Task 2 before Task 5) reduce rework more than longer system prompts.

---
