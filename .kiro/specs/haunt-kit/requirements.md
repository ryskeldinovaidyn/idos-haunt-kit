# Requirements Document

## Introduction

Haunt Kit is iDos Games' Kiroween 2026 Skeleton Crew entry. It consists of three deliverables:

1. **Skeleton** (`/skeleton/`) — a reusable Vite + TypeScript starter template built on the iDos Games SDK. It provides sign-in and lobby bootstrap, a typed config file, a soft-currency wallet ("Souls"), a leaderboard service, a thin AI wrapper, a spooky design system, and a build pipeline targeting idosgames.com static bundles.
2. **Grave Runner** (`/apps/grave-runner/`) — a 60 fps HTML canvas endless-runner game where the player jumps and slides through a haunted graveyard, collects Souls, and posts scores to a global leaderboard.
3. **Séance** (`/apps/seance/`) — an AI tool where the user asks a spirit a question, receives an AI-generated answer and portrait, views them in an animated séance scene, and browses a local gallery of past séances. Each séance costs 1 Soul.

Both apps import the skeleton as an npm workspace package and never duplicate its code. The repository is MIT-licensed with a README quickstart.

## Glossary

- **Skeleton**: The `@haunt-kit/skeleton` npm workspace package located at `/skeleton/`. It is the reusable template all apps build on.
- **App_Shell**: The `@idosgames/app-shell` package (v0.16.7) responsible for sign-in and lobby UI.
- **Core**: The `@idosgames/core` package (v0.26.3) providing auth, currencies, leaderboards, and AI client.
- **AIService**: The `client.ai` interface from Core used for all AI text and image generation at runtime.
- **Souls**: The soft-currency unit used across all apps. Currency code is defined in `skeleton/src/config.ts`.
- **Wallet**: The Souls balance and transaction service implemented in `skeleton/src/services/wallet.ts`, wrapping Core's currency API.
- **Leaderboard**: The global score-ranking service implemented in `skeleton/src/services/leaderboard.ts`, wrapping Core's leaderboard API.
- **Config**: The single typed configuration object defined in `skeleton/src/config.ts` (title ID, currency code, theme overrides).
- **Grave_Runner**: The endless-runner game app at `/apps/grave-runner/`.
- **Seance**: The AI séance tool app at `/apps/seance/`.
- **Static_Bundle**: The output of a Vite production build: an `index.html` plus hashed asset files uploadable to idosgames.com.
- **Design_System**: The CSS custom properties, dark theme, spooky visual effects (fog, candle flicker), and shared UI components in `skeleton/src/ui/`.
- **Gallery**: The local (browser) store of past séance results (question, answer text, portrait image) maintained by the Seance app.

---

## Requirements

### Requirement 1: Skeleton — Configuration

**User Story:** As a developer starting a new app, I want a single typed config file, so that I can configure my app's identity and theme without touching SDK internals.

#### Acceptance Criteria

1. THE Skeleton SHALL export a `Config` TypeScript interface with required fields `titleId` (string) and `currencyCode` (string), and an optional field `theme` (partial Design System token overrides object where any subset of token keys may be provided).
2. THE Skeleton SHALL provide a `defineConfig` helper that accepts a `Config` object, validates that `titleId` and `currencyCode` are present and non-empty after trimming whitespace, and returns the validated config object unchanged.
3. IF `defineConfig` receives a value for `titleId` or `currencyCode` that is absent, null, undefined, an empty string, or a string containing only whitespace characters, THEN THE Skeleton SHALL throw an `Error` whose message identifies the invalid field by name.
4. THE Skeleton SHALL export the `Config` interface and `defineConfig` helper from its package entry point so that consuming apps can import them with a top-level package import.

---

### Requirement 2: Skeleton — Authentication and Lobby Bootstrap

**User Story:** As a developer, I want sign-in and lobby handled for me, so that my app starts authenticated without writing auth code.

#### Acceptance Criteria

1. WHEN `boot` is called with a valid `Config`, THE Skeleton SHALL initialise the App_Shell with `@idosgames/app-shell`, display the sign-in and lobby UI, and resolve with an authenticated session object.
2. WHEN the user completes sign-in, THE Skeleton SHALL provide the authenticated `client` instance from Core to the calling app.
3. IF the sign-in flow fails for any reason (including network error, invalid credentials, or server error), THEN THE Skeleton SHALL display an error message indicating the failure reason in the lobby UI and present a user-accessible retry action, without requiring a page reload.
4. IF `boot` does not resolve with an authenticated session within 5 seconds of invocation, THEN THE Skeleton SHALL reject the returned promise with an error indicating a timeout.
5. WHEN `boot` is called on a connection with a minimum download speed of 4 Mbps, THE Skeleton SHALL invoke the app's start callback within 5 seconds of the `boot` call.

---

### Requirement 3: Skeleton — Souls Wallet Service

**User Story:** As a developer, I want a ready-made soft-currency wallet, so that my app can credit and debit Souls without writing currency logic.

#### Acceptance Criteria

1. WHEN `getBalance()` is called, THE Wallet SHALL return the authenticated user's current Souls balance as a non-negative integer.
2. WHEN `creditSouls(amount)` is called with a positive integer `amount` between 1 and 999,999,999 inclusive, THE Wallet SHALL add `amount` to the user's Souls balance via Core and return the updated balance.
3. WHEN `debitSouls(amount)` is called, IF the user's balance is greater than or equal to `amount`, THEN THE Wallet SHALL subtract `amount` from the balance via Core and return the updated balance.
4. IF `debitSouls(amount)` is called and the user's balance is less than `amount`, THEN THE Wallet SHALL throw an `InsufficientFundsError` without modifying the balance.
5. IF `creditSouls` or `debitSouls` is called with a value that is not a positive integer between 1 and 999,999,999 inclusive, THEN THE Wallet SHALL throw an `InvalidAmountError` without modifying the balance.
6. IF the Core service call fails during `creditSouls` or `debitSouls`, THEN THE Wallet SHALL throw a `WalletServiceError` and leave the balance unchanged.
7. THE Wallet SHALL maintain a balance equal to the initial balance plus the sum of all successfully credited amounts minus the sum of all successfully debited amounts across all `creditSouls` and `debitSouls` calls.

---

### Requirement 4: Skeleton — Leaderboard Service

**User Story:** As a developer, I want a leaderboard service, so that my app can post and retrieve scores without writing ranking logic.

#### Acceptance Criteria

1. THE Leaderboard SHALL expose a `submitScore(score)` function that posts the authenticated user's integer score to the global leaderboard via Core.
2. THE Leaderboard SHALL expose a `getTopScores(limit)` function that returns the top `limit` entries from the global leaderboard, each containing `rank` (number), `username` (string), and `score` (number).
3. IF `submitScore` is called with a value that is not a non-negative integer, THEN THE Leaderboard SHALL throw an `InvalidScoreError` without posting to Core.
4. IF `getTopScores` is called with a `limit` less than 1 or greater than 100, THEN THE Leaderboard SHALL throw a `RangeError`.
5. WHEN `submitScore(score)` is called and the authenticated user has no prior score, THE Leaderboard SHALL create a new leaderboard entry for the user.
6. WHEN `submitScore(score)` is called and the authenticated user already has a prior score, THE Leaderboard SHALL update the entry only if `score` is greater than the prior score.
7. IF `submitScore` is called when no user is authenticated, THEN THE Leaderboard SHALL throw an `AuthenticationError` without posting to Core.
8. IF the Core service is unavailable when `submitScore` or `getTopScores` is called, THEN THE Leaderboard SHALL throw a `ServiceUnavailableError` within 5 seconds.
9. WHEN `getTopScores(limit)` is called and fewer than `limit` entries exist in the global leaderboard, THE Leaderboard SHALL return all available entries ranked from highest to lowest score.

---

### Requirement 5: Skeleton — AI Service Wrapper

**User Story:** As a developer, I want a thin AI wrapper, so that my app can generate text and images without handling raw SDK polling or shipping API keys.

#### Acceptance Criteria

1. WHEN `generateText(prompt)` is called with a valid prompt, THE AIService SHALL return the generated text as a non-empty string.
2. WHEN `generateImage(prompt)` is called with a valid prompt, THE AIService SHALL return an absolute HTTPS URL string pointing to the generated image.
3. THE AIService SHALL never call any third-party AI API directly and SHALL never include API keys in the source code or build output.
4. IF `generateText` or `generateImage` is called with a prompt that is an empty string or contains only whitespace characters, THEN THE AIService SHALL throw an `InvalidPromptError` without calling `client.ai`.
5. IF `generateText` or `generateImage` is called with a prompt exceeding 4000 characters, THEN THE AIService SHALL throw an `InvalidPromptError` without calling `client.ai`.
6. IF `client.ai` returns a response indicating a failed generation, THEN THE AIService SHALL throw an `AIGenerationError` containing the original error message from `client.ai`.
7. WHILE a generation is in progress, THE AIService SHALL accept an `AbortSignal` parameter so the caller can cancel the pending request, and IF the signal is aborted, THEN THE AIService SHALL reject the returned promise with an `AbortError` without returning a partial result.

---

### Requirement 6: Skeleton — Design System

**User Story:** As a developer, I want a spooky design system, so that my app looks Halloween-themed and accessible without writing CSS from scratch.

#### Acceptance Criteria

1. THE Design_System SHALL define CSS custom properties for all colour roles (background, surface, text-primary, text-secondary, accent, error) using dark-theme values by default.
2. THE Design_System SHALL provide a fog effect implemented in CSS/Canvas that layers an animated mist with opacity between 0.3 and 0.6 over any element with the `data-haunt-fog` attribute, with one full fog cycle completing between 4 and 8 seconds.
3. THE Design_System SHALL provide a candle-flicker effect implemented as a CSS animation that oscillates the opacity between 0.85 and 1.0 and the brightness between 0.9 and 1.1 on any element with the `data-haunt-candle` attribute, with one flicker cycle completing between 100 and 300 milliseconds.
4. THE Design_System SHALL ensure all text and interactive element colour combinations meet WCAG 2.1 AA contrast ratio (minimum 4.5:1 for normal text, and minimum 3:1 for large text defined as text at least 18pt regular weight or at least 14pt bold weight).
5. THE Design_System SHALL export all design tokens and effect class names from the skeleton package entry point, such that every token defined in criterion 1 and every effect attribute defined in criteria 2 and 3 is accessible via a named export.
6. WHERE a `theme` override is provided in the Config, THE Design_System SHALL merge the override values with the default tokens, applying the override value for each specified token and retaining the default value for any token not present in the override.

---

### Requirement 7: Skeleton — Build and Package

**User Story:** As a developer, I want each app to produce a deployable static bundle, so that I can upload it to idosgames.com without a server.

#### Acceptance Criteria

1. THE Skeleton SHALL provide a Vite configuration that builds any consuming app to a Static_Bundle containing `index.html`, hashed JavaScript files, and hashed CSS files, with all asset URLs resolved using a relative base path so the bundle loads correctly from a static file host.
2. WHEN `npm run build` is executed in an app directory, THE Skeleton SHALL produce a `dist/` directory containing the Static_Bundle within 60 seconds on a machine with at least 8 GB RAM and a quad-core CPU.
3. THE Skeleton SHALL be published as the npm workspace package `@haunt-kit/skeleton` and SHALL be importable from app code using that package specifier.
4. THE Skeleton SHALL include an ESLint configuration and a Prettier configuration at the workspace root; the ESLint configuration SHALL extend a TypeScript-aware shared ruleset and SHALL report zero errors when run against any file in the workspace.
5. THE Skeleton SHALL contain no secrets, credentials, or API keys in any committed file.
6. THE Skeleton SHALL include a Vitest configuration that discovers and runs all `*.test.ts` files within the workspace and reports a final pass/fail result within 120 seconds on a machine with at least 8 GB RAM and a quad-core CPU.

---

### Requirement 8: Grave Runner — Core Gameplay

**User Story:** As a player, I want to run through a haunted graveyard, so that I can dodge obstacles and collect Souls in an endless runner.

#### Acceptance Criteria

1. THE Grave_Runner SHALL render the game world on an HTML canvas element targeting 60 frames per second, with a minimum sustained frame rate of 30 frames per second on a device with at least a 2-core CPU at 1.8 GHz and 2 GB RAM.
2. THE Grave_Runner SHALL scroll the graveyard background and obstacle layer continuously from right to left at an initial speed of 300 px/s, increasing scroll speed by 10% every 30 seconds of elapsed play time up to a maximum of 1200 px/s.
3. WHEN the player presses the spacebar or ArrowUp key while the player character is on the ground, THE Grave_Runner SHALL make the player character jump with an initial vertical velocity of 800 px/s upward.
4. IF the player presses the spacebar or ArrowUp key while the player character is airborne, THEN THE Grave_Runner SHALL ignore the input and not trigger an additional jump.
5. WHEN the player presses the ArrowDown key or performs a downward swipe of at least 30 px on a touch screen, THE Grave_Runner SHALL reduce the player character's collision box height by 50% for 500 ms, then restore it to its standing height.
6. WHEN the player taps the upper half of the touch screen, THE Grave_Runner SHALL trigger the same jump action as the spacebar.
7. THE Grave_Runner SHALL spawn tombstone and bat obstacles at pseudo-random intervals between 1.0 s and 3.0 s, with a minimum horizontal gap of 20 px between any two consecutive obstacles.
8. IF the player character's bounding box overlaps an obstacle's bounding box by more than 4 px in both the horizontal and vertical axes simultaneously, THEN THE Grave_Runner SHALL end the current run.

---

### Requirement 9: Grave Runner — Soul Collection

**User Story:** As a player, I want to collect Souls during a run, so that my Souls wallet balance grows as I play.

#### Acceptance Criteria

1. WHILE an active run is in progress, THE Grave_Runner SHALL spawn Soul collectibles at pseudo-random positions within the canvas bounds at intervals between 2.0 s and 5.0 s, with no more than 10 Soul collectibles present on the canvas at any one time.
2. WHEN the player character's bounding box overlaps a Soul collectible's bounding box, THE Grave_Runner SHALL remove the collectible from the canvas and increment the run's collected-Souls counter by 1.
3. WHEN a run ends, THE Grave_Runner SHALL call `creditSouls` with the total Souls collected during that run and display the credited Soul count to the player for a minimum of 3 s or until the player dismisses the end-of-run screen, whichever comes first.
4. WHILE an active run is in progress, THE Grave_Runner SHALL display the current run's collected-Souls count as a HUD element in the top-right corner of the canvas.
5. THE Grave_Runner SHALL ensure that for every completed run the Soul count passed to `creditSouls` equals the number of Soul collectibles the player overlapped during that run.
6. IF the `creditSouls` call fails, THEN THE Grave_Runner SHALL display an error message indicating the Soul credit could not be applied and retain the uncredited Soul count so the player is aware of the discrepancy.

---

### Requirement 10: Grave Runner — Scoring and Leaderboard

**User Story:** As a player, I want my run score posted to the global leaderboard, so that I can compete with other players.

#### Acceptance Criteria

1. THE Grave_Runner SHALL calculate the run score as the sum of elapsed play time in whole seconds (truncated, not rounded) plus ten times the number of Souls collected during that run.
2. WHEN a run ends, THE Grave_Runner SHALL call `submitScore` with the calculated score and display the submitted score value to the player for a minimum of 3 seconds before proceeding.
3. WHEN a run ends, THE Grave_Runner SHALL call `getTopScores(10)` and display the top 10 leaderboard entries on the end-of-run screen, where each entry shows the entry's rank (1–10), player name (up to 32 characters), and score.
4. IF `submitScore` fails due to a network error, THEN THE Grave_Runner SHALL retain the calculated score in local storage and display a player-activated retry button that re-attempts `submitScore` with the retained score when selected.
5. IF the player has at least one completed run, THEN THE Grave_Runner SHALL display the player's highest recorded score on the lobby screen before each run.
6. IF `getTopScores(10)` fails due to a network error, THEN THE Grave_Runner SHALL display an error message indicating the leaderboard is unavailable and show the player's most recently cached leaderboard data if available.

---

### Requirement 11: Grave Runner — Accessibility and Controls

**User Story:** As a player, I want keyboard and touch controls to work reliably, so that I can play on any device.

#### Acceptance Criteria

1. WHEN the player presses the spacebar or ArrowUp key, or taps a point with a y-coordinate at or above 50% of the viewport height, THEN THE Grave_Runner SHALL register a jump action, provided the game is in an active running state.
2. WHEN the player presses the ArrowDown key, or performs a downward swipe with a minimum vertical displacement of 30px, THEN THE Grave_Runner SHALL register a slide action, provided the game is in an active running state.
3. WHEN a touch-capable device is detected at game start, THE Grave_Runner SHALL display on-screen icons indicating the jump tap zone and the swipe-down slide gesture.
4. THE Grave_Runner SHALL not require a mouse for any game interaction.
5. WHEN the browser tab loses focus, THE Grave_Runner SHALL pause the game loop and preserve the current game state, including score and obstacle positions.
6. WHEN the browser tab regains focus, THE Grave_Runner SHALL resume the game loop from the preserved state and clear any inputs that were active at the time of focus loss.
7. IF the player triggers any jump or slide input while the game loop is paused, THEN THE Grave_Runner SHALL not register any game action.

---

### Requirement 12: Séance — Question and AI Generation

**User Story:** As a user, I want to ask a spirit a question and receive an AI-generated answer and portrait, so that I get a unique, personalised séance experience.

#### Acceptance Criteria

1. WHEN the user submits a non-empty question of at most 500 characters, THE Seance SHALL deduct 1 Soul via `debitSouls(1)` and then call `generateText(prompt)` via AIService, displaying a loading indicator until the text answer is returned or 30 seconds have elapsed.
2. WHEN the AI text answer is returned within 30 seconds, THE Seance SHALL call `generateImage(prompt)` via AIService with a prompt derived from the answer, displaying a loading indicator until the portrait is returned or 30 seconds have elapsed.
3. WHEN both the answer and portrait are available, THE Seance SHALL display the spirit's answer text and portrait image together in the séance scene.
4. IF the user submits an empty or whitespace-only question, THEN THE Seance SHALL display a validation message and SHALL NOT call AIService.
5. IF the user submits a question exceeding 500 characters, THEN THE Seance SHALL display a validation message and SHALL NOT call AIService.
6. IF AIService returns an `AIGenerationError` or 30 seconds elapse without a response, THEN THE Seance SHALL display an error message indicating the generation failed, refund 1 Soul via `creditSouls(1)`, and present a retry option that returns the user to the question input state with the previous question text preserved.

---

### Requirement 13: Séance — Animated Séance Scene

**User Story:** As a user, I want an atmospheric animated scene during and after the séance, so that the experience feels immersive and spooky.

#### Acceptance Criteria

1. WHEN generation begins, THE Seance SHALL apply the Design_System fog effect to the séance scene container, and the fog effect SHALL remain active until the user navigates away from the séance scene.
2. WHILE generation is in progress, THE Seance SHALL display the Design_System candle-flicker animation on the séance scene container.
3. WHEN generation completes, THE Seance SHALL stop the candle-flicker animation.
4. WHEN the portrait is displayed, THE Seance SHALL animate the portrait's appearance using a CSS fade-in with a duration between 1 second and 2 seconds.
5. WHEN the spirit's answer becomes available, THE Seance SHALL display the answer text with a typewriter animation, revealing characters sequentially at a rate of 30 characters per second.
6. THE Seance SHALL be fully operable with keyboard navigation alone, with all interactive elements reachable via Tab key, activated via Enter or Space, and each focused element displaying a visible focus indicator.

---

### Requirement 14: Séance — Local Gallery

**User Story:** As a user, I want a gallery of my past séances, so that I can revisit previous answers and portraits.

#### Acceptance Criteria

1. WHEN a séance completes successfully, THE Seance SHALL persist the question, answer text, portrait image URL, and an ISO 8601 timestamp to `localStorage` under a key namespaced with a fixed application prefix (e.g., the same prefix for all entries, distinguishable from non-application keys).
2. WHEN the user navigates to the gallery view, THE Seance SHALL display all past séance entries retrieved from `localStorage`, ordered by timestamp with the most recent entry first.
3. WHEN the user selects a gallery entry, THE Seance SHALL display the question, answer text, and portrait image for that entry without any editable input fields or submission controls.
4. IF the number of stored gallery entries reaches 20 and a new séance completes, THEN THE Seance SHALL remove the oldest entry from `localStorage` before persisting the new entry, maintaining a maximum of 20 entries at all times.
5. IF `localStorage` is unavailable or write fails, THEN THE Seance SHALL display a message indicating that the result could not be saved and complete the séance without persisting the entry.
6. WHEN the user activates the delete control for a gallery entry, THE Seance SHALL remove that entry from `localStorage` and re-render the gallery view reflecting the removal within 100 milliseconds.

---

### Requirement 15: Repository — Structure and Documentation

**User Story:** As a developer, I want a documented monorepo with a quickstart README, so that I can understand and extend the project within minutes.

#### Acceptance Criteria

1. THE Skeleton SHALL be structured as an npm workspaces monorepo with the root `package.json` declaring a `workspaces` field listing `/skeleton/`, `/apps/grave-runner/`, and `/apps/seance/`.
2. THE Skeleton SHALL include a root `README.md` containing: a project overview section describing the purpose of the monorepo and the role of each workspace package; a prerequisites section listing the minimum required Node.js major version and npm major version; a quickstart section with separate `npm install` and `npm run dev` commands shown individually for each of the two apps (`grave-runner` and `seance`); and a "Create a new app" section explaining how to copy the `/skeleton/` package, rename its directory, register it in the root `package.json` `workspaces` field, and edit `config.ts` to set the app name and entry point.
3. THE Skeleton SHALL include a `LICENSE` file containing the full MIT license text.
4. WHEN `npm install` is run at the workspace root, THE Skeleton SHALL install all dependencies for all workspace packages in a single command without requiring separate installs in individual package directories.
5. THE Skeleton SHALL pin all `@idosgames/*` package versions to exact semver values (no `^` or `~` ranges) in every `package.json` where they appear.
6. THE Skeleton SHALL not include any file, comment, or environment variable that contains a secret, credential, private key, or API key.
