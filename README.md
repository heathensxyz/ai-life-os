# AI Life OS

Reusable architectural patterns for AI-powered personal operating systems. Pulled from a production system, documented so you can steal them.

**Live site:** [heathensxyz.github.io/ai-life-os](https://heathensxyz.github.io/ai-life-os/)

## What this covers

- **Memory architecture:** typed, file-based memory that persists across sessions
- **Hook systems:** safety and validation at the tool level
- **Self-healing automation:** monitoring loops that detect and fix failures
- **Composable skills:** modular capabilities that chain together
- **Privacy-first content pipelines:** autonomous publishing with guardrails

## How the content pipeline works

This site generates its own content through an autonomous pipeline:

1. An AI agent reads the source system (CLAUDE.md files, memory, hooks, skills, automation configs)
2. The agent identifies patterns worth documenting and drafts posts
3. A privacy review filters against `PRIVACY_MANIFEST.md` to strip personal details
4. Approved posts are committed and deployed via GitHub Actions

## Local development

Requires Node.js 22+.

```sh
npm install
npm run dev        # dev server at localhost:4321
npm run build      # production build to ./dist/
npm run preview    # preview the production build
```

## Stack

- [Astro](https://astro.build) with MDX and sitemap
- GitHub Pages via GitHub Actions
- Content pipeline powered by Claude Code

## License

Content and code in this repository are provided as-is for learning and adaptation.
