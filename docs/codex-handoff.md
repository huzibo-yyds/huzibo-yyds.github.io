# Codex Handoff

This file is the persistent project memory for future Codex pages. Read it at the start of non-trivial work and update it before finishing when anything meaningful changes.

## Current Project Snapshot

- Repository: `/Users/huzi/Documents/Code/huzibo-yyds.github.io`
- Project type: Jekyll / GitHub Pages academic personal site.
- Local preview URL: `http://localhost:4000/`
- Main local instructions: `LOCAL_USAGE.md`
- Project-level Codex instructions: `AGENTS.md`

## Environment Notes

- Use Homebrew Ruby instead of system Ruby:

```bash
/opt/homebrew/opt/ruby/bin/bundle
```

- Recommended local preview command from the repository root:

```bash
/opt/homebrew/opt/ruby/bin/bundle exec jekyll serve -H localhost
```

- Bundler should install dependencies into the project-local `vendor/bundle` path when setup is needed.
- `LOCAL_USAGE.md` says the current shell has `node` but may not have `npm`; check before using npm scripts.
- Docker is not currently available from the shell according to `LOCAL_USAGE.md`.
- GitHub push failures with `Permission denied (publickey)` here were traced to SSH auth, not repo permissions: the local `id_ed25519` key exists and is offered, but GitHub rejects it unless the matching public key is added to the correct GitHub account.
- The current SSH config routes `github.com` through `ssh.github.com:443` with a local proxy command; that path reaches GitHub successfully, so the failure happens after connection establishment.

## User Preferences

- Keep project assumptions, environment details, and change history in repository files so new Codex pages can recover context.
- Do not require the user to explicitly remind Codex to read or update context after every conversation.
- Prefer concise Chinese explanations when discussing workflow with the user.

## Recent Change Log

### 2026-05-28: Added ResNet18 Relay IR structure blog

Changed files:

- `_posts/2026-05-28-resnet18-relay-ir-structure.md`: New blog post converted from the user's Obsidian note about reading ResNet18 from TVM Relay IR, with front matter, excerpt, explicit heading anchors, Mermaid diagrams, and a `toc_items` outline for the custom post TOC.
- `_pages/about.md`: Updates the "阅读最新文章" button and recent timeline to link to the new ResNet18 Relay IR article.
- `docs/codex-handoff.md`: Records this publication work.

Purpose:

- Publish a polished article titled "ResNet18 结构详解：如何阅读 TVM Relay IR" at `/posts/2026/05/resnet18-relay-ir-structure/`.
- Keep the blog's right-side outline behavior consistent by ensuring each TOC item maps to an explicit `##` or `###` heading id.

Validation:

- Ran `git diff --check` successfully.
- Parsed Markdown front matter for posts/pages successfully.
- Verified the new post has 27 TOC entries and 27 matching heading anchors with no missing or extra ids.
- Ran `/opt/homebrew/opt/ruby/bin/bundle exec jekyll build` successfully.
- Confirmed generated HTML includes the new post, homepage link, and TOC entries.

### 2026-05-25: Fixed blog timestamp header layout

Changed files:

- `_layouts/single.html`: Replaces the post timestamp wrapper's theme `page__date` classes with a dedicated `post-date-meta` component, and uses the post date as a visible fallback for update time while GitHub commit data loads.
- `_sass/layout/_page.scss`: Adds standalone timestamp layout rules with full-width placement, cleared floats, no internal label/time wrapping, and a mobile stacked layout.
- `docs/codex-handoff.md`: Records this fix.

Purpose:

- Fix deployed blog pages where "发布时间" and "更新时间" could be squeezed into a narrow right-side column and wrap vertically.
- Avoid showing "自动获取中" as a persistent or slow-network visible state; the GitHub commits API still replaces the fallback when available.

Validation:

- Ran `/opt/homebrew/opt/ruby/bin/bundle exec jekyll build` successfully.
- Verified generated `_site` HTML uses `.post-date-meta` and generated CSS contains the new rules.
- Confirmed the GitHub commits API returns commit data for `_posts/2026-05-18-tvm-relay-resnet18-beginner-guide.md`.
- Note: local Jekyll pages currently load CSS from the configured production `url`, so browser preview may show the old deployed CSS until the new CSS is pushed.

### 2026-05-21: Added blog updated timestamps

Changed files:

- `_layouts/single.html`: Blog posts now render both creation time and update time in the header. A client-side script queries GitHub commits for the current post source path, using the first commit as creation time and the latest commit as update time.
- `_data/ui-text.yml`: Adds localized `updated_label` text for English, simplified Chinese, and traditional Chinese.
- `_includes/seo.html`: Keeps static `article:modified_time` limited to `modified`; the visible post time is filled from GitHub commit data at runtime.
- `_sass/layout/_page.scss`: Adds compact responsive styling for post publish/update metadata.
- `_posts/2026-05-18-tvm-relay-resnet18-beginner-guide.md`: Removes manual `last_modified_at`; no timestamp field is required.
- `_posts/2026-05-15-tvm-custom-operator-layernorm.md`: Removes manual `last_modified_at`; no timestamp field is required.
- `LOCAL_USAGE.md`: Documents that blog creation/update times are automatically read from GitHub commit history.

Purpose:

- Show visible "发布时间" and "更新时间" for every blog post, formatted down to hour/minute and no seconds.
- Avoid requiring manual timestamp fields in Markdown front matter.
- Keep the implementation GitHub Pages compatible without adding a custom Jekyll plugin.

Operational note:

- Future posts do not need `last_modified_at`. The visible timestamps are based on remote GitHub commit history for each post file, not local file save time.

### 2026-05-21: Restyled related posts on blog pages

Changed files:

- `_layouts/single.html`: Replaces the old `site.related_posts` grid rendering with a related-posts section that first selects posts sharing tags with the current post, then falls back to Jekyll's default related posts if no tag overlap exists.
- `_includes/related-post-card.html`: New dedicated related-post card include with date, read time, title, excerpt, and tags.
- `_sass/layout/_page.scss`: Restyles the related-posts area so it aligns with the post content column and uses full-width responsive cards instead of the narrow archive grid item.
- `docs/codex-handoff.md`: Records this change for future Codex sessions.

Purpose:

- Fix the visually awkward "你可能感兴趣的" block at the bottom of blog posts.
- Make the recommendation behavior easier to explain and more intuitive by preferring shared tags.

Validation:

- Ran `/opt/homebrew/opt/ruby/bin/bundle exec jekyll build` successfully.
- Verified the generated ResNet18 post uses `.related-post` markup, no longer uses the old `.grid__wrapper`, and recommends the LayerNorm post via shared tags.
- Checked local `http://localhost:4000/posts/2026/05/tvm-relay-resnet18-beginner-guide/`; the related area width is aligned to the content column.

### 2026-05-20: Split tag index from tag detail pages

Changed files:

- `_layouts/archive.html`: Adds an optional `archive_class` wrapper class so archive-specific pages can style their titles without affecting other pages.
- `_includes/taxonomy-cloud.html`: New tag cloud include for `/tags/`, with tag size classes based on related post counts.
- `_includes/taxonomy-card-grid.html`: New card-grid include for category-style archive previews.
- `_includes/taxonomy-archive.html`: Removed the previous combined tag index and grouped-post list include.
- `_includes/tag-list.html`: Updates post-level tag links to open `/tag/?tag=<slug>` instead of the old `/tags/#<slug>` anchors.
- `_pages/tag-archive.html`: Changes `/tags/` into a centered "all tags" cloud that links each tag to `/tag/?tag=<slug>`.
- `_pages/tag-detail.html`: New single-tag detail page that reads the `tag` query parameter and shows only that tag's posts in a year timeline.
- `_pages/category-archive.html`: Changes `/categories/` into a card-grid archive preview, currently backed by post tags because existing posts do not define `categories`.
- `_sass/layout/_archive.scss`: Adds styles for centered archive headings, weighted tag cloud, category cards, and tag-detail timeline.

Purpose:

- Avoid the messy stacked `/tags/` page by separating index and detail views.
- Match the reference site's "all tags" cloud and "tag detail timeline" interaction while staying compatible with static GitHub Pages.

Validation:

- Ran `/opt/homebrew/opt/ruby/bin/bundle exec jekyll build` successfully.
- Started local Jekyll preview at `http://localhost:4000/` with approval because sandboxed port binding failed.
- Verified `/tags/` renders only weighted tags, `/tag/?tag=compiler` shows only the Compiler posts, and `/categories/` renders card previews.

### 2026-05-19: Added tag/category archive pages

Changed files:

- `_includes/taxonomy-archive.html`: New reusable Liquid include that groups posts by a field, renders a top index of all taxonomy values with post counts, and lists matching posts under each group.
- `_pages/tag-archive.html`: Reworked `/tags/` into a Chinese tag archive showing all tags, counts, and grouped blog posts.
- `_pages/category-archive.html`: Reworked `/categories/` to share the same archive include and show an empty state until posts have categories.
- `_data/navigation.yml`: Added a top navigation link to `/tags/`.
- `_sass/layout/_archive.scss`: Added styling for taxonomy chips, counts, grouped sections, and compact post rows.

Purpose:

- Match the reference pattern from `albresky.cn/categories/` and `albresky.cn/tags/`: tags/categories are collected at the top with counts, then posts are grouped below by taxonomy.
- Keep implementation GitHub Pages compatible by using Liquid instead of `jekyll-archives`.

Validation:

- Ran `/opt/homebrew/opt/ruby/bin/bundle exec jekyll build` successfully.
- Started local Jekyll preview at `http://localhost:4000/` with approval because sandboxed port binding failed.
- Verified `/tags/` shows 7 tags and 2 blog posts, and `/categories/` shows the empty-state message because current posts do not define `categories`.

### 2026-05-19: Added persistent Codex context workflow

Changed files:

- `AGENTS.md`: Adds project-level instructions requiring Codex to read local usage notes, read this handoff, check git status, and update the handoff after meaningful work.
- `docs/codex-handoff.md`: Adds the living project memory file for environment notes, user preferences, recent changes, and follow-ups.

Purpose:

- Preserve project settings and work history across new Codex pages.
- Reduce repeated manual prompting from the user.

## Open Follow-Ups

- For future feature/content edits, append a short dated entry under "Recent Change Log" with changed files and purpose.
- If the local environment changes, update both `LOCAL_USAGE.md` and the environment notes above.
