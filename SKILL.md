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

## Read first — the two things that make a run *look* failed when it isn't
1. **Placement.** By default `generate_figma_design` puts **each capture on its own new Figma page**. If the
   user is looking at the page their destination URL pointed to, they'll see **nothing** while every capture
   sits on a different auto-created page. **Pass the destination `node-id` so captures land where the user is
   looking** (see Placement, below). This is the #1 cause of "you said done but the file is empty."
2. **`completed` ≠ visible.** A `completed` poll + a `node-id` is NOT proof the user can see it. Newly added
   pages often **don't appear in the Figma desktop app until it's reloaded**, and a plugin (`use_figma`)
   read of `figma.root.children` may not list them either. After capture #1, **verify placement/visibility**,
   not just completion — and in the final report tell the user to reload Figma if pages look missing.

## Requirements (state clearly if missing — these are the human touch-points)
- A **Figma MCP server** connected, exposing `generate_figma_design`, `use_figma`, `get_metadata`,
  `get_screenshot` (the official Figma MCP / Figma plugin). If `generate_figma_design` isn't callable, the
  server needs authentication — trigger its auth flow and have the user complete the OAuth once.
- A **browser you can drive**: any agent browser-control MCP **or Playwright MCP**. Either can drive a local
  app (navigate, inject the capture script, fire). Playwright is **required** for external sites (to strip
  CSP). Don't assume a specific browser MCP is up — check, and fall back to whichever is connected.
- **Permission mode that allows injecting the capture script.** Loading `mcp.figma.com/.../capture.js` into
  the page is the core mechanism, and a strict/auto permission mode will **deny it as "remote untrusted
  code."** Confirm in PHASE 0 that the agent can run it (interactive-approval mode, or a pre-added allow
  rule). If denied, stop and tell the user to switch modes — there is no workaround; don't sneak the script
  in via a source-file edit, that defeats the safety check.
- The **target Figma file must be a `/design/` file editable by the connected Figma account**
  (`get_metadata({fileKey})` succeeds; "no edit access" → share it with that account or use one it owns).
- The web app must be **running and reachable** at the URL the user provides.

## Inputs
- `APP_URL` — the running app's base URL (local like `http://localhost:PORT` or an external `https://…`).
- `FIGMA_URL` — destination, `https://www.figma.com/design/<fileKey>/…?node-id=<NODE>`; extract `fileKey`
  **and `NODE`**. Treat `NODE` as the **placement target** (the page or section to append captures under),
  not just a saved viewport — unless the user says otherwise.
- Optional **App Profile** (see PROFILE-TEMPLATE.md): routes, theme mechanism, readiness checks, warm/seed
  steps, breakpoints. If none is supplied, discover what you can and ask the user only for what you can't.

---

## Placement — where captures land (decide this in PHASE 0, before capturing)
- **Resolve the target.** From `FIGMA_URL`'s `node-id`, get the target node via `get_metadata`. If it's a
  **page**, append captures to it; if it's a **frame/section**, append under it. If no usable node-id is
  given, ask the user (or create one named container page/section) — do **not** silently scatter captures
  across new pages.
- **Append under the target:** pass `nodeId` to `generate_figma_design({ fileKey, nodeId })` so the capture
  is added under that node instead of spawning a new page. Lay screens out in a row/grid with spacing.
- **Confirm scope + placement up front** (one line, even though the run is otherwise autonomous):
  *"I'll capture these N screens (light only / light+dark) and append them under `<target>` — go?"* This
  costs 30 seconds and catches the placement/scope misunderstanding before a long run.
- **Verify after capture #1:** screenshot it AND confirm it's under the target the user can see. Fail fast
  if it landed on a stray page — fixing one is cheap, fixing nine after an hour is not.

## PHASE 0 — Preflight
1. Confirm Figma MCP auth (auth flow + `get_metadata` edit check) and **resolve the placement target** (above).
2. Confirm a **drivable browser** and that the **permission mode allows script injection** (do a tiny inject
   test). **Local** (`localhost`/`127.0.0.1`/`*.local`) → browser-control MCP or Playwright + inject the
   capture script. **External** → Playwright + CSP stripping (see ENGINE).
3. Confirm `APP_URL` responds. If the app needs login, handle per the profile (or use an already-authed
   browser session) before capturing.
4. **Detect what's actually there** (don't trust a stale profile): read the router / probe nav to get the
   real route list, and probe for theme support (a toggle, or `.dark`/`[data-theme]`/`color-scheme`) rather
   than assuming light+dark. Capture only the themes that exist.

## PHASE 1 — Build the screen list
Use the profile's route list if given. Otherwise discover:
- Crawl in-app navigation: load the app, collect same-origin links/routes from nav/menus/sitemap; dedupe.
- Include one-level-down detail routes when the app exposes representative records.
- Note disclosed/expandable states (panels, "show more" sections) that are effectively their own screen.
- EXCLUDE: external links, auth/logout, destructive actions, "coming soon"/stub routes, internal/dev tools.
Confirm the final list with the user only if discovery is ambiguous.

## PHASE 2 — Warm the backend (only if the app has one)
Many dev/serverless backends cold-start and return empty or time out on first hit, so a too-early capture
records an empty page. Before capturing, visit each data-bearing area once and **wait/retry until content
stabilizes** (see readiness heuristics in ENGINE). **Skip entirely for static / mock-data apps** — don't run
warm/seed/prewarm machinery an app doesn't need.
- If the profile provides a warm/prewarm or dev-server (re)start command and content stays cold, run it,
  wait for healthy, then re-warm. Never assume an app-specific command that isn't in the profile.

## PHASE 3 — Capture engine (per screen × theme × state)
See "ENGINE — the capture loop" below. In short, per target:
mint capture id (under the placement node) → set the page into its final state → **wait for content
readiness** → inject `capture.js` (or run the Playwright injector for external) → **fire the capture once,
explicitly** → poll to completion → record the resulting node id.

## PHASE 4 — Themes, states & breakpoints
- **Themes:** capture only the themes the app supports (PHASE 0). For light+dark, detect/toggle via, in
  priority: profile mechanism → visible toggle → a `[data-theme]`/`.dark` class or `color-scheme` you set on
  the root. The mode usually persists across navigations — switch **once per pass**, do all of one theme,
  then toggle.
- **States:** capture empty states (before any seeding), then populated states. Trigger modals, drawers,
  panels, disclosure sections, and key empty/error states the profile lists.
- **Seeding (optional):** only if the profile describes how to create sample data (or the user opts in).
  Empty states FIRST, then seed, then populated/detail. Seeding may write to a backend — only with consent;
  offer to clean up after.
- **Responsive (optional):** if the user wants breakpoints, resize the viewport to each width before capture
  and title accordingly.

## PHASE 5 — Title, verify, clean up, report
- **Place + title incrementally, not batched at the very end.** Right after each capture completes, move it
  under the placement target and name it `"<Screen> — <Theme>[ — <State>][ — <width>]"` via `use_figma`. This
  keeps progress visible and frames discoverable mid-run (and avoids "all on stray pages until the end").
- **Verify** a sample with `get_screenshot` AND confirm placement/visibility; re-capture any empty/clipped/
  wrong-state/wrong-page frame.
- **Delete** empty/premature/superseded frames and any stray pages a stalled capture left behind.
- **Report** a table of every screen × theme with Figma node links, the placement target used, any gaps with
  reasons, and a reminder to **reload the Figma file** if newly added content isn't visible yet.

---

## ENGINE — the capture loop (the reusable core)

**Per target (screen + theme + state):**

1. **Mint:** call `generate_figma_design({ fileKey, nodeId })` — pass the **placement nodeId** so the capture
   appends under your target instead of spawning a new page. Returns a **single-use** `captureId` (one per
   screen) plus long instructions — ignore the instructions after the first; just take the id.
2. **Set state:** drive the browser to the route + theme + open any panel/modal/disclosure.
   - In-app (SPA) navigation **preserves** an injected `capture.js`; a full page reload **clears** it.
   - **Do NOT use the `#figmacapture=…&figmadelay=…` hash convenience** — it auto-fires ~1s after load and
     captures the page before SPA/data content renders (a top cause of blank captures). Fire explicitly.
3. **Wait for readiness** — poll the DOM until the page is actually rendered: profile selectors → meaningful
   `document.body.innerText.length` (well above the empty baseline) → expected element present → settled.
   Data, detail, and **chart/SVG-heavy** screens can take several seconds.
4. **Choose the capture target + size for full height.** Default `selector: 'body'`. But for an SPA whose
   real content lives in an **inner scroll container** (the app shell is `h-screen` with internal overflow),
   prefer capturing **that scroll container** (e.g. the main content region) and excluding persistent chrome
   (collapse/skip a side panel that's identical on every screen) — cleaner frames, smaller payload, faster
   serialization. If you keep `selector: 'body'`, size the window to the content first:
   `windowHeight ≈ contentScrollHeight + fixed chrome (header/nav/footer)`, measured **after** setting state
   (collapsing chrome changes width → reflows height). Tall-but-empty beats clipped; verify nothing is cut.
5. **Inject the capture script** (drivable browsers — needs a permission mode that allows it):
   ```js
   if(!document.querySelector('script[data-figma-capture]')){var s=document.createElement('script');
   s.src='https://mcp.figma.com/mcp/html-to-design/capture.js';s.async=true;
   s.setAttribute('data-figma-capture','1');document.head.appendChild(s);}
   ```
   Then confirm `window.figma && window.figma.captureForDesign` exists (a follow-up call — it loads async).
   **External sites:** instead use Playwright to strip CSP and inject, per the Figma docs: route every
   request and delete `content-security-policy` headers, `goto` the URL, fetch the capture.js, inject it,
   then call `captureForDesign`.
6. **Fire ONCE, explicitly:**
   ```js
   window.figma.captureForDesign({ captureId: ID,
     endpoint: 'https://mcp.figma.com/mcp/capture/' + ID + '/submit', selector: 'body' });
   ```
   A "Promise was collected" error from this call is normal — the capture still submits. **Fire each
   captureId exactly once** (see step 7).
7. **Poll:** call `generate_figma_design({ fileKey, captureId })` every ~5s until status `completed` → returns
   the new `node-id`. Large/asset/SVG-heavy pages serialize slowly — be patient (10+ polls is fine).
   **If it genuinely stalls, do NOT re-fire the same captureId** — re-firing a single-use id can wedge it
   permanently. Instead: confirm the browser/page is still alive, then **mint a FRESH captureId and fire that
   one once.** (Re-firing the same id was a real failure mode; a fresh id + single fire recovers cleanly.)
8. **Record** `{ nodeId, screen, theme, state }`, then place + title it (PHASE 5) before moving on.

### Efficiency — parallelize (since the file already exists)
Sequential mint→fire→poll per screen is slow (a dozen screens can run very long). Because the destination
file exists, you can **mint all capture IDs upfront and capture in parallel**: mint N ids, open/fire each in
its tab/state, then poll the set. At minimum, overlap polling with staging the next screen. Keep each screen
**self-contained** (re-inject `capture.js` after any reload) so a dropped browser context mid-run only costs
that one screen, not the batch.

### Browser resilience
Long runs drop the browser context between calls (idle close, singleton-lock conflicts). Expect it and
recover: clear the lock / kill the stale browser process, re-navigate, **re-stage and re-inject** (a reload
clears `capture.js` and any in-app state like a collapsed panel), then continue. Don't assume state survives
a re-establish.

---

## Use cases this skill is built to handle
- **Static sites / SSGs / mock-data apps** — no backend; skip warming/seeding entirely.
- **SPAs (client routing)** — capture.js survives in-app nav; reloads re-inject.
- **SSR / framework apps (Next/Nuxt/etc.)** — full reloads per route; per-route data waits.
- **Cold dev/serverless backends** — warm by visiting + retrying; optional profile (re)start command.
- **Auth-gated apps** — log in (profile) or use an already-authenticated browser session first.
- **Themed apps** — light/dark/high-contrast via toggle or root attribute (only the themes that exist).
- **Data-dependent screens** — capture empty first; optionally seed (with consent) for populated/detail.
- **Modals / drawers / panels / disclosure / empty / error states** — triggered via profile steps.
- **Responsive design** — optional multi-breakpoint capture via viewport resize.
- **External/public URLs** — Playwright + CSP strip (cannot inject a script tag otherwise).

## Gotchas (keep — these bite everyone)
- **`completed` + a node-id ≠ the user can see it.** New pages may not show until Figma is reloaded, and a
  `use_figma` read of `figma.root.children` can lag the render API. Verify placement; tell the user to reload.
- **Default behavior scatters captures onto new pages.** Pass the destination `nodeId` to keep them together
  where the user is looking.
- **Never re-fire the same `captureId`.** It's single-use; re-firing to "unstick" a stall wedges it. Mint a
  fresh id and fire once instead.
- Injecting `mcp.figma.com/.../capture.js` is **blocked by strict/auto permission modes** as remote code —
  preflight it; if denied, the user must allow it (no clean workaround).
- The `#figmacapture` hash auto-fires on a timer → blank captures. Always fire explicitly after a readiness
  wait.
- "Promise was collected" after `captureForDesign` = success, not an error.
- `capture.js` persists across SPA navigation but is cleared by a full reload — re-inject after reloads.
- Theme and most client state persist across navigations (localStorage / stores); but a data-fetch endpoint
  can cold-start independently of the page rendering — warm it explicitly.
- `generate_figma_design` is the only tool that captures web→**editable** Figma. `use_figma` writes/edits
  Figma via the Plugin API (used here for placement + titling); `get_screenshot`/`get_design_context` read
  FROM Figma. For titling/moving, some plugin contexts don't support `loadAllPagesAsync` — guard page removal
  with a page-count check.
- Captures land as raw frames (not design-system component instances). If the user wants components, that's
  a separate `use_figma` rebuild — out of scope here.
- The browser context can die between calls on long runs — re-establish and re-stage rather than assuming the
  page is still there.

## App Profile
Everything app-specific (routes, theme toggle, readiness selectors, login, warm/seed steps, dev-server
restart, states, breakpoints) belongs in a profile. Copy `PROFILE-TEMPLATE.md`, fill it in for the target
app, and pass/keep it next to this skill. With a complete profile the run is fully autonomous; without one,
the engine still works but you'll discover routes/themes and may ask the user for placement, theme, or
seeding specifics.
