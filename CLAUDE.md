# g-system

The front door of G-System: the README, the install guide for agents, the diagrams and the
screenshots. It holds no code. Each component lives in its own `g-` repo, linked from the README
and pinned here as a submodule.

**Owner:** Gabriel Penman
**Licence:** PolyForm Noncommercial 1.0.0 (`LICENSE`), source-available. Never call it open source.

## Layout
- `README.md`: what G-System is, the component table, framework and instance, the screenshots.
- `INSTALL.md`: written for an AI agent installing on a fresh Linux server, with the desktop app
  on a Mac.
- `docs/diagrams/*.svg`: hand-authored, no build step.
- `docs/screens/*.png`: redacted screenshots of g-command-web and g-command-mac.
- `.gitmodules`: the nine public framework repos at the root, `branch = main`, pinned to their
  released commits. When a part ships a change the install guide depends on, run
  `git submodule update --remote <part>` and commit the new pin.
- `.gitignore`: `gp-core/` and `gcore/`, so an instance made inside a clone is never added here.

## Rules
- This repo is public. Nothing in it may name a real host, IP, domain, person other than Gabriel,
  client, business, amount of money or secret, and nothing may describe what is in an instance or
  the Brain. The `gp-` projects get one line in the README and no names.
- Diagrams use role labels only: central server, production host, worker host, Mac, phone.
- All prose passes `ganti score` with no absolutes: no em or en dashes, semicolons, ellipses,
  middle dots or emojis.
- When a component is added or renamed, update the README table and the topology diagram together.
- Link only repos that are public. The macOS layer is not yet, so the README mentions it without
  a name or a link.

## Diagrams
Each SVG carries its own `<style>` with g-design tokens: Radix slate light by default and a
`@media (prefers-color-scheme: dark)` block, the chart colours as accents. GitHub renders a README
SVG as an `<img>`, so the media query follows the viewer's OS theme, and scripts, external fonts and
external CSS are all ignored. Keep text in separate `<text>` elements where spacing matters, because
whitespace inside one element collapses. Check a change by rendering each SVG as an `<img>` in
Playwright with `color_scheme` light and dark on `#ffffff` and `#0d1117`, then looking at both PNGs.

## Screenshots
Take them from a second copy of g-command-web on a spare port with throwaway `ADMIN_COOKIE` and
`ADMIN_COOKIE_VALUE`, never the live service or its secret, and stop the copy afterwards. Capture
at 1440x900 with the browser time zone set to Europe/London, since the cards print London time and
the footer prints the browser's. Replace personal text in the DOM before the capture, never fake a
state, and record what each page had replaced. Every cron command becomes "Command not shown",
because the command lines hold home paths. The desktop panel comes from g-command-mac with
`GCM_REDACT` set, run with `TZ=Europe/London` and an en-GB locale for the same reason.
Blur the hours worked on /compute and the Used and Created times in the Workspace 3 and 4 cards.
Wrap the text in a span with `filter: blur(6px)`, because a blurred cell smears its table border.
Quantise PNGs to 256 colours to keep them under about 50 KB.
