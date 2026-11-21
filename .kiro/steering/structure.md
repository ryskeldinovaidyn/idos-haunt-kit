---
inclusion: always
---
# Repository structure

```
/                      npm workspaces root, README, LICENSE (MIT)
/.kiro/                specs, steering, hooks (must stay committed)
/skeleton/             @haunt-kit/skeleton: the reusable template
  src/config.ts        one typed app config (title id, currency code, theme)
  src/services/        wallet.ts, leaderboard.ts, ai.ts (thin wrappers over @idosgames/core)
  src/ui/              design tokens, spooky theme, shared components
  src/boot.ts          sign-in + lobby bootstrap via @idosgames/app-shell
/apps/grave-runner/    app #1 (game) — imports @haunt-kit/skeleton
/apps/seance/          app #2 (AI tool) — imports @haunt-kit/skeleton
```

Rules:
- Apps import the skeleton as a workspace package. Never copy skeleton code into an app.
- Anything two apps both need belongs in the skeleton; anything only one app needs stays in that app.
- Keep the skeleton small and documented: a new app should start from it by copying one folder and editing src/config.ts.
