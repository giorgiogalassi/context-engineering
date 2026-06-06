# Context Engineering: Stateless by Default, Stateful by Design

A Slidev presentation by **Giorgio Galassi** on the discipline of context engineering for AI coding agents — what agents know, when they know it, and how that knowledge survives session boundaries and tool switches.

## Abstract

LLM-powered coding agents are stateless by default. Every session starts cold. This talk makes the case that the highest-leverage skill when building with AI agents is not prompt engineering — it's context engineering: deliberately deciding what your agents know before the first prompt is ever written.

The talk covers:
- Why sessions fail (session amnesia, cross-project blindness, tool switch loss)
- The four named failure modes from the research community (context confusion, distraction, poisoning, clash)
- Empirical evidence: the U-shaped attention curve and context rot across 18 frontier models
- Four practical techniques: index-first loading, anchored iterative summarization, phase-based context loading, and just-in-time file retrieval
- The decision space: where memory lives, what storage primitive to use, and how to keep memory from becoming a liability

## Stack

- [Slidev](https://sli.dev/) — presentation framework
- Vue 3
- UnoCSS (Slidev default)
- Shiki — code highlighting

## Getting started

```bash
npm install
npm run dev
```

Open [http://localhost:3030](http://localhost:3030) in your browser.

## Commands

| Command           | Description                        |
| ----------------- | ---------------------------------- |
| `npm run dev`     | Start local dev server with HMR    |
| `npm run build`   | Build static site for production   |
| `npm run export`  | Export presentation to PDF         |

## Structure

```
slides.md          — all slide content and speaker notes
src/               — images used in slides
styles/            — custom CSS
```

All slide content lives in `slides.md`. Speaker notes are in `<!-- -->` comment blocks at the bottom of each slide.

## Author

**Giorgio Galassi** — Senior Frontend Engineer (Freelancer) · GDG Roma Città Organizer

## License

ISC
