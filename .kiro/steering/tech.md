---
inclusion: always
---
# Tech stack

- Language: TypeScript (strict). Build: Vite. Package manager: npm workspaces.
- iDos Games SDK packages, pinned to exact versions:
  - @idosgames/core 0.26.3 (client, auth, currencies, leaderboards, client.ai)
  - @idosgames/app-shell 0.16.7 (sign-in + lobby)
  - @idosgames/react 0.7.2
  - @idosgames/module-sdk 0.13.4
  - @idosgames/wallet 0.6.10
- AI at runtime goes only through client.ai (AIService) and client.ai.waitForGeneration. Never call third-party AI APIs directly and never ship API keys.
- Rendering: plain HTML canvas for the game; DOM + CSS for UI. No heavy game engines.
- Output: each app builds to a static bundle (index.html + assets) that can be uploaded as a build to idosgames.com.
- Quality: no secrets in the repo; ESLint + Prettier defaults; unit tests with Vitest for pure logic (scoring, wallet math, spawn rules); property-based tests with fast-check where rules are numeric.
- License: MIT.
