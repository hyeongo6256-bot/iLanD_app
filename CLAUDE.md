# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Overview

iLanD (82부동산 · "섬을 분양한다") is a single-file, no-build static PWA demo built for the 2026 Seoul Design Festival. An NFC tag on a physical product opens the site, which walks through an onboarding flow into a small social app where each visitor "owns" a randomly-allocated island. There is no backend — all state lives in the browser (`localStorage`) and the whole app is one `index.html` file (~1000+ lines: inline `<style>`, inline screen markup, one inline `<script>` IIFE at the bottom).

## Commands

- Local preview: `python3 -m http.server 8934` from the repo root, then open `http://localhost:8934/` (any static file server works — there is no build step).
- Deploy: `npx vercel deploy --prod --yes --scope nohyeongo` (Vercel project `iland-nfc-demo` is already linked via `.vercel/project.json`; the `--scope nohyeongo` is required because the linked org differs from the default scope).
- No package manager, linter, formatter, or test suite — there is no `package.json`.

## Architecture

### Screen state machine
The whole onboarding flow is a set of sibling `<div class="screen" data-step="N">` elements inside `#frame`, toggled by a single `step` variable in the bottom `<script>` IIFE. `goTo(n)` sets `step`, calls `render()` (toggles `.active` on the matching `.screen`), and runs any per-step side effects (starting/stopping videos, prefilling forms, initializing the 3D viewer). `advance()` is `goTo(step + 1)`.

- **Step 0** — NFC-detected tap screen. Tapping anywhere calls `advance()`.
- **Step 1** — "welcome" screen: full-bleed `#welcome-video` (`assets/iland-intro.mp4`), played at 1.35x. Tapping the screen skips it (no auto-advance).
- **Step 2** — coordinate "dial" screen: a drifting world-map background (`.coord-map-drift`, `assets/world-map.png`, looping `background-position-x` animation) with `#dial-video` (`assets/intro-dial.mp4`) layered on top via `mix-blend-mode: screen` (its black background disappears, only the bright baked-in text shows) plus a `filter: brightness(1.5)` boost since the source video's "white" tops out around 90%. The video fades in via its `playing` event (avoids a black flash before the first frame decodes). Advancing requires a swipe-up gesture on `#dial-drag` (`initSwipeUp()`), not a tap — the numbers shown are fixed video content, not the actual random plot data.
- **Step 3** — basic info form (nickname + island name), gates the confirm button on both fields being non-empty, then calls `goTo(4)`.
- **Step 4** — the main app: a bottom tab bar (`showTab('home'|'explore'|'chat')`) swapping between three full-panel `#tab-*` divs.

Only steps not in `{2, 3, 4}` get the generic tap-to-advance click handler (see the `screens.forEach` near `goTo`); those three manage their own interaction (swipe gesture, form submit, tab bar).

### localStorage-backed state (no backend)
- `iland_plot_v1` — `{ lat, lng, serial, allocatedAt }`, lazily generated once by `getPlot()` and reused after that. Feeds the "섬 등기부증서" (Island Deed) modal.
- `iland_profile_v1` — `{ nickname, islandName, emotion, diary }`. `emotion` is one of `EMOTIONS = ['고요','그리움','위로','설렘']` picked via chips on the home tab; `diary` is the free-text "오늘의 한 줄" line.
- `iland_chats_v1` — map of `partnerName -> { messages: [...] }`, built lazily via `ensureChat()`/`pushMessage()`. The 커뮤니티 tab shows a conversation-list view first (`#chat-list-view`) and switches to a thread view (`#chat-thread-view`) via `openChatThread(name)`/`closeChatThread()`.
- `iland_hide_a2hs` — dismissal flag for the "Add to Home Screen" banner (`initA2HS()`), which shows OS-specific instructions (iOS/Android/other) unless already running standalone.

### 3D island viewer (main tab)
`initIslandViewer()` lazily boots a Three.js scene into `#island-canvas` the first time step 4 is reached (`typeof THREE === 'undefined'` guard). Three.js (r128, non-module builds) and `GLTFLoader` are loaded from jsdelivr in `<script src>` tags — there is no bundler/npm install for this. The model is `assets/island-sample.glb`, scaled and re-centered by hand in the `GLTFLoader.load` callback. Camera orbit is done manually (no `OrbitControls`): dragging updates `camYaw`/`camPitch`, and `renderIslandLoop()` recomputes `camera.position` from spherical coordinates every frame — dragging right must decrease `camYaw` (this was inverted once and fixed; don't re-flip it without re-checking). `resizeIslandViewer()` caches the last known canvas size and skips `renderer.setSize()`/`updateProjectionMatrix()` when unchanged, since calling them unconditionally every frame caused visible stutter. The canvas and its wrapper use `transform: translateZ(0)` / `will-change: transform` to force their own GPU compositing layer — this works around an iOS Safari bug where the canvas clips incorrectly against its box while the page scrolls. Rendering is paused (`islandRenderActive = false`) whenever the `explore`/`chat` tab is active, and resumed on returning to `home`.

### Legacy / unused assets
`assets/island-model/` (an earlier low-poly "glacier island" OBJ+MTL) and `assets/island-model-beach/` (the OBJ+MTL+PNG source used to produce the current GLB via `obj2gltf`) are **not referenced by `index.html`** — only `assets/island-sample.glb` is loaded. Don't assume every file under `assets/` is live; check `index.html` for what's actually referenced before reusing or deleting one.

### PWA shell
`manifest.json` + `sw.js` make this installable. The service worker (`sw.js`) is network-first with a cache fallback (`fetch().then(cache.put).catch(() => cache.match)`), so it does not need a version bump to pick up new deploys as long as the network is reachable — it only serves stale content if the network request fails.
