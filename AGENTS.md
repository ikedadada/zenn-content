# Repository Guidelines

This repository hosts Zenn content (articles and books) maintained with Zenn CLI. Keep changes small, reviewable, and consistent with the existing tooling and conventions below.

## Project Structure & Module Organization
- `articles/`: Standalone articles (`YYYYMMDD_slug.md`) with Zenn frontmatter.
- `books/`: Book chapters/metadata (kept by `.keep` until used).
- Config: `.prettierrc`, `.markdownlint.jsonc`, `.textlintrc`, `prh.yml`, `lefthook.yml`.
- Tooling: `package.json`, `node_modules/`.
- Assets: Prefer a top-level `images/` or per-article relative paths.

## Build, Test, and Development Commands
- `npm ci`: Install dependencies.
- `npm run new:article`: Scaffold a new article with frontmatter.
- `npm run preview`: Launch local Zenn preview server.
- `npm run fmt`: Format Markdown/MDX via Prettier.
- `npm run lint:md`: Run markdownlint checks.
- `npm run lint:text`: Run textlint (JA technical writing + prh rules).
- `npm run lint`: Run both Markdown and text lints.

## Coding Style & Naming Conventions
- Language: Markdown/MDX. Frontmatter required for Zenn articles.
- Prettier: `printWidth: 100`, `proseWrap: always`.
- Markdownlint: `MD013` 100 chars; `MD033`/`MD041` disabled (inline HTML and non-heading first line allowed).
- Textlint: Japanese technical writing preset; keep sentences ≤120 chars, avoid doubled particles; terminology normalized by `prh.yml` (e.g., “GitHub”, “TypeScript”, “Zenn”).
- Filenames: lowercase, hyphenated slug. Example: `articles/20250115_my-topic.md`.
- Headings: Start main content at `##` below frontmatter; use concise, descriptive titles.

## Testing Guidelines
- Lint must pass: `npm run lint` (fix with `npm run fmt` as needed).
- Preview locally (`npm run preview`) and verify links, images, and frontmatter (`title`, `emoji`, `type`, `topics`, `published`).

## Commit & Pull Request Guidelines
- Commits: Conventional Commits (`feat:`, `fix:`, `docs:`, `chore:`). Example: `feat: 記事の初稿を追加`.
- PRs: clear description, scope of changes, related issues, and preview screenshots if layout/visuals change.
- Branches: `feat/<slug>` or `chore/<task>`.

## Tooling & Hooks
- Pre-commit (optional): If using Lefthook, `fmt` and `lint` run on commit (see `lefthook.yml`). Install with `npx lefthook install`.
- Node: Managed via `mise.toml` (`node = "latest"`). Ensure a compatible Node runtime.

