# Repository Guidelines

## Project Structure & Module Organization
- `articles/`: Main content. One article per file (`{slug}.md`).
- `images/`: Image content.
- `mise.toml`: Tasks and pinned tools (Node 24.6.0, Zenn tasks).
- `package.json`: Zenn CLI dependency and scripts.

## Build, Test, and Development Commands
- Create article: `mise run create-article` (or `npx zenn new:article`).
- Preview locally: `mise run preview` → opens http://localhost:8000.
- Publish (Git push with slugged message): `mise run publish`.
- Node version: `mise install` ensures the pinned Node in `mise.toml`.

## Coding Style & Naming Conventions
- Files: `articles/{kebab-case-slug}.md` (ASCII, hyphen-separated).
- Frontmatter (required): `title`, `emoji`, `type`, `topics`, `published`.
- Markdown: use `#` for title; tag code fences with language (e.g., `kotlin`).
- Keep lines readable; prefer concise headings and meaningful code blocks.

## Testing Guidelines
- No unit tests. Validate via preview: `mise run preview`.
- Check frontmatter renders correctly, links work, and code blocks highlight.
- For drafts, set `published: false` until ready.

