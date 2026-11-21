# Design Document

## Overview

Haunt Kit is a monorepo containing three deliverables that share a single skeleton package:

- **`@haunt-kit/skeleton`** — a reusable TypeScript/Vite starter that wraps the iDos Games SDK. It provides configuration validation, authentication bootstrap, Souls wallet, leaderboard service, AI service wrapper, a spooky design system, and shared Vite/ESLint/Vitest tooling.
- **Grave Runner** (`/apps/grave-runner/`) — an HTML canvas endless-runner game that consumes the skeleton for auth, wallet (Soul crediting), and leaderboard.
- **Séance** (`/apps/seance/`) — a DOM/CSS AI tool that consumes the skeleton for auth, wallet (Soul debiting/refunding), and AI generation.

Both apps are built with Vite into independent static bundles (one `index.html` + hashed assets each) that are uploaded to idosgames.com. No backend is involved at runtime; all SDK calls go through `@idosgames/core 0.26.3` and `@idosgames/app-shell 0.16.7`.

---

## Architecture

### Monorepo layout

```
/                            workspace root
  package.json               npm workspaces: [skeleton, apps/grave-runner, apps/seance]
  eslint.config.mjs          shared ESLint config (TypeScript-aware)
  .prettierrc                shared Prettier config
  vitest.workspace.ts        discovers *.test.ts across all packages
  README.md                  quickstart
  LICENSE                    MIT

/skeleton/                   @haunt-kit/skeleton
  package.json
  vite.config.ts             library-mode build (ESM)
  src/
    config.ts                Config interface + defineConfig()
    boot.ts                  boot() — app-shell bootstrap
    services/
      wallet.ts              WalletService
      leaderboard.ts         LeaderboardService
      ai.ts                  AIService
    ui/
      tokens.css             CSS custom properties (design tokens)
      effects.ts             fog + candle-flicker helpers
      index.ts               re-exports all tokens & effect names
    index.ts                 package entry point

/apps/grave-runner/
  package.json               deps: @haunt-kit/skeleton, @idosgames/react
  vite.config.ts             app-mode build (SPA)
  src/
    main.ts                  calls boot(), then starts game
    game/
      loop.ts                GameLoop (requestAnimationFrame)
      world.ts               World — scroll, speed ramp
      entities/
        player.ts            Player (position, velocity, collision box)
        obstacle.ts          Obstacle (tombstone / bat)
        soul.ts              SoulCollectible
      physics.ts             collision detection
      input.ts               InputHandler (keyboard + touch)
      hud.ts                 HUD renderer
      run.ts                 RunState, scoring, end-of-run flow
    ui/
      lobby.ts               pre-run screen (high score)
      end-screen.ts          end-of-run overlay

/apps/seance/
  package.json               deps: @haunt-kit/skeleton, @idosgames/react
  vite.config.ts             app-mode build (SPA)
  src/
    main.ts                  calls boot(), mounts app
    App.tsx                  root component, router
    views/
      QuestionView.tsx        question input
      SeanceView.tsx          animated scene (fog, candle, typewriter, portrait)
      GalleryView.tsx         past séance list
      EntryView.tsx           single gallery entry detail
    services/
      gallery.ts             localStorage CRUD + 20-entry eviction
    hooks/
      useSeance.ts           orchestrates debit → generateText → generateImage flow
```

### Dependency graph

```
apps/grave-runner  ──┐
                     ├──▶  @haunt-kit/skeleton  ──▶  @idosgames/core
apps/seance        ──┘                          ──▶  @idosgames/app-shell
```

No circular dependencies. Apps import the skeleton; the skeleton never imports apps.

### Boot sequence

```
app main.ts
  └─ boot(config)                        [skeleton/src/boot.ts]
       ├─ AppShell.init(config.titleId)  [@idosgames/app-shell]
       ├─ display sign-in / lobby UI
       ├─ await sign-in (timeout: 5 s)
       ├─ if timeout  → reject promise with TimeoutError
       ├─ if error    → show error + retry button (no reload)
       └─ resolve({ client })            [authenticated Core client]
```

---

## Components and Interfaces

### Skeleton — `config.ts`

```typescript
export interface Config {
  titleId: string;
  currencyCode: string;
  theme?: Partial<DesignTokenOverrides>;
}

export function defineConfig(config: Config): Config
// Throws Error('titleId is required') or Error('currencyCode is required')
// if the respective field is absent, null, undefined, empty, or whitespace-only.
```

### Skeleton — `boot.ts`

```typescript
export interface BootResult {
  client: import('@idosgames/core').CoreClient;
}

export async function boot(config: Config): Promise<BootResult>
// Rejects with TimeoutError after 5 s if sign-in does not complete.
// Renders error UI + retry on auth failure without page reload.
```

### Skeleton — `services/wallet.ts`

```typescript
export class InsufficientFundsError extends Error {}
export class InvalidAmountError extends Error {}
export class WalletServiceError extends Error {}

export class WalletService {
  getBalance(): Promise<number>
  creditSouls(amount: number): Promise<number>   // returns updated balance
  debitSouls(amount: number): Promise<number>    // returns updated balance
}
```

Valid amount range: integer in [1, 999_999_999].

### Skeleton — `services/leaderboard.ts`

```typescript
export class InvalidScoreError extends Error {}
export class AuthenticationError extends Error {}
export class ServiceUnavailableError extends Error {}

export interface LeaderboardEntry {
  rank: number;
  username: string;
  score: number;
}

export class LeaderboardService {
  submitScore(score: number): Promise<void>
  getTopScores(limit: number): Promise<LeaderboardEntry[]>
}
```

`submitScore` accepts non-negative integers. `getTopScores` accepts limit in [1, 100].

### Skeleton — `services/ai.ts`

```typescript
export class InvalidPromptError extends Error {}
export class AIGenerationError extends Error { original: unknown }

export class AIService {
  generateText(prompt: string, signal?: AbortSignal): Promise<string>
  generateImage(prompt: string, signal?: AbortSignal): Promise<string>
}
```

Both methods validate: non-empty after trim, ≤ 4000 characters. All calls go through `client.ai` + `client.ai.waitForGeneration`; no third-party API keys exist in source.

### Skeleton — `ui/tokens.css` and `ui/effects.ts`

**CSS custom properties (design tokens):**

| Token | Role |
|---|---|
| `--hk-color-bg` | Page background |
| `--hk-color-surface` | Card/panel surface |
| `--hk-color-text-primary` | Primary body text |
| `--hk-color-text-secondary` | Subdued / caption text |
| `--hk-color-accent` | Interactive accent (buttons, links) |
| `--hk-color-error` | Error state |

All values are dark-theme by default. `theme` overrides from `Config` are merged at boot time by setting the matching CSS custom property on `:root`.

**Effects:**

```typescript
export function attachFog(el: HTMLElement): () => void
// Sets data-haunt-fog attribute; returns cleanup function.
// CSS animation: opacity oscillates between 0.3–0.6, cycle 4–8 s.

export function attachCandle(el: HTMLElement): () => void
// Sets data-haunt-candle attribute; returns cleanup function.
// CSS animation: opacity 0.85–1.0, brightness 0.9–1.1, cycle 100–300 ms.
```

Exported token names and effect attribute names are re-exported from the package entry point.

### Grave Runner — game architecture

```
GameLoop (rAF)
  ├─ InputHandler.poll()           keyboard + touch state snapshot
  ├─ World.update(dt)              scroll position, speed ramp
  ├─ Player.update(dt, input)      velocity integration, jump/slide state machine
  ├─ ObstacleSpawner.update(dt)    pseudo-random interval spawn
  ├─ SoulSpawner.update(dt)        pseudo-random interval spawn (max 10 on canvas)
  ├─ Physics.checkCollisions(...)  AABB overlap test
  │     ├─ player × obstacle  →  if overlap > 4 px both axes → endRun()
  │     └─ player × soul      →  collect soul
  ├─ Renderer.draw(state)          clear + composite canvas layers
  │     ├─ background scroll layer
  │     ├─ obstacle layer
  │     ├─ soul collectible layer
  │     ├─ player sprite
  │     └─ HUD (score, souls, speed)
  └─ RunState.update(dt)           elapsed time, score accumulation
```

**Input handling:**

- Keyboard: `keydown` event listener; spacebar / ArrowUp → jump; ArrowDown → slide.
- Touch: `touchstart` → y < 50% viewport height → jump; `touchmove` detecting ≥ 30 px downward delta → slide.
- On tab blur (`visibilitychange`): pause loop, record paused state; on focus restore: resume from saved state, clear active inputs.

**Collision detection:** Axis-aligned bounding box (AABB). Overlap is computed per axis; run ends only when both horizontal and vertical overlap exceed 4 px simultaneously.

**Scoring formula:**

```
score = floor(elapsedSeconds) + 10 × soulsCollected
```

**Speed ramp:** starts at 300 px/s, multiplied by 1.1 every 30 s of elapsed time, capped at 1 200 px/s.

### Séance — UI flow

```
QuestionView
  ├─ validate (non-empty, ≤ 500 chars)
  └─ onSubmit → useSeance hook
        ├─ wallet.debitSouls(1)
        ├─ navigate to SeanceView (loading state)
        ├─ ai.generateText(spiritPrompt(question))
        ├─ ai.generateImage(portraitPrompt(answer))
        ├─ on success → show answer + portrait, persist to gallery
        └─ on error / timeout (30 s) → wallet.creditSouls(1) refund
                                      → show error + retry (restore question)

SeanceView
  ├─ fog effect active from generation start until navigation away
  ├─ candle effect active during generation, removed on completion
  ├─ portrait: CSS fade-in 1–2 s when available
  └─ answer: typewriter at 30 chars/s

GalleryView
  ├─ load from gallery.ts (sorted newest-first)
  ├─ max 20 entries (evict oldest on overflow)
  └─ per-entry: delete button → remove + re-render within 100 ms

EntryView
  └─ read-only display: question, answer, portrait
```

---

## Data Models

### Config

```typescript
interface Config {
  titleId: string;          // non-empty, non-whitespace
  currencyCode: string;     // non-empty, non-whitespace
  theme?: Partial<DesignTokenOverrides>;
}

interface DesignTokenOverrides {
  '--hk-color-bg': string;
  '--hk-color-surface': string;
  '--hk-color-text-primary': string;
  '--hk-color-text-secondary': string;
  '--hk-color-accent': string;
  '--hk-color-error': string;
}
```

### RunState (Grave Runner, in-memory)

```typescript
interface RunState {
  status: 'idle' | 'running' | 'paused' | 'ended';
  elapsedMs: number;          // milliseconds of active (non-paused) play
  soulsCollected: number;     // incremented on each soul pickup
  score: number;              // floor(elapsedMs / 1000) + 10 * soulsCollected
  scrollSpeed: number;        // px/s, 300–1200
  obstacles: Obstacle[];
  souls: SoulCollectible[];
}

interface Obstacle {
  kind: 'tombstone' | 'bat';
  x: number;                  // canvas x position
  y: number;
  width: number;
  height: number;
}

interface SoulCollectible {
  id: string;                 // unique per run
  x: number;
  y: number;
  width: number;
  height: number;
  collected: boolean;
}

interface Player {
  x: number;
  y: number;
  width: number;
  height: number;             // full or 50% (sliding)
  velocityY: number;
  state: 'running' | 'jumping' | 'sliding';
  slideExpiresAt: number;     // performance.now() ms
}
```

### GalleryEntry (Séance, localStorage)

```typescript
interface GalleryEntry {
  id: string;                 // crypto.randomUUID()
  question: string;
  answer: string;
  portraitUrl: string;        // absolute HTTPS URL
  timestamp: string;          // ISO 8601 (new Date().toISOString())
}
```

**localStorage key scheme:**

```
haunt-kit:gallery:entries    →  JSON array of GalleryEntry[], newest-first
```

Maximum 20 entries stored. On write: if length would exceed 20, remove the last item (oldest) before pushing the new entry.

---

## Correctness Properties

*A property is a characteristic or behavior that should hold true across all valid executions of a system — essentially, a formal statement about what the system should do. Properties serve as the bridge between human-readable specifications and machine-verifiable correctness guarantees.*

### Property 1: Wallet balance invariant

*For any* sequence of `creditSouls` and `debitSouls` calls where each call uses a valid amount and each debit is preceded by a sufficient balance, the final balance SHALL equal the initial balance plus the sum of all credited amounts minus the sum of all successfully debited amounts.

**Validates: Requirements 3.7**

### Property 2: Invalid amount rejection

*For any* value passed to `creditSouls` or `debitSouls` that is not a positive integer in [1, 999_999_999] (including zero, negative numbers, non-integers, strings, NaN, and Infinity), the wallet SHALL throw `InvalidAmountError` and the balance SHALL remain unchanged.

**Validates: Requirements 3.5**

### Property 3: Insufficient funds rejection

*For any* wallet with balance `b` and any debit amount `a` where `a > b`, calling `debitSouls(a)` SHALL throw `InsufficientFundsError` and the balance SHALL remain `b`.

**Validates: Requirements 3.4**

### Property 4: Score calculation correctness

*For any* non-negative integer elapsed-seconds value `t` and any non-negative integer souls-collected value `s`, the computed run score SHALL equal `t + 10 × s` (where elapsed seconds is the floor of elapsed milliseconds divided by 1000).

**Validates: Requirements 10.1**

### Property 5: Soul collection count invariant

*For any* completed run, the integer passed to `creditSouls` SHALL equal the number of `SoulCollectible` entities whose `collected` flag was set to `true` during that run.

**Validates: Requirements 9.5**

### Property 6: Obstacle spawn gap invariant

*For any* sequence of spawned obstacles generated by the obstacle spawner, the horizontal distance between the right edge of any obstacle and the left edge of the next obstacle SHALL be at least 20 px.

**Validates: Requirements 8.7**

### Property 7: defineConfig rejects blank fields

*For any* `Config` object where `titleId` or `currencyCode` is absent, null, undefined, an empty string, or a string composed entirely of whitespace, `defineConfig` SHALL throw an `Error` and SHALL NOT return a config object.

**Validates: Requirements 1.2, 1.3**

### Property 8: Gallery FIFO eviction at capacity

*For any* gallery that already contains 20 entries, persisting one new entry SHALL result in exactly 20 entries stored, with the oldest entry removed and the new entry present.

**Validates: Requirements 14.4**

---

## Error Handling

### Skeleton services — general pattern

Each service method is `async` and throws a typed error subclass on failure. Callers catch specific error types. No error is silently swallowed.

| Scenario | Error type | Side-effect |
|---|---|---|
| `defineConfig` — blank field | `Error` | None |
| `boot` — sign-in timeout | `TimeoutError` | Promise rejected |
| `boot` — auth failure | (displayed in UI) | Retry presented, no reload |
| `wallet` — invalid amount | `InvalidAmountError` | Balance unchanged |
| `wallet` — insufficient funds | `InsufficientFundsError` | Balance unchanged |
| `wallet` — Core call fails | `WalletServiceError` | Balance unchanged |
| `leaderboard` — invalid score | `InvalidScoreError` | No Core call |
| `leaderboard` — limit out of range | `RangeError` | No Core call |
| `leaderboard` — unauthenticated | `AuthenticationError` | No Core call |
| `leaderboard` — Core unavailable | `ServiceUnavailableError` | Rejects within 5 s |
| `ai` — blank/long prompt | `InvalidPromptError` | No `client.ai` call |
| `ai` — generation failed | `AIGenerationError` | Original error attached |
| `ai` — AbortSignal fired | `AbortError` | No partial result returned |

### Grave Runner — runtime error handling

- **`creditSouls` failure on run end:** display error message with uncredited soul count. Retain count for the session so the player is aware; do not block the end-of-run screen.
- **`submitScore` failure:** persist score to `localStorage` under `grave-runner:pending-score`. Display a retry button; on user activation, re-attempt `submitScore` with the stored score.
- **`getTopScores` failure:** display "leaderboard unavailable" and render the most recent cached response from `localStorage` (`grave-runner:cached-leaderboard`) if present.
- **Tab focus loss:** `visibilitychange` to `hidden` pauses the game loop and snapshots `RunState`. On `visible`, restore state and clear pressed-key set to avoid phantom inputs.

### Séance — runtime error handling

- **`debitSouls` failure before generation:** display an error message, do not proceed to generation, return user to `QuestionView` with question text preserved.
- **`generateText` or `generateImage` failure / 30 s timeout:** call `creditSouls(1)` to refund. If refund also fails, display both failure messages. Present retry option that restores the previous question text.
- **`localStorage` unavailable or write failure:** display "result could not be saved" message, complete the séance display without persisting.
- **Gallery delete failure:** re-render the gallery immediately after `localStorage.removeItem`; if `localStorage` throws, show an inline error on the entry.

---

## Testing Strategy

### Unit tests (Vitest)

Unit tests cover all pure logic in the skeleton services and app-specific logic that does not require a live SDK connection. The Core client is mocked with `vi.fn()` / `vi.spyOn()`.

**Skeleton**

| Module | What to test |
|---|---|
| `config.ts` | `defineConfig` accepts valid configs; throws on each blank-field variant |
| `services/wallet.ts` | Credit/debit happy paths; `InvalidAmountError` boundary values (0, 1, 999_999_999, 1_000_000_000, -1, 1.5); `InsufficientFundsError`; `WalletServiceError` on Core throw |
| `services/leaderboard.ts` | `submitScore` happy path; `InvalidScoreError` for negatives/non-integers; `RangeError` for limit=0/101; `AuthenticationError`; personal-best update-only logic |
| `services/ai.ts` | `InvalidPromptError` for empty, whitespace, 4001-char prompts; `AIGenerationError` forwarding; AbortSignal cancellation |

**Grave Runner**

| Module | What to test |
|---|---|
| `game/run.ts` | Score formula edge cases (0 s / 0 souls, large values) |
| `game/physics.ts` | AABB overlap: exactly 4 px (no end), 5 px (end); non-overlapping |
| `game/world.ts` | Speed ramp: value at t=0, t=30 s, t=60 s, capped at 1200 |
| `ObstacleSpawner` | Spawn interval range; minimum gap enforcement |
| `SoulSpawner` | Max 10 collectibles on canvas at any time |

**Séance**

| Module | What to test |
|---|---|
| `services/gallery.ts` | Add entry; eviction at exactly 20; delete; `localStorage` failure path |
| `useSeance` | Debit-then-generate sequence; refund on AI failure; refund on timeout |

### Property-based tests (fast-check, minimum 100 runs each)

Each test is tagged with its design property for traceability.

```
// Feature: haunt-kit, Property 1: Wallet balance invariant
// Feature: haunt-kit, Property 2: Invalid amount rejection
// Feature: haunt-kit, Property 3: Insufficient funds rejection
// Feature: haunt-kit, Property 4: Score calculation correctness
// Feature: haunt-kit, Property 5: Soul collection count invariant
// Feature: haunt-kit, Property 6: Obstacle spawn gap invariant
// Feature: haunt-kit, Property 7: defineConfig rejects blank fields
// Feature: haunt-kit, Property 8: Gallery FIFO eviction at capacity
```

**Property 1 — Wallet balance invariant (`wallet.test.ts`)**  
Generator: array of `{ op: 'credit' | 'debit', amount: integer(1, 999_999_999) }` operations, filtered so no debit exceeds the running balance. Assert final balance equals initial + Σcredits − Σdebits.

**Property 2 — Invalid amount rejection (`wallet.test.ts`)**  
Generator: values outside [1, 999_999_999] (zero, negative integers, floats, NaN, Infinity, strings coerced to number). Assert `InvalidAmountError` is thrown and balance is unchanged after each call.

**Property 3 — Insufficient funds (`wallet.test.ts`)**  
Generator: `(balance: nat, overage: nat(1, MAX))`. Assert `InsufficientFundsError` when debit amount = balance + overage.

**Property 4 — Score formula (`run.test.ts`)**  
Generator: `(elapsedMs: nat, soulsCollected: nat)`. Assert `score === Math.floor(elapsedMs / 1000) + 10 * soulsCollected`.

**Property 5 — Soul collection invariant (`run.test.ts`)**  
Generator: array of `SoulCollectible` stubs, random subset marked as collected. Assert `creditSouls` is called with exactly the count of collected-flagged souls.

**Property 6 — Obstacle gap invariant (`spawner.test.ts`)**  
Generator: sequence of spawn decisions from `ObstacleSpawner` given random canvas widths and scroll speeds. Assert for every consecutive pair the gap ≥ 20 px.

**Property 7 — defineConfig blank fields (`config.test.ts`)**  
Generator: `Config` objects where one of `titleId`/`currencyCode` is drawn from `fc.oneof(fc.constant(''), fc.constant('  '), fc.constant(null), fc.constant(undefined))`. Assert `Error` is thrown.

**Property 8 — Gallery eviction (`gallery.test.ts`)**  
Generator: sequence of 21–50 add operations. After each operation assert stored count ≤ 20 and the most-recently-added entry is present.

### Integration and smoke considerations

- **Boot flow:** A smoke test using a mocked `AppShell` verifies that `boot()` resolves with a `client` object when `AppShell.init` resolves within 5 s and rejects with `TimeoutError` when it hangs.
- **Vite build:** Verified by `npm run build` in CI for both apps; output must contain `index.html` and at least one hashed JS file.
- **ESLint / Prettier:** Run at workspace root; must report zero errors across all packages.
- **Canvas rendering:** Not unit-tested. Manual and visual smoke testing on desktop Chrome and a mobile browser covers 60 fps target and touch input.
- **Contrast ratios:** Verified with an automated accessibility tool (e.g., axe-core) run against the design system demo page; all colour pairs must meet WCAG 2.1 AA minimums (4.5:1 normal text, 3:1 large text).
