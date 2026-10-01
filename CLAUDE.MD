# Slidev Presentation — Context Engineering Talk

## Project
This is a Slidev presentation project for the talk:
"Context Engineering: Stateless by Default, Stateful by Design" by Giorgio Galassi.

## Stack
- Slidev (latest)
- Vue 3
- UnoCSS / Windi CSS (Slidev default)
- Shiki for code highlighting

## Source
The slide content lives in `slides.md` at the project root.
Do not split slides across multiple files unless explicitly asked.

## Theme
Use the default Slidev theme unless instructed otherwise.
Keep the visual style clean, minimal, and professional.
No decorative elements that are not in the slide content.

## Conventions
- Slide content is in `slides.md`
- Speaker notes are inside `<!-- -->` comment blocks at the bottom of each slide
- Do not modify speaker notes unless explicitly asked
- Do not add slides, remove slides, or reorder slides unless explicitly asked
- Frontmatter is at the top of `slides.md` and controls global settings

## Commands
- `npm run dev` — start local dev server
- `npm run build` — build for production
- `npm run export` — export to PDF

## Tasks Claude Code may be asked to do
- Adjust theme or layout of specific slides
- Add or change layout frontmatter on individual slides (e.g. `layout: center`)
- Add custom CSS in `style.css` or inline via `<style>` blocks
- Add transitions or animations
- Fix rendering issues
- Export the presentation

## Rules
- Never modify slide text content unless explicitly asked
- Never modify speaker notes unless explicitly asked
- Always preserve the `---` slide separators exactly as they are
- If a layout change breaks a slide visually, report it before applying
- Keep all work in the project root unless Slidev convention requires otherwise

## Typography-as-architecture version (under evaluation)
- `slides-type.md` is a full alternative of `slides.md` in the "typography as architecture" style. Same content and speaker notes, plus three number beats (20+, 18/18, 1 line).
- Pieces: `layouts/ruled.vue` (labelled rule from frontmatter `rule: [left, right]`), `components/FitText.vue` (full-width display lines), `styles/type.css` (all rules scoped to `.ta` / `.ta-deck`, so `slides.md` is unaffected), Roboto Flex loaded in `index.html`.
- Colours in `type.css` are tuned to WCAG AA (4.5:1) in light and dark; use its variables (`--ta-accent`, `--ta-ink`, `--ta-muted`), never hex.
- Until one version is chosen, content changes must be made in both files.
- Commands: `npm run dev:type` · `npm run build:type` · `npm run export:type` · `npm run export-dark:type`. Previews in `type-sample/`.
