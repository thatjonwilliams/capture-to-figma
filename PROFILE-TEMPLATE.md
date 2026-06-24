# App Profile — <APP NAME>

Copy this file, fill it in for your app, and keep it with the `web-to-figma` skill (or paste it into your
request). Anything you leave blank, the skill will try to discover or will ask about. A complete profile =
a fully autonomous run.

## Basics
- **App URL:** `http://localhost:<PORT>`  (or external `https://…`)
- **Destination Figma file:** `https://www.figma.com/design/<fileKey>/<name>?node-id=<NODE>`
- **Placement target:** the `<NODE>` above = the page/section to append every capture under (so they land
  where you're looking, not on scattered new pages). Leave blank to create a fresh container page.
- **Auth:** none | dev-bypass | login required (if login required, give the steps or use a pre-authed browser)

## Routes (user-facing screens to capture)
List the routes. Mark which need a record id and how to get one.
- `/`  — home
- `/dashboard`
- `/items`  — list  (detail: click first `<selector>` → `/items/:id`)
- … (EXCLUDE: external links, logout, destructive actions, internal/dev tools)

## Readiness checks (how to know a screen finished rendering)
Per route (or a sensible default): a CSS selector that appears when loaded, and/or a min text length.
- default: `document.body.innerText.length > <N>`
- `/items`: `[data-…="item-card"]` present (or "empty" message)
- chat/detail/data screens often need several seconds — note any slow ones.

## Theme mechanism (for light/dark/high-contrast)
- How to switch: e.g. click `[aria-label="Toggle theme"]` | set `documentElement.classList` `dark` |
  set `data-theme` | localStorage key `<key>`.
- How to detect current mode: e.g. `documentElement.classList.contains('dark')`.
- Persists across navigation? yes/no.

## Backend warming (if the app has a backend that cold-starts)
- Data areas to warm before capturing: `<routes/endpoints>`.
- Optional dev-server / prewarm (re)start command (run only if warming fails):
  ```
  <command to (re)start the local stack, e.g. your dev/start script>
  ```
- Any endpoint that warms separately / is especially slow: `<note>` (e.g. a messages/stream API).

## Data seeding (optional — only if you want populated/detail screens and consent to writes)
- Create a sample record: e.g. open `<route>` → click `<New>` → fill `<#field>` (use a React/Vue-safe value
  setter: native setter + dispatch `input`+`change`) → click `<Create>`. Click once; verify count.
- Other sample data needed for detail/state screens: `<steps>`.
- Clean up after? yes/no.

## Extra states to capture (app-specific)
- Empty states: `<routes>` (capture BEFORE seeding)
- Modals/drawers/panels: `<how to open each>`
- Error/loading states: `<how to trigger, if desired>`

## Responsive breakpoints (optional)
- Widths to capture each screen at: e.g. `390` (mobile), `768` (tablet), `1440` (desktop)

## Notes
- Anything else the agent should know (feature flags that hide routes, seeded test account, etc.).
