# wp-bindly

Code name for the project that replaces the "`?print` hack" described in the [Balisage 2026 presentation](https://evlist.github.io/balisage-2026/) with a cleaner mechanism, and helps assemble PDFs into books. "wp-bindly" is the umbrella name with the widest scope; the final names of the plugins (if there are several) are not chosen yet.

Status: **brainstorming**. No repository and no code yet. Written on 2026-10-09 from the discussion in a Claude Code session; update it as points are settled.

How to read this document: each point is marked **Decided** (Eric accepted it), **Proposed** (suggested by Claude, not confirmed) or **Open**.

## Goal

One body of content (a WordPress travel blog) published in several forms for several audiences: web, print, monthly photo books, a printed blog with videos. WordPress serves HTML and CSS media queries cannot change the *structure* of the content, hence the use of different templates per form. Today the choice is a hack selecting a `-print` template when the query string contains `?print`. The goal is to make this a proper, declared mechanism, and to help build books from the resulting PDFs.

## Three functional domains

**Decided** (Eric's framing): the project has three distinct domains, with one-way dependencies.

| # | Domain | Role | Depends on |
|---|---|---|---|
| 1 | Triples | Registry of predicates, storage and API for *(subject, predicate, object)* relations. Knows nothing about templates or books. | nothing |
| 2 | Multiple templates | Modes and relations between templates, selected through the query string. | 1 |
| 3 | Books | Compose a book from posts, assemble and export PDFs. | 1, 2 |

## 1. Triples

Eric's idea: the notion of a triple goes beyond templates and can describe other relations, for example a more general link than attachments between posts and media, or between posts themselves.

**Proposed** model:

- A **triple** is *(subject, predicate, object)* with an optional position (for ordering) and optional metadata.
- Subjects and objects are **typed identifiers**: `post:123`, `term:45`, `user:7`, `template:theme//slug`, `attachment:88`, `ext:youtube:ID` (an external resource such as a video).
- A **predicate registry**, declared like a post type: slug, label and inverse label, allowed subject and object types, cardinality (one or many), ordered or not, symmetric or not, behavior when a subject or object is deleted.
- **Storage** in a dedicated table rather than post meta, with indexes on *(subject, predicate)* and *(object, predicate)* so that lookups work in both directions, plus object caching.
- Later: REST API, `WP_Query` integration, an editor panel in the block editor, JSON export (JSON-LD is an option).

Examples of what it covers:

| Need | Triple |
|---|---|
| Print version of a template | `(template:foo, mode/print, template:bar)` |
| Photo illustrating a post (without the single-parent limit of `post_parent`) | `(post:12, illustrated-by, attachment:88)` |
| Video of a post | `(post:12, has-video, ext:youtube:abc123)` |
| Trip stages | `(post:12, next, post:13)` or `(post:12, part-of, post:2)` |
| Content of a book | `(post:book-2026, contains, post:12)` with a position |

Risk: a generic relation engine can absorb all the development time. **Proposed** mitigation: deliver first only the two uses that serve the books (template modes and the ordered content of a book) behind a clean API; add media and post-to-post relations as real needs appear. The "Posts 2 Posts" plugin did something similar for posts and is no longer maintained, which shows both the need and the effort.

## 2. Multiple templates

**Decided**: replace the naming-convention hack with explicit relations between templates, and generalize "print" into a **mode**, with several modes definable.

- A **mode** is a declared entity: slug (`print`, `book`, `cover`...), label, query-string trigger.
- A **relation** *(source template, mode) → target template* says "template `bar` is the `print` version of template `foo`". One target can serve several sources. A default target per mode is possible: *(\*, print) → print-default*.
- **Proposed**: a mode is a registered predicate (`mode/print`), so the model stays a pure triple.
- **Resolution**: WordPress picks the template as usual (full hierarchy); if a mode is active, the plugin looks for a relation for that template, then a default target for the mode, then applies the fallback.

Decided details:

- **Fallback** when no relation applies: the normal template. Mode inheritance (for example `book` inherits from `print`) is optional.
- Relations also apply to **template parts** (for example `header` and `header-print`), not only to whole templates.
- **Trigger**: `?mode=print`, with an optional `?print` alias so existing links keep working.
- **Storage and editing**: an admin screen, with JSON export and import so the relations can be versioned.

Proposed details:

- Optional side effects per mode: `noindex`, `rel=canonical` to the normal version, a CSS class on `<body>`, mode-specific styles and scripts.
- Internal links and pagination keep the active mode; a "printable version" link is exposed.
- No chaining of relations by default; detect cycles when saving.
- Security: allow-list of modes, the query-string value never designates a file path, only administrators edit relations.
- Page caches must vary on the query string.
- Set the filter priority explicitly and document coexistence with themes and plugins hooking the same filter (see the Thumbnails Folder lesson in `CLAUDE.md`).

To verify in the WordPress source before building: block themes (HTML files, or `wp_template` posts edited in the site editor) and classic PHP templates are selected by different mechanisms, so the point where the filter hooks in must cover both.

## 3. Books

Only sketched so far; nothing decided.

- **Proposed**: a book is a selection of posts (by date, category, tag or trip) with an order and sections (month, stage), stored as an ordered `contains` relation.
- **Proposed**: `book` and `cover` are modes of domain 2, each with its own target templates. Each post is rendered in the `print`/`book` mode.
- **PDF production, proposed order**:
  1. A single "book" page: all posts concatenated with a table of contents, CSS `@page` rules and page breaks, turned into a PDF by the browser or a paged-media tool. This needs no external binary and keeps the "no `shell_exec`" rule used in `wp-i18nly`.
  2. Later, optionally, server-side rendering (headless Chrome or a service) to produce PDFs automatically.
- Final assembly (cover, continuous pagination, binding margins) could be done in PHP with a library or stay in an external tool.

Not yet known: how PDFs are produced and assembled today, page sizes and printers targeted (A4, square, binding).

## Open questions

1. **Packaging**: one repository per plugin, or one repository with three plugins? WordPress 6.5+ lets a plugin declare dependencies with the `Requires Plugins` header. Claude's leaning: one repository with three plugins while the core API is not stable, each distributable separately; Eric's past plugins (`wp-scatter-elsewhere`, `wp-media-helper`) are separate repositories.
2. **Names**: candidates `wp-triples` or `wp-relations`, `wp-template-modes`, `wp-bindly` or `wp-books`. "Template Modes" no longer describes the whole.
3. **Current hack and PDF chain**: Eric will provide the PHP of the `?print` hack and describe how PDFs are made and assembled. The presentation does not contain them.
4. **Page formats and printers** for the books.
5. **Domain 1 scope**: what is built first, and whether the triples component is extracted as a standalone plugin later.
6. **Gitea**: Eric's private repositories on his Gitea server may contain the hack. Reaching them requires allowing the server's domain in the environment's network settings and a read-only token stored as a secret, not pasted in chat.

## Reusable assets

From existing projects (see the repository `README.md` for the list):

- `wp-i18nly`: repository layout (`plugin/`, `tests/`, `docs/`, `scripts/`), PSR-4 code, PHPUnit and `phpcs`, REUSE/SPDX, CI, slice-by-slice development.
- `wp-media-helper` and `wp-scatter-elsewhere`: slice documents (`docs/slices/NNN-name.md`), admin Tools pages for long jobs run by WP-Cron with stop and resume, REST controllers and WP-CLI commands sharing one service layer.

## Suggested first steps (not started)

1. Write the slice plan for domain 1 (predicate registry, storage, API) in the project repository, following the `wp-media-helper` format.
2. Study the real `?print` hack and PDF chain once provided.
3. Check in WordPress core how to hook template selection for both block and classic themes.
