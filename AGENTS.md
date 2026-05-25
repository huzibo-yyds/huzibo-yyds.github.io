# Codex Project Instructions

This repository is a Jekyll / GitHub Pages personal academic site based on Academic Pages.

## Persistent Context Workflow

The user wants project context to persist across new Codex pages without having to repeat instructions. For any non-trivial work in this repository:

1. At the start, read:
   - `LOCAL_USAGE.md`
   - `docs/codex-handoff.md`
   - `git status --short`
2. If the task touches existing work, inspect the relevant recent changes with `git diff` or `git log` before editing.
3. Before finishing, update `docs/codex-handoff.md` whenever files changed, decisions were made, commands were learned, or follow-up work remains.
4. In the final response, summarize what changed and mention whether the handoff file was updated.

Treat `docs/codex-handoff.md` as the living memory for:

- Current project environment and commands.
- Recently changed files and why they changed.
- Open decisions, TODOs, and risks.
- User preferences that affect future work in this repository.

## Local Environment

- Project path: `/Users/huzi/Documents/Code/huzibo-yyds.github.io`
- Site type: Jekyll / GitHub Pages
- Local preview: `http://localhost:4000/`
- Preferred Ruby/Bundler command prefix: `/opt/homebrew/opt/ruby/bin/bundle`
- Recommended preview command:

```bash
/opt/homebrew/opt/ruby/bin/bundle exec jekyll serve -H localhost
```

- Dependencies should be installed into the project with Bundler local path, not into the system Ruby.
- See `LOCAL_USAGE.md` for full local setup notes.

## Common Content Entry Points

- Site config and profile data: `_config.yml`
- Home page content: `_pages/about.md`
- CV page: `_pages/cv.md`
- Navigation: `_data/navigation.yml`
- Blog posts: `_posts/`
- Publications: `_publications/`
- Talks: `_talks/`
- Teaching: `_teaching/`
- Images: `images/`
- PDFs and public files: `files/`

## Working Style

- Prefer small, focused changes.
- Do not rewrite upstream template files unless the task requires it.
- Preserve user edits in the working tree.
- Use `rg` / `rg --files` for searching.
- For visual or layout changes, run or inspect the local Jekyll site when feasible.
