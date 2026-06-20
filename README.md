# capture-to-figma

A Claude skill that captures a **running web app's screens into an editable Figma design file** using
Figma's [code-to-canvas](https://developers.figma.com/docs/figma-mcp-server/code-to-canvas/)
(`generate_figma_design`). Output is **real, layer-editable Figma frames — not screenshots.**

It bakes in the things that make unattended capture actually work: content-readiness waits (so it never
captures a half-rendered page), optional backend warming for cold dev/serverless APIs, light/dark/state
coverage, automatic frame titling, and cleanup. App-specific details live in a small, swappable **App
Profile**, so the same engine works across static sites, SPAs, SSR apps, and authenticated dashboards.

## What it does
- Discovers (or takes) the list of screens to capture.
- Warms the backend if the app has one that cold-starts.
- For each screen × theme (× state × breakpoint): waits for content, injects Figma's capture script, fires
  the capture, and polls until the editable frame lands in your file.
- Captures empty states, and optionally seeds sample data (with consent) for populated/detail screens.
- Titles every frame and its page, deletes stray/empty frames, and reports a table of Figma links.

## Requirements
- **A Figma MCP server** connected in your agent, exposing `generate_figma_design`, `use_figma`,
  `get_metadata`, `get_screenshot` (the official Figma MCP). You complete its OAuth once.
- **A browser you can drive**: an agent browser-control MCP (for local apps — to inject the capture script)
  and/or **Playwright MCP** (required for external/public URLs, to bypass CSP).
- A **Figma `/design/` file** editable by the connected Figma account.
- The web app **running** at a URL you provide.

## Install
Copy this folder into your agent's skills directory, e.g.:

```
~/.claude/skills/web-to-figma/
```

(or wherever your agent loads skills from). The skill is self-contained: `SKILL.md` is the workflow,
`PROFILE-TEMPLATE.md` is the per-app config template.

## Usage
Ask your agent, for example:

> Capture every screen of `http://localhost:3000` into `https://www.figma.com/design/<fileKey>/<name>`

For a fully autonomous run, fill in `PROFILE-TEMPLATE.md` for your app (routes, theme toggle, readiness
checks, optional warm/seed steps) and provide it. Without a profile it still works — it discovers routes and
asks only for what it can't infer.

## Notes & limits
- Captures land as **raw frames**, not design-system component instances. Mapping to components is a
  separate Figma `use_figma` rebuild and is out of scope here.
- Seeding sample data may write to your backend — the skill only does this with your consent and offers to
  clean up.
- This skill drives your browser and your Figma account through MCP tools; review what your agent runs.

## License
[CC0 1.0 Universal](LICENSE) — public-domain dedication. No rights reserved; use it freely, no attribution
required (though a star or credit is always appreciated).
