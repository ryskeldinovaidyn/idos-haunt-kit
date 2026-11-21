# Implementation Plan: Haunt Kit

## Overview

Implement the Haunt Kit monorepo in TypeScript/Vite: the `@haunt-kit/skeleton` package (config, auth bootstrap, wallet, leaderboard, AI wrapper, design system, shared tooling), the Grave Runner endless-runner game, and the Séance AI tool. Tasks follow the dependency order: workspace scaffolding → skeleton services → apps → integration wiring.

## Tasks

- [ ] 1. Scaffold monorepo and workspace tooling
  - Create root `package.json` with `"workspaces": ["skeleton", "apps/grave-runner", "apps/seance"]` and workspace-level dev dependencies (Vite, TypeScript strict, ESLint TypeScript-aware, Prettier, Vitest, fast-check)
  - Create `eslint.config.mjs` extending a TypeScript-aware shared ruleset with zero-error policy
  - Create `.prettierrc` with shared Prettier defaults
  - Create `vitest.workspace.ts` that discovers all `*.test.ts` files across all packages
  - Create `tsconfig.base.json` at root with `"strict": true` and path alias for `@haunt-kit/skeleton`
  - Create `LICENSE` with full MIT license text
  - _Requirements: 15.1, 15.3, 15.4, 15.5, 7.4, 7.6_

- [ ] 2. Implement skeleton package entry and config module
  - Create `/skeleton/package.json` as `@haunt-kit/skeleton` with exact-pinned `@idosgames/*` versions and a `main`/`exports` field pointing to the package entry
  - Create `/skeleton/vite.config.ts` in library mode (ESM output)
  - Create `/skeleton/src/config.ts` exporting the `Config` interface, `DesignTokenOverrides` interface, and `defineConfig()` helper with validation logic (throws `Error('titleId is required')` / `Error('currencyCode is required')` for absent, null, undefined, empty, or whitespace-only values)
  - Create `/skeleton/src/index.ts` as the package entry point, re-exporting `Config`, `defineConfig`, design tokens, and effect names
  - _Requirements: 1.1, 1.2, 1.3, 1.4, 7.1, 7.3_

  - [ ]* 2.1 Write property test for defineConfig blank-field rejection
    - **Property 7: defineConfig rejects blank fields**
    - Generator: `Config` objects with `titleId` or `currencyCode` drawn from `fc.oneof(fc.constant(''), fc.constant('  '), fc.constant(null), fc.constant(undefined))`
    - Assert `Error` is thrown and no config object is returned
    - Minimum 100 runs; tag comment: `// Feature: haunt-kit, Property 7`
    - **Validates: Requirements 1.2, 1.3**

  - [ ]* 2.2 Write unit tests for defineConfig
    - Test valid config is returned unchanged
    - Test each blank-field variant (`''`, `'  '`, null, undefined, absent) for both `titleId` and `currencyCode`
    - Test that error message names the invalid field
    - _Requirements: 1.1, 1.2, 1.3_

- [ ] 3. Implement skeleton — WalletService
  - Create `/skeleton/src/services/wallet.ts`
  - Implement `InsufficientFundsError`, `InvalidAmountError`, `WalletServiceError` custom error classes
  - Implement `WalletService` with `getBalance()`, `creditSouls(amount)`, and `debitSouls(amount)`
  - Validate amount is a positive integer in [1, 999_999_999]; throw `InvalidAmountError` without touching balance otherwise
  - Guard `debitSouls` for insufficient balance; throw `InsufficientFundsError` without touching balance
  - Wrap Core currency calls; throw `WalletServiceError` and leave balance unchanged on Core failure
  - Export error classes and `WalletService` from `/skeleton/src/index.ts`
  - _Requirements: 3.1, 3.2, 3.3, 3.4, 3.5, 3.6, 3.7_

  - [ ]* 3.1 Write property test for wallet balance invariant
    - **Property 1: Wallet balance invariant**
    - Generator: array of `{ op: 'credit' | 'debit', amount: integer(1, 999_999_999) }` filtered so no debit exceeds running balance
    - Assert final balance = initial + Σcredits − Σdebits
    - Minimum 100 runs; tag: `// Feature: haunt-kit, Property 1`
    - **Validates: Requirements 3.7**

  - [ ]* 3.2 Write property test for invalid amount rejection
    - **Property 2: Invalid amount rejection**
    - Generator: values outside [1, 999_999_999] (zero, negatives, floats, NaN, Infinity, strings)
    - Assert `InvalidAmountError` thrown and balance unchanged
    - Minimum 100 runs; tag: `// Feature: haunt-kit, Property 2`
    - **Validates: Requirements 3.5**

  - [ ]* 3.3 Write property test for insufficient funds rejection
    - **Property 3: Insufficient funds rejection**
    - Generator: `(balance: nat, overage: nat(1, MAX))`, debit amount = balance + overage
    - Assert `InsufficientFundsError` thrown and balance unchanged
    - Minimum 100 runs; tag: `// Feature: haunt-kit, Property 3`
    - **Validates: Requirements 3.4**

  - [ ]* 3.4 Write unit tests for WalletService
    - Credit/debit happy paths with mocked Core client (`vi.fn()`)
    - Boundary values: amount=0, 1, 999_999_999, 1_000_000_000, -1, 1.5
    - `WalletServiceError` when Core call throws
    - _Requirements: 3.1–3.6_

- [ ] 4. Implement skeleton — LeaderboardService
  - Create `/skeleton/src/services/leaderboard.ts`
  - Implement `InvalidScoreError`, `AuthenticationError`, `ServiceUnavailableError` custom error classes
  - Implement `LeaderboardEntry` interface and `LeaderboardService` with `submitScore(score)` and `getTopScores(limit)`
  - Validate score is a non-negative integer; throw `InvalidScoreError` for negatives or non-integers without calling Core
  - Validate limit in [1, 100]; throw `RangeError` outside range
  - Throw `AuthenticationError` when no user is authenticated
  - Implement personal-best logic: submit only if score > prior score
  - Wrap Core calls; throw `ServiceUnavailableError` within 5 s on Core unavailability
  - Export error classes and `LeaderboardService` from `/skeleton/src/index.ts`
  - _Requirements: 4.1, 4.2, 4.3, 4.4, 4.5, 4.6, 4.7, 4.8, 4.9_

  - [ ]* 4.1 Write unit tests for LeaderboardService
    - `submitScore` happy path (new entry, personal-best update, ignored lower score)
    - `InvalidScoreError` for negative numbers and non-integers
    - `RangeError` for limit=0 and limit=101
    - `AuthenticationError` when unauthenticated
    - `ServiceUnavailableError` on Core unavailability
    - _Requirements: 4.1–4.9_

- [ ] 5. Implement skeleton — AIService
  - Create `/skeleton/src/services/ai.ts`
  - Implement `InvalidPromptError` and `AIGenerationError` (with `original` field) custom error classes
  - Implement `AIService` with `generateText(prompt, signal?)` and `generateImage(prompt, signal?)`
  - Validate prompt: non-empty after trim, ≤ 4000 characters; throw `InvalidPromptError` without calling `client.ai`
  - Route all calls through `client.ai` and `client.ai.waitForGeneration`; never import third-party AI APIs or include API keys
  - Wrap generation failures in `AIGenerationError` with original error attached
  - Respect `AbortSignal`; reject with `AbortError` if signal is aborted, return no partial result
  - Export error classes and `AIService` from `/skeleton/src/index.ts`
  - _Requirements: 5.1, 5.2, 5.3, 5.4, 5.5, 5.6, 5.7_

  - [ ]* 5.1 Write unit tests for AIService
    - `InvalidPromptError` for empty string, whitespace-only, and 4001-character prompt
    - `AIGenerationError` forwarding with original error attached
    - AbortSignal cancellation: assert `AbortError` rejection
    - Assert no `client.ai` call is made for invalid prompts
    - _Requirements: 5.4, 5.5, 5.6, 5.7_

- [ ] 6. Implement skeleton — Design System
  - Create `/skeleton/src/ui/tokens.css` defining all six CSS custom properties (`--hk-color-bg`, `--hk-color-surface`, `--hk-color-text-primary`, `--hk-color-text-secondary`, `--hk-color-accent`, `--hk-color-error`) with dark-theme default values that meet WCAG 2.1 AA contrast ratios (≥4.5:1 normal text, ≥3:1 large text)
  - Create `/skeleton/src/ui/effects.ts` with `attachFog(el)` and `attachCandle(el)` functions; fog: `data-haunt-fog` attribute, opacity 0.3–0.6, cycle 4–8 s; candle: `data-haunt-candle` attribute, opacity 0.85–1.0 and brightness 0.9–1.1, cycle 100–300 ms
  - Create `/skeleton/src/ui/index.ts` re-exporting all token names and effect attribute names
  - Apply `theme` override merging in `boot.ts` (see task 7) by setting matching CSS custom properties on `:root` after sign-in
  - Re-export all design tokens and effect names from `/skeleton/src/index.ts`
  - _Requirements: 6.1, 6.2, 6.3, 6.4, 6.5, 6.6_

- [ ] 7. Implement skeleton — boot module
  - Create `/skeleton/src/boot.ts` with `BootResult` interface and `boot(config: Config): Promise<BootResult>`
  - Call `AppShell.init(config.titleId)` from `@idosgames/app-shell`, display sign-in and lobby UI
  - Apply `theme` overrides from `config.theme` onto `:root` CSS custom properties after sign-in
  - Reject with `TimeoutError` if sign-in does not complete within 5 s
  - On auth failure: render error message and retry button in lobby UI without page reload; do not reject the promise until user retries or timeout is reached
  - Resolve with `{ client }` (authenticated Core client) on success
  - Export `boot` and `BootResult` from `/skeleton/src/index.ts`
  - _Requirements: 2.1, 2.2, 2.3, 2.4, 2.5_

  - [ ]* 7.1 Write unit tests for boot module
    - Smoke test: mocked `AppShell` resolving within 5 s → resolves with `client`
    - Timeout test: mocked `AppShell` hanging → rejects with `TimeoutError` after 5 s
    - Auth failure test: mocked `AppShell` rejecting → error UI rendered, retry action available
    - _Requirements: 2.1, 2.2, 2.3, 2.4_

- [ ] 8. Checkpoint — skeleton unit and property tests pass
  - Ensure all tests pass, ask the user if questions arise.

- [ ] 9. Scaffold Grave Runner app
  - Create `/apps/grave-runner/package.json` with deps `@haunt-kit/skeleton` (workspace), exact-pinned `@idosgames/react 0.7.2`, and dev dep Vite
  - Create `/apps/grave-runner/vite.config.ts` in app-mode SPA, relative base path, output to `dist/`
  - Create `/apps/grave-runner/src/main.ts` calling `boot(config)` then invoking the game start callback
  - Create stub directory structure: `src/game/entities/`, `src/game/`, `src/ui/`
  - _Requirements: 7.1, 7.2, 15.1_

- [ ] 10. Implement Grave Runner — input and game loop foundation
  - Create `/apps/grave-runner/src/game/input.ts` — `InputHandler` class:
    - Keyboard: `keydown` listeners for spacebar, ArrowUp (jump), ArrowDown (slide)
    - Touch: `touchstart` y < 50% viewport height → jump; `touchmove` ≥ 30 px downward delta → slide
    - `visibilitychange` to hidden: snapshot active-key set; on visible: restore `RunState`, clear active-key set
    - Return snapshot of current input state each frame via `poll()`
  - Create `/apps/grave-runner/src/game/loop.ts` — `GameLoop` using `requestAnimationFrame`, computing `dt`, pausing on `visibilitychange` hidden, resuming on visible
  - _Requirements: 8.3, 8.4, 8.5, 8.6, 11.1, 11.2, 11.4, 11.5, 11.6, 11.7_

- [ ] 11. Implement Grave Runner — world, player, and physics
  - Create `/apps/grave-runner/src/game/world.ts` — `World` with scroll starting at 300 px/s, multiplied by 1.1 every 30 s of elapsed time, capped at 1200 px/s
  - Create `/apps/grave-runner/src/game/entities/player.ts` — `Player` with position, velocity, full/50%-height collision box; `state: 'running' | 'jumping' | 'sliding'`; jump initial velocity 800 px/s upward; ignore jump input while airborne; slide reduces collision box height by 50% for 500 ms then restores
  - Create `/apps/grave-runner/src/game/physics.ts` — AABB collision detection: overlap computed per axis; run ends only when both horizontal and vertical overlap exceed 4 px simultaneously; soul collection on player-soul bounding-box overlap
  - Define `RunState`, `Obstacle`, `SoulCollectible`, and `Player` data interfaces in a shared `src/game/types.ts`
  - _Requirements: 8.1, 8.2, 8.3, 8.4, 8.5, 8.8_

  - [ ]* 11.1 Write unit tests for World speed ramp
    - Value at t=0 s (300 px/s), t=30 s (330 px/s), t=60 s (363 px/s), cap at 1200 px/s
    - _Requirements: 8.2_

  - [ ]* 11.2 Write unit tests for physics AABB
    - Overlap exactly 4 px both axes → no run end
    - Overlap 5 px both axes → run ends
    - Non-overlapping → no run end
    - _Requirements: 8.8_

- [ ] 12. Implement Grave Runner — obstacle and soul spawners
  - Create `/apps/grave-runner/src/game/entities/obstacle.ts` — `Obstacle` entity (kind: `'tombstone' | 'bat'`, position, dimensions)
  - Create `ObstacleSpawner` in `/apps/grave-runner/src/game/entities/obstacle.ts`: spawn tombstone and bat at pseudo-random intervals [1.0 s, 3.0 s]; enforce ≥ 20 px horizontal gap between consecutive obstacles
  - Create `/apps/grave-runner/src/game/entities/soul.ts` — `SoulCollectible` entity with `id`, position, dimensions, `collected` flag
  - Create `SoulSpawner` in `/apps/grave-runner/src/game/entities/soul.ts`: spawn at pseudo-random positions within canvas bounds at intervals [2.0 s, 5.0 s]; cap at 10 collectibles on canvas at any time
  - _Requirements: 8.7, 9.1_

  - [ ]* 12.1 Write property test for obstacle spawn gap invariant
    - **Property 6: Obstacle spawn gap invariant**
    - Generator: sequence of spawn decisions from `ObstacleSpawner` given random canvas widths and scroll speeds
    - Assert every consecutive obstacle pair has gap ≥ 20 px between right edge and next left edge
    - Minimum 100 runs; tag: `// Feature: haunt-kit, Property 6`
    - **Validates: Requirements 8.7**

  - [ ]* 12.2 Write unit tests for SoulSpawner cap
    - Assert no more than 10 `SoulCollectible` entities exist on canvas simultaneously
    - _Requirements: 9.1_

- [ ] 13. Implement Grave Runner — RunState, scoring, and end-of-run flow
  - Create `/apps/grave-runner/src/game/run.ts` — `RunState` manager:
    - Track `status`, `elapsedMs` (non-paused time only), `soulsCollected`, `scrollSpeed`
    - Score formula: `Math.floor(elapsedMs / 1000) + 10 * soulsCollected`
    - On `collected` flag set for a soul: increment `soulsCollected` counter by 1
    - On run end: call `creditSouls(soulsCollected)`; on failure display error with uncredited count
    - On run end: call `submitScore(score)`; on failure persist to `localStorage` under `grave-runner:pending-score` and display retry button
    - On run end: call `getTopScores(10)`; on failure show "leaderboard unavailable" and render cached `grave-runner:cached-leaderboard` from `localStorage` if present
  - _Requirements: 9.2, 9.3, 9.4, 9.5, 9.6, 10.1, 10.2, 10.3, 10.4, 10.6_

  - [ ]* 13.1 Write property test for score calculation correctness
    - **Property 4: Score calculation correctness**
    - Generator: `(elapsedMs: nat, soulsCollected: nat)`
    - Assert `score === Math.floor(elapsedMs / 1000) + 10 * soulsCollected`
    - Minimum 100 runs; tag: `// Feature: haunt-kit, Property 4`
    - **Validates: Requirements 10.1**

  - [ ]* 13.2 Write property test for soul collection count invariant
    - **Property 5: Soul collection count invariant**
    - Generator: array of `SoulCollectible` stubs with random subset marked `collected: true`
    - Assert `creditSouls` is called with exactly the count of `collected: true` souls
    - Minimum 100 runs; tag: `// Feature: haunt-kit, Property 5`
    - **Validates: Requirements 9.5**

  - [ ]* 13.3 Write unit tests for scoring and run state
    - Score formula edge cases: 0 s / 0 souls → 0, large elapsed time and soul counts
    - `creditSouls` failure path: error message shown, uncredited count retained
    - `submitScore` failure path: score persisted to `localStorage`, retry button displayed
    - _Requirements: 9.3, 9.6, 10.1, 10.2, 10.4_

- [ ] 14. Implement Grave Runner — renderer, HUD, lobby, and end-screen UI
  - Create `/apps/grave-runner/src/game/hud.ts` — `HUD` renderer drawing current souls count in top-right corner and speed indicator on the canvas
  - Implement `Renderer` in `src/game/loop.ts` or a dedicated `renderer.ts`: clear canvas each frame, composite background scroll layer, obstacle layer, soul collectible layer, player sprite, HUD
  - Create `/apps/grave-runner/src/ui/lobby.ts` — pre-run screen: display player's highest recorded score from `localStorage` if at least one completed run exists
  - Create `/apps/grave-runner/src/ui/end-screen.ts` — end-of-run overlay: show Soul credit result (minimum 3 s), submitted score (minimum 3 s), top-10 leaderboard entries (rank, name up to 32 chars, score), retry button for failed `submitScore`
  - Render on-screen touch-zone icons when touch-capable device is detected at game start
  - _Requirements: 8.1, 9.3, 9.4, 10.2, 10.3, 10.5, 11.3_

- [ ] 15. Checkpoint — Grave Runner tests pass, game loop runs end-to-end
  - Ensure all tests pass, ask the user if questions arise.

- [ ] 16. Scaffold Séance app
  - Create `/apps/seance/package.json` with deps `@haunt-kit/skeleton` (workspace), exact-pinned `@idosgames/react 0.7.2`, and dev dep Vite
  - Create `/apps/seance/vite.config.ts` in app-mode SPA, relative base path, output to `dist/`
  - Create `/apps/seance/src/main.ts` calling `boot(config)` then mounting the React app
  - Create stub directory structure: `src/views/`, `src/services/`, `src/hooks/`
  - _Requirements: 7.1, 7.2, 15.1_

- [ ] 17. Implement Séance — gallery service
  - Create `/apps/seance/src/services/gallery.ts`
  - Define `GalleryEntry` interface (`id`, `question`, `answer`, `portraitUrl`, `timestamp`)
  - Implement `addEntry(entry)`: prepend to `haunt-kit:gallery:entries` in `localStorage`; if length would exceed 20, remove last (oldest) element before writing; handle `localStorage` unavailable or write failure by surfacing an error without throwing
  - Implement `getEntries()`: read and parse `haunt-kit:gallery:entries`; return `[]` on parse error or missing key
  - Implement `deleteEntry(id)`: remove entry by id, write updated array back; if `localStorage` throws, surface inline error
  - _Requirements: 14.1, 14.2, 14.4, 14.5, 14.6_

  - [ ]* 17.1 Write property test for gallery FIFO eviction at capacity
    - **Property 8: Gallery FIFO eviction at capacity**
    - Generator: sequence of 21–50 `addEntry` calls
    - After each call assert stored count ≤ 20 and the most-recently-added entry is present
    - Minimum 100 runs; tag: `// Feature: haunt-kit, Property 8`
    - **Validates: Requirements 14.4**

  - [ ]* 17.2 Write unit tests for gallery service
    - `addEntry` happy path, eviction at exactly 20 entries, delete, `localStorage` failure path
    - _Requirements: 14.1, 14.4, 14.5, 14.6_

- [ ] 18. Implement Séance — useSeance hook
  - Create `/apps/seance/src/hooks/useSeance.ts`
  - Orchestrate: `wallet.debitSouls(1)` → navigate to `SeanceView` (loading) → `ai.generateText(spiritPrompt(question))` → `ai.generateImage(portraitPrompt(answer))` → on success: return result and trigger `gallery.addEntry`
  - If `debitSouls` fails: surface error, return user to `QuestionView` with question text preserved, do not proceed to generation
  - If `generateText` or `generateImage` fails or 30 s elapse: call `wallet.creditSouls(1)` to refund; if refund also fails, surface both failure messages; return retry option that restores the previous question text
  - If `generateText` succeeds: proceed to `generateImage` with a portrait prompt derived from the answer
  - Expose loading state, result, and error state to consumers
  - _Requirements: 12.1, 12.2, 12.3, 12.6_

  - [ ]* 18.1 Write unit tests for useSeance hook
    - Debit-then-generate happy path: assert `debitSouls(1)` called before `generateText`, `generateImage` called after answer returned, gallery entry persisted
    - Refund on `generateText` failure: assert `creditSouls(1)` called; question text preserved
    - Refund on 30 s timeout: assert `creditSouls(1)` called
    - `debitSouls` failure: assert no AI call made, question text preserved
    - _Requirements: 12.1, 12.2, 12.6_

- [ ] 19. Implement Séance — views and animated séance scene
  - Create `/apps/seance/src/views/QuestionView.tsx`:
    - Input limited to 500 characters; show character-count validation message on empty/whitespace submit and on exceeding 500 characters
    - Submit button calls `useSeance`
    - All interactive elements reachable via Tab, activated via Enter/Space, with visible focus indicator
  - Create `/apps/seance/src/views/SeanceView.tsx`:
    - Apply `attachFog()` from design system to scene container on mount; call cleanup on unmount
    - Apply `attachCandle()` during generation loading; call cleanup on completion
    - Portrait: CSS fade-in 1–2 s when available
    - Answer text: typewriter animation revealing 30 characters/s
    - Loading indicator displayed until text answer returns or 30 s elapse
  - Create `/apps/seance/src/views/GalleryView.tsx`:
    - Load entries from `gallery.getEntries()` (newest first)
    - Each entry card shows question excerpt, timestamp, and delete button
    - Delete button calls `gallery.deleteEntry(id)` and re-renders gallery within 100 ms
    - Display "result could not be saved" message if gallery entry persistence fails
  - Create `/apps/seance/src/views/EntryView.tsx`:
    - Read-only display of question, answer, portrait image; no editable inputs or submit controls
  - Create `/apps/seance/src/App.tsx` as root component with router wiring all four views
  - _Requirements: 12.1, 12.2, 12.3, 12.4, 12.5, 12.6, 13.1, 13.2, 13.3, 13.4, 13.5, 13.6, 14.2, 14.3, 14.5, 14.6_

- [ ] 20. Implement Séance — boot wiring and localStorage error boundary
  - Connect `boot(config)` result's `client` to `WalletService`, `AIService` instances used in `useSeance`
  - Confirm `localStorage` error from `gallery.ts` surfaces as the "result could not be saved" message in `SeanceView` (not a thrown exception)
  - _Requirements: 2.1, 14.5_

- [ ] 21. Write root README
  - Create `/README.md` with:
    - Project overview section describing the monorepo purpose and the role of each workspace package
    - Prerequisites section listing minimum Node.js and npm major versions
    - Quickstart section with separate `npm install` and `npm run dev` commands shown individually for `grave-runner` and `seance`
    - "Create a new app" section explaining how to copy `/skeleton/`, rename its directory, register it in root `package.json` `workspaces`, and edit `config.ts`
  - _Requirements: 15.2_

- [ ] 22. Final checkpoint — all tests pass, ESLint clean, build succeeds
  - Run `npx vitest --run` at workspace root; assert all tests pass within 120 s
  - Run `npx eslint .` at workspace root; assert zero errors
  - Run `npm run build` in each app directory; assert each produces `dist/index.html` and at least one hashed JS file within 60 s
  - Ensure all `@idosgames/*` versions are exact (no `^` or `~`) in every `package.json`
  - Ensure no secrets, credentials, or API keys exist in any committed file
  - Ensure all tests pass, ask the user if questions arise.

## Notes

- Tasks marked with `*` are optional and can be skipped for a faster MVP build
- Each task references specific requirements for full traceability to requirements.md
- The design document (design.md) contains all interface signatures, data models, and correctness properties — refer to it during implementation rather than re-deriving them here
- Property-based tests use fast-check with a minimum of 100 runs each; each test file carries a `// Feature: haunt-kit, Property N` tag for traceability
- Unit tests mock Core SDK calls with `vi.fn()` / `vi.spyOn()`; no live SDK connection is required for any unit or property test
- Canvas rendering and 60 fps performance are verified via manual smoke testing on desktop Chrome and a mobile browser; no automated canvas tests are included
- WCAG 2.1 AA contrast ratios are verified with an accessibility tool (e.g., axe-core) run against the design system demo page

## Task Dependency Graph

```json
{
  "waves": [
    { "id": 0, "tasks": ["1"] },
    { "id": 1, "tasks": ["2", "9", "16"] },
    { "id": 2, "tasks": ["2.1", "2.2", "3", "4", "5", "6"] },
    { "id": 3, "tasks": ["3.1", "3.2", "3.3", "3.4", "4.1", "5.1", "7"] },
    { "id": 4, "tasks": ["7.1", "10", "17"] },
    { "id": 5, "tasks": ["11", "17.1", "17.2", "18"] },
    { "id": 6, "tasks": ["11.1", "11.2", "12", "18.1"] },
    { "id": 7, "tasks": ["12.1", "12.2", "13", "19"] },
    { "id": 8, "tasks": ["13.1", "13.2", "13.3", "14", "20"] },
    { "id": 9, "tasks": ["21"] },
    { "id": 10, "tasks": ["22"] }
  ]
}
```
