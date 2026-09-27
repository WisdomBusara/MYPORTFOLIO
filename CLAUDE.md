# CLAUDE.md

Context and rules for working on this repository with Claude Code. Read this before making changes.

## What this is

The personal portfolio site for **Ian Kimani** (GitHub: @WisdomBusara), hosted at **wisdombusara.com**.

It is a single, self-contained static site. One HTML file holds all markup, CSS, and JavaScript. There is no framework, no bundler, no package manager, and no build step. You edit `index.html` directly and deploy the file as is.

**Positioning (do not drift from this):** Ian is presented as a software engineer and builder focused on **automation, AI agents, and full-stack products**. Banking and payments integration is a supporting credential, not the headline. Do not re-center the site on banking.

## Golden rules

These are hard constraints, not suggestions.

1. **Never use em-dashes** (Unicode U+2014) anywhere: copy, HTML comments, CSS comments, JS comments, shell scripts, commit messages, or your own chat replies. Use commas, colons, or parentheses instead. After any edit, verify none crept in with `grep -c $'\xe2\x80\x94' index.html` (Git Bash) and expect `0`. Do not use `grep -P` for this: on this Windows machine it errors out, and an `||` fallback will then report a false pass. This is a persistent, repeated requirement.
2. **Keep it self-contained.** No new external runtime dependencies without asking first. The one third-party library, Three.js, is vendored locally at `wisdombusara-site/assets/three.min.js` (served as `/assets/three.min.js`). If a new dependency is genuinely needed, vendor it the same way rather than adding a CDN call.
3. **No fabricated content.** Only real projects, skills, and facts go on the site. Public repos link to real code. Private repos are described at the category level only. Do not invent metrics, features, or claims that cannot be verified. If asked to add something unverifiable, say so instead of inventing it.
4. **Preserve accessibility.** Semantic HTML, correct heading order, keyboard navigation, visible focus states, alt text, and sufficient contrast in both themes.
5. **Respect reduced motion.** Every animation (the WebGL mesh, the pipeline replay, card tilt, scroll reveals, the tour's staggered stops) is gated behind `prefers-reduced-motion`. Keep that gating on anything animated you add.
6. **No unguarded browser storage.** `localStorage` is used only for the theme, wrapped in `try/catch` so it fails silently where storage is blocked. Keep any storage access wrapped the same way.
7. **The repo is public.** Anything committed is visible to everyone, including this file. Never commit private details (employer, credentials, server IPs).

## Repository layout

```
index.html            # the entire site: inline CSS + JS, single file (edit this one)
robots.txt
sitemap.xml
wisdombusara-site/    # deploy bundle: the web root as it sits on the VPS
  index.html          # identical copy of the root index.html; keep them in sync
  robots.txt
  sitemap.xml
  assets/
    three.min.js      # Three.js r128, vendored (the only JS library)
RUNBOOK.md            # VPS setup, update, rollback and troubleshooting steps
CLAUDE.md             # this file
```

## Running locally

The page references `/assets/three.min.js` by absolute path, and that file only exists inside the bundle folder. Serve the bundle, not the repo root:

```bash
cd wisdombusara-site && python -m http.server 8000
# then open http://localhost:8000
```

Everything works offline except two visitor-side calls: Google Fonts (styling) and `api.github.com` (the live repo panel, which has a static fallback). Neither is required for the page to function.

`curl` cannot see JavaScript errors. To confirm "no console errors", load the page in a real browser (headless Chrome over the DevTools protocol works on this machine) and exercise the theme toggle, Cmd/Ctrl+K, filters, and the tour.

## How index.html is organized

It is one large file, so here is the map before you go editing.

- **`<head>`**: metadata, Open Graph (`og:type` profile), an inline SVG favicon, a JSON-LD graph (Person, WebSite, ProfilePage, and an ItemList of the six public repos), Google Fonts link, an inline FOUC-prevention script that sets `data-theme` before paint.
- **`<style>`**: CSS custom properties on `:root` (dark) with a `html[data-theme="light"]` override block. Colours, fonts, and spacing are all tokens. The print stylesheet is at the bottom.
- **Body sections**: sticky nav (with theme toggle and command-palette button), hero, about, projects (the guided tour), "how I build", skills, trajectory, github, contact, footer, the project-tour dialog, and the command-palette dialog.
- **`<script>`**: one IIFE. In order: mobile menu, scroll reveal, scroll-spy nav, replayable pipeline, project filters and tour stops, guided tour dialog, provider-adapter demo, "how I build" stage switcher, copy-email, live GitHub explorer with static fallback, command palette, theme toggle, 3D pointer tilt, and the hero WebGL mesh. An exception anywhere in this IIFE silently kills every feature after it, so never call methods on a lookup that can return null.

### Design tokens (keep these consistent)

- Theme: "Console Bold". Dark background `#0C0C0E`, amber accent `#F5A524`. Light theme shifts the accent to a readable ochre `#B4740A`. `--accent-soft` is the translucent accent used for glows.
- Fonts: Bricolage Grotesque (display and body), IBM Plex Mono (code, labels, technical data).
- Change colours via the CSS variables, never by hardcoding hex values in individual rules.

### Interactive features already present

Command palette (Cmd/Ctrl+K or `/`), dark/light toggle (nav + palette, OS-default, persisted), replayable automation-pipeline hero, WebGL wireframe mesh with pointer parallax, pointer-tilt on cards, guided project tour (numbered stops, progress bar, arrow keys, focus trap, finishes on the contact section), project filtering by pillar, provider-adapter demo (illustrative, labelled as not repo code), interactive "how I build" diagram, live GitHub explorer with sort, copy-to-clipboard email, print resume stylesheet.

## Deploying

The site sits behind a **Cloudflare Tunnel** on a VPS that already runs nginx and cloudflared for other projects. nginx serves plain HTTP on localhost; the tunnel provides public TLS.

There is no deploy script in this repo. Updates are manual: upload `index.html`, install it into the web root with `www-data` ownership, purge the Cloudflare cache, verify. The exact commands, including the backup and rollback, are in `RUNBOOK.md` section 9. Keep backups outside the web root, or nginx will serve them publicly.

## Content source of truth

Do not contradict these.

- Display name **Ian Kimani**; handle **@WisdomBusara**; email **ianmwaura@gmail.com**; LinkedIn **in/ian-mwaura**. Ian asked (2026-09-27) for the name "Mwaura" to be removed from visible copy: do not add a Kimani/Mwaura explanation, and label the LinkedIn link "Ian Kimani on LinkedIn". The email address and LinkedIn URL still contain it and stay as-is until Ian changes them.
- **Never name the day-job employer or job title** on the site, in commits, or in this file. Ian asked for this to stay private. Describe the day job generically as integration work with core platforms and payment rails.
- Featured public repos: `payments-integration-kit`, `tips-rtgs-client`, `laravel-ecommerce-starter`, `mern-starter-pro`, `rn-payments-starter`, `odk-forms-toolkit`.
- Skills are grouped Strong / Working knowledge / Security foundation / Education. Keep the tiers honest; do not promote a working-knowledge item to strong without cause.

## How Ian wants the work done

- **Produce the artifact, do not propose it.** Lead with the built change, not a plan or a menu of options. Make the call and ship it.
- **Commit and push to GitHub after every change** (`origin main`). This is standing permission for normal pushes. It does not cover force-pushes or history rewrites; ask first for those.
- **Explain root causes for bugs.** When you fix a runtime bug, say what actually caused it, not just what you changed.
- **Push back on exaggeration or fabrication** rather than complying silently. Flag honesty calls out loud.
- **Sweep for consistency.** When you correct something, fix every place it appears across the files, not just the one you were pointed at.
- **Deliver complete files**, not scattered inline snippets.

## Definition of done for any change

- `grep -c $'\xe2\x80\x94' index.html` prints `0`.
- `wisdombusara-site/index.html` is byte-identical to the root `index.html`.
- The page still parses and loads with no console errors (checked in a browser, not with curl).
- Both themes render correctly; the toggle works.
- Reduced-motion still disables animation.
- No new external runtime dependency crept in.
- Keyboard navigation and focus states still work.
- Committed and pushed.

## Open items (not yet done)

- Turn the private-repo category tiles into real project cards once Ian supplies a one-line description per repo and confirms each name may be shown publicly. Category-level only until then.
- GitHub profile hygiene: pin the six brand repos, remove the `byob` and `JS-Shopping-Cart` forks, archive the 2020 tutorial repos, refresh the bio.
- Optional hardening: self-host the two Google fonts to remove the last third-party call.
- A real 1200x630 social share image (`og:image`) for richer link previews.
