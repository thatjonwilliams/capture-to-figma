---
name: capture-to-figma
description: "Capture a running web app's screens into an EDITABLE Figma design file using Figma's code-to-canvas (generate_figma_design) — real editable frames, not screenshots. Use when the user says 'capture to Figma', 'capture all pages/screens to Figma', 'push the app/site into Figma', 'code to canvas', or 'send screens to Figma as editable layers'. Works across locally-hosted and external web apps; captures multiple themes/states with content-readiness waits, optional backend warming, titling, and cleanup. Inputs: the app URL + a destination Figma design-file URL."
license: CC0-1.0
---

# web → Figma (autonomous code-to-canvas capture)

Capture the screens of a running web app into a Figma design file as **editable frames** using Figma's
**code-to-canvas** tool (`generate_figma_design`). The output is real, layer-editable Figma frames, not flat
images. The engine below is app-agnostic; everything app-specific lives in a small **App Profile** you fill
in (a template ships alongside this skill). Run it end-to-end with minimal user involvement.

> Docs: https://developers.figma.com/docs/figma-mcp-server/code-to-canvas/

## Requirements (state clearly if missing — these are the only human touch-points)
- A **Figma MCP server** connected, exposing `generate_figma_design`, `use_figma`, `get_metadata`,
  `get_screenshot` (the official Figma MCP / Figma plugin). If `generate_figma_design` isn't callable, the
  server needs authentication — trigger its auth flow and have the user complete the OAuth once.
- A **browser you can drive**: a Claude/agent browser-control MCP (preferred for local apps — lets you
  inject the capture script) and/or **Playwright MCP** (needed for external sites, to bypass CSP).
- The **target Figma file must be a `/design/` file editable by the connected Figma account**
  (`get_metadata({fileKey})` succeeds; "no edit access" → share it with that account or use one it owns).
- The web app must be **running and reachable** at the URL the user provides.

## Inputs
- `APP_URL` — the running app's base URL (local like `http://localhost:PORT` or an external `https://…`).
- `FIGMA_URL` — destination, `https://www.figma.com/design/<fileKey>/…`; extract `fileKey`.
- Optional **App Profile** (see PROFILE-TEMPLATE.md): routes, theme mechanism, readiness checks, warm/seed
  steps, breakpoints. If none is supplied, discover what you can and ask the user only for what you can't.

---

## PHASE 0 — Preflight
1. Confirm Figma MCP auth (auth flow + `get_metadata` edit check).
2. Confirm a drivable browser. **Local** (`localhost`/`127.0.0.1`/`*.local`) → use the browser-control MCP
   and inject the capture script. **External** → use Playwright MCP with CSP stripping (see ENGINE).
3. Confirm `APP_URL` responds. If the app needs login, handle per the profile (or use an already-authed
   browser session) before capturing.

## PHASE 1 — Build the screen list
Use the profile's route list if given. Otherwise discover:
- Crawl in-app navigation: load the app, collect same-origin links/routes from nav/menus/sitemap; dedupe.
- Include one-level-down detail routes when the app exposes representative records.
- EXCLUDE: external links, auth/logout, destructive actions, and internal/dev-only tools.
Confirm the final list with the user only if discovery is ambiguous.

## PHASE 2 — Warm the backend (only if the app has one)
Many dev/serverless backends cold-start and return empty or time out on first hit, so a too-early capture
records an empty page. Before capturing, visit each data-bearing area once and **wait/retry until content
stabilizes** (see readiness heuristics in ENGINE). Skip entirely for static sites.
- If the profile provides a warm/prewarm or dev-server (re)start command and content stays cold, run it,
  wait for healthy, then re-warm. Never assume an app-specific command that isn't in the profile.

## PHASE 3 — Capture engine (per screen × theme × state)
See "ENGINE — the capture loop" below. In short, per target:
mint capture id → set the page into its final state → **wait for content readiness** → inject `capture.js`
(local) or run the Playwright injector (external) → **fire the capture explicitly** (full-height) → poll to
completion → record the resulting node id.

## PHASE 4 — Themes, states & breakpoints
- **Themes:** capture light and dark (and high-contrast if present). Detect/toggle via, in priority: the
  profile's theme mechanism → a visible theme toggle control → a `[data-theme]`/`.dark` class or
  `color-scheme` override you set on the root. The mode usually persists across navigations.
- **States:** capture empty states (before any seeding), then populated states. Trigger modals, drawers,
  panels, and key empty/error states the profile lists (these are app-specific).
- **Seeding (optional):** only if the profile describes how to create sample data (or the user opts in).
  Capture empty states FIRST, then seed, then populated/detail screens. Seeding may write to a backend —
  do it only with consent and offer to clean up after.
- **Responsive (optional):** if the user wants breakpoints, resize the viewport to each width before capture
  and title accordingly.

## PHASE 5 — Title, verify, clean up, report
- **Title** each captured frame (and its wrapping Figma page) `"<Screen> — <Theme>[ — <State>][ — <width>]"`
  via `use_figma`.
- **Verify** a sample with `get_screenshot`; re-capture any empty/clipped/wrong-state frame.
- **Delete** empty/premature/superseded frames.
- **Report** a table of every screen × theme with Figma node links.

---

## ENGINE — the capture loop (the reusable core)

**Per target (screen + theme + state):**

1. **Mint:** call `generate_figma_design({ fileKey })` (no captureId) → a **single-use** `captureId`
   (one per screen). It returns long instructions — ignore them after the first; just take the id.
2. **Set state:** drive the browser to the route + theme + open any panel/modal.
   - In-app (SPA) navigation **preserves** an injected `capture.js`; a full page reload **clears** it.
   - **Do NOT use the `#figmacapture=…&figmadelay=…` hash convenience** — it auto-fires ~1s after load and
     captures the page before SPA/data content renders (the #1 cause of blank captures). Fire explicitly.
3. **Wait for readiness** — poll the DOM until the page is actually rendered, using whichever applies:
   profile-provided selectors → meaningful `document.body.innerText.length` (well above the empty baseline)
   → expected element present → network/quiet settled. Data and detail screens can take several seconds.
4. **Full-height:** capture with `selector: 'body'`. If content lives in an inner scroll container whose
   height equals the viewport, resize the window tall (e.g. width×3000) so the whole page is laid out, then
   verify nothing is clipped.
5. **Inject the capture script** (local apps):
   ```js
   if(!document.querySelector('script[data-figma-capture]')){var s=document.createElement('script');
   s.src='https://mcp.figma.com/mcp/html-to-design/capture.js';s.async=true;
   s.setAttribute('data-figma-capture','1');document.head.appendChild(s);}
   ```
   Then confirm `window.figma && window.figma.captureForDesign` exists (a follow-up call — it loads async).
   **External sites:** instead use Playwright to strip CSP and inject, per the Figma docs:
   route every request and delete `content-security-policy` headers, `goto` the URL, fetch
   `https://mcp.figma.com/mcp/html-to-design/capture.js`, inject it, then call `captureForDesign`.
6. **Fire explicitly:**
   ```js
   window.figma.captureForDesign({ captureId: ID,
     endpoint: 'https://mcp.figma.com/mcp/capture/' + ID + '/submit', selector: 'body' });
   ```
   A "Promise was collected" error from this call is normal — the capture still submits.
7. **Poll:** call `generate_figma_design({ fileKey, captureId })` every ~5s until status `completed`
   → returns the new `node-id`. If it stays pending past ~6 polls, re-fire once (the state may have
   re-rendered); large/asset-heavy pages serialize slowly.
8. **Record** `{ nodeId, screen, theme, state }` for titling; verification is batched in PHASE 5.

---

## Use cases this skill is built to handle
- **Static sites / SSGs** — no backend; skip warming.
- **SPAs (client routing)** — capture.js survives in-app nav; reloads re-inject.
- **SSR / framework apps (Next/Nuxt/etc.)** — full reloads per route; per-route data waits.
- **Cold dev/serverless backends** — warm by visiting + retrying; optional profile (re)start command.
- **Auth-gated apps** — log in (profile) or use an already-authenticated browser session first.
- **Themed apps** — light/dark/high-contrast via toggle or root attribute.
- **Data-dependent screens** — capture empty first; optionally seed (with consent) for populated/detail.
- **Modals / drawers / panels / empty / error states** — triggered via profile steps.
- **Responsive design** — optional multi-breakpoint capture via viewport resize.
- **External/public URLs** — Playwright + CSP strip (cannot inject a script tag otherwise).

## Gotchas (keep — these bite everyone)
- `generate_figma_design` is the only tool that captures web→**editable** Figma. `use_figma` writes/edits
  Figma via the Plugin API (used here for titling); `get_design_context`/`get_screenshot` read FROM Figma.
- The `#figmacapture` hash auto-fires on a timer → blank captures. Always fire explicitly after a readiness
  wait.
- "Promise was collected" after `captureForDesign` = success, not an error.
- Each `captureId` is single-use (one screen). Mint a fresh one per screen.
- `capture.js` persists across SPA navigation but is cleared by a full reload — re-inject after reloads.
- Theme and most client state persist across navigations (localStorage / stores); but a data-fetch endpoint
  can cold-start independently of the page rendering — warm it explicitly.
- For titling, prefer the Figma MCP's `use_figma`; some plugin contexts don't support `loadAllPagesAsync`,
  so guard page removal with a page-count check.
- Captures land as raw frames (not design-system component instances). If the user wants components, that's
  a separate `use_figma` rebuild — out of scope here.

## App Profile
Everything app-specific (routes, theme toggle, readiness selectors, login, warm/seed steps, dev-server
restart, states, breakpoints) belongs in a profile. Copy `PROFILE-TEMPLATE.md`, fill it in for the target
app, and pass/keep it next to this skill. With a complete profile the run is fully autonomous; without one,
the engine still works but you'll discover routes and may ask the user for theme/seeding specifics.
