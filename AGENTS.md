# AGENTS.md (for GitHub Pages repo)

## Project type
- Personal playground / notes / small demos.
- No fixed framework. Keep edits minimal, additive.

## Do / Don't
- DO: Create new demos under `samples/` (one folder per demo).
- DO: Add small utilities under `scripts/` (bash/python ok).
- DON'T: Modify or delete `_site/` (build output) and `docs/` assets.
- DON'T: Force-push or rewrite git history.

## Build/preview (if needed)
- Prefer local preview (`python -m http.server` etc.).
- Avoid adding heavyweight toolchains without approval.

## Style
- Keep pages static-first (HTML/CSS/vanilla JS).
- For experimental libs, pin via CDN and isolate under `samples/<slug>/`.

## Review
- Before committing, run: `npm run lint` (if available) or keep changes < 200 LOC.
