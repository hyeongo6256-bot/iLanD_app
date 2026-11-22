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

## Design rules (디자인 규칙 — 반드시 지킬 것)

디자이너(사용자)가 정한 규칙이다. 디자인/스타일 작업 전에 항상 이 섹션을 먼저 읽는다. 여기 없는 값을 새로 만들지 말고, 꼭 필요하면 먼저 제안하고 승인받는다.

### 작업 방식
- **요청한 범위만 수정한다.** 요청에 없는 요소의 색·크기·간격·위치는 건드리지 않는다. 비슷해 보이는 다른 곳도 같이 바꾸고 싶으면 먼저 물어본다.
- **수정 전에 계획을 짧게 말한다.** 어떤 요소(id/위치)를 어떤 값에서 어떤 값으로 바꿀지 먼저 말하고 진행한다. 해석이 모호하면 추측하지 말고 질문한다.
- **"조금 더 크게/여유 있게" 같은 모호한 요청**은 아래 단계표에서 바로 다음 단계로만 이동한다. 단계 사이 임의 값(예: 12.5 → 12.8)을 만들지 않는다.
- **레퍼런스 이미지가 주어지면** 색은 이미지에서 픽셀 샘플링해 HEX로 확인하고, 크기·간격은 비율로 측정해서 근거와 함께 반영한다. 눈대중으로 해석하지 않는다.
- **색을 바꿀 때는 같은 역할의 색이 쓰인 모든 곳을 grep으로 찾아서** 함께 바꾸고, 바꾼 위치 목록을 보고한다. 특히 같은 색을 따라가야 하는 곳: Google Maps `backgroundColor`(바다색), `tabBg` 맵(탭바 뒷배경), `EXPLORE_MAP_STYLE`.
- 되돌리기 쉽게, 의미 있는 단위로 작업이 끝나면 커밋을 제안한다.

### 컬러 (역할별)
| 역할 | HEX | 비고 |
|---|---|---|
| **메인 핑크** | `#f4aec1` | 브랜드 기준 핑크. 새로 핑크가 필요하면 무조건 이 값 |
| 브라운 (포인트/로고/주요 버튼) | `#663b2c` | |
| 텍스트 기본 (다크) | `#2b2622` | |
| 텍스트 보조 | `#8a8074` | 설명문, placeholder |
| 텍스트 약함 | `#a39a8d` | 캡션, 라벨 |
| 크림 (어두운/컬러 배경 위 텍스트) | `#f3ede4` | |
| 바다 블루 (둘러보기 지도) | `#78abbe` | |
| 화이트 | `#ffffff` | 시트, 카드 |
| 날씨 점 | `#ffcc01` | 홈 헤더 날씨 표시 |

CSS 변수(`index.html` `:root`): `--pink`, `--pink-soft`(핑크 60%, 흰 배경 위 연핑크 카드), `--brown`, `--brown-50`(placeholder·슬라이더 남은 구간·잠금 안내), `--brown-35`(비활성 칩), `--weather-dot`. 새 스타일은 HEX 대신 이 변수를 쓴다.

⚠️ 아래 핑크/브라운 계열은 메인 핑크와 거의 같은 **정리 대상 값**이다. 새 코드에 쓰지 말고, 기존 위치를 고칠 때 메인 핑크로 통일할지 사용자에게 확인한다:
`#f5bacb`(홈 배경), `#f3adc0`(지도 땅), `#f7acc0`(지도 기본색 모드), `#e8aab7`(온보딩), `#bf898c`(감정 칩), `#9c6a60`(보조 브라운 텍스트)

### 타이포
- 한글 본문/UI: `Noto Sans KR` (400/500/600/700)
- 영문 이탤릭 라벨·제목 (TODAY'S MOOD 등): `Playfair Display` italic
- 숫자·좌표·코드 느낌: `Space Mono`
- 섬 이름(로고형 디스플레이): `Blazeface Hangeul 18pt`(한글) / `Ohno Blazeface 18pt Italic`(영문·숫자 폴백) — Adobe Fonts 라이선스 구매, `assets/fonts/`에 자체 호스팅(`@font-face`). 한글 짝꿍 서체가 이탤릭 컷에 맞춰 디자인돼 있어 영문도 이탤릭을 쓴다. 이 외 용도로 쓰지 않는다
- 이 외 새 폰트를 추가하지 않는다.

**글자 크기 단계(px)** — 새로 쓰거나 수정할 때는 이 값만 쓴다:
`10.5` 캡션 · `11` 라벨 · `12.5` 보조 본문 · `13.5` 본문 · `16` 입력창(iOS 줌 방지 최소값, 입력 요소는 16 미만 금지) · `17` 카드 강조 문구 · `19` 섹션 제목 · `23` 홈 시트 큰 제목 · `25` 디스플레이
(기존 코드의 10, 11.5, 12, 13, 14, 15, 17, 18, 23 등은 정리 대상 — 해당 부분을 수정할 때 가까운 단계로 맞출지 확인한다.)

### 모서리 반경
- 알약형 버튼/칩/입력창: `999px`
- 카드: `16px` / 홈 시트 안 달력·인사이트·기록 카드: `24px`
- 감정 선택 박스: `22px`
- 바텀 시트 상단(홈): `40px 40px 0 0` (키보드 화이트아웃 `setHomeModelWhiteout`도 같은 값 유지)
- 작은 아이콘 버튼(사각): `12px`
