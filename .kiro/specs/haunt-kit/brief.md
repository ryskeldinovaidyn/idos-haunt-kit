# Haunt Kit — project brief (input for the spec)

Build "Haunt Kit", iDos Games' Skeleton Crew entry for Kiroween 2026. Follow the steering docs.

1) skeleton/ — reusable Vite + TypeScript template on the iDos Games SDK. Provides: sign-in and lobby (app-shell); one typed config file; a soft-currency wallet called "Souls"; a leaderboard service; a thin AI wrapper over client.ai (text + image generation with waitForGeneration); a spooky design system (CSS tokens, dark theme, fog and candle effects, accessible contrast); a build that outputs a static bundle for idosgames.com.

2) apps/grave-runner/ — endless runner on HTML canvas through a haunted graveyard. Jump and slide over tombstones and bats, collect souls (credited to the Souls wallet), submit the score to the global leaderboard. 60 fps on mobile and desktop; keyboard and touch controls.

3) apps/seance/ — AI tool. The user asks a spirit a question; the app generates the spirit's answer (AI text) and portrait (AI image), shows them in an animated séance scene, and keeps a local gallery of past séances. Each séance costs 1 Soul.

Constraints: both apps import the skeleton as a workspace package; no secrets; MIT license; README with quickstart; each app builds to its own static bundle.
