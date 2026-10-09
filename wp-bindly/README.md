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
| Print version of a template | `(template:foo, has-variant, template:bar)` qualified by `mode: print` |
| Photo illustrating a post (without the single-parent limit of `post_parent`) | `(post:12, illustrated-by, attachment:88)` |
| Video of a post | `(post:12, has-video, ext:youtube:abc123)` |
| Trip stages | `(post:12, next, post:13)` or `(post:12, part-of, post:2)` |
| Content of a book | `(post:book-2026, contains, post:12)` with a position |

### Reification: statements with identity and qualifiers

**Decided.** Eric's example: a photo attached to a post, where the attachment must say whether it concerns the web presentation, the print presentation, or both. That is an n-ary relation, so a plain triple is not enough. Options considered: pure triples reified on demand (a fact becomes several triples), one predicate per combination (unmanageable), quads with a context column (covers the mode but not position or caption), and **statements with identity and qualifiers**, which was chosen.

- Every statement has its own **identity**. `rel:123` is a valid typed identifier, usable as subject or object of another statement, so true reification (a statement about a statement) stays possible without any schema change.
- A statement can carry **qualifiers**: typed key/value pairs declared by the predicate in the registry (name, type, single or multiple values). The core validates them.
- The core knows nothing about modes. Domain 2 registers a qualifier type "mode" (a set of mode slugs). This keeps the one-way dependency. This replaces the earlier idea of making a mode a predicate (`mode/print`), which was dropped: it multiplies predicates and cannot carry other qualifiers.
- Reading rule (**proposed**): no `mode` qualifier means "all modes"; the most specific statement wins, as for templates.

```
(post:12, illustrated-by, attachment:88) {modes: [web, print], position: 2}
(post:12, illustrated-by, attachment:91) {modes: [print]}
(template:foo, has-variant, template:bar) {mode: print}
```

**Decided: two tables at the start.**

- Statements: logically `id, subject, predicate, object`; physically `id`, `subject_type`, `subject_id`, `predicate`, `object_type`, `object_id`, and probably a position.
- Qualifiers: `(statement_id, key, value)`, indexed on `(key, value)`. A JSON column is avoided because WordPress does not guarantee the JSON type on every database it supports, and filtering by mode must be efficient.
- Reasons for two tables over one: qualifiers can be type-checked through the registry, queries by mode are simpler to index, and deleting a statement removes its qualifiers without a special rule.
- The two forms are **logically equivalent**: a qualifier is a triple whose subject is the statement, `(rel:123, mode, "print")`. A single table (literals allowed as objects) is therefore possible later without changing the model.

Open sub-questions:

- May several statements share the same *(subject, predicate, object)* with different qualifiers (needed if the position of a photo differs between web and print)? Proposed: a flag per predicate.
- Is an exclusion form ("all modes except print") needed? Proposed: not at the start; one statement per mode is enough.
- Not planned: named graphs, inference.

Risk: a generic relation engine can absorb all the development time. **Proposed** mitigation: deliver first only the two uses that serve the books (template modes and the ordered content of a book) behind a clean API; add media and post-to-post relations as real needs appear. The "Posts 2 Posts" plugin did something similar for posts and is no longer maintained, which shows both the need and the effort.

## 2. Multiple templates

**Decided**: replace the naming-convention hack with explicit relations between templates, and generalize "print" into a **mode**, with several modes definable.

- A **mode** is a declared entity: slug (`print`, `book`, `cover`...), label, query-string trigger.
- A **relation** *(source template, mode) → target template* says "template `bar` is the `print` version of template `foo`". One target can serve several sources. A default target per mode is possible: *(\*, print) → print-default*.
- **Decided**: a mode is *not* a predicate. The mode of a relation is a qualifier of the statement (see "Reification" above): "`bar` is the `print` version of `foo`" is `(template:foo, has-variant, template:bar)` qualified by `mode: print`.
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
- **Proposed**: `book` and `cover` are modes of domain 2, each with its own target templates. Each post is rendered in the `print`/`book` mode. A photo can then be attached to a post for the web only, for print only, or for both, through the `mode` qualifier.
- **PDF production, proposed order**:
  1. A single "book" page: all posts concatenated with a table of contents, CSS `@page` rules and page breaks, turned into a PDF by the browser or a paged-media tool. This needs no external binary and keeps the "no `shell_exec`" rule used in `wp-i18nly`.
  2. Later, optionally, server-side rendering (headless Chrome or a service) to produce PDFs automatically.
- Final assembly (cover, continuous pagination, binding margins) could be done in PHP with a library or stay in an external tool.

Not yet known: how PDFs are produced and assembled today, page sizes and printers targeted (A4, square, binding).

## Packaging

**Decided: one plugin with three modules** (`Triples`, `Modes`, `Books`), built so that it can later be split into three plugins at little cost. Extract the core into its own plugin once its API has been stable for a while and a second consumer exists.

Reasons:

- The triples model is still moving (see the reification discussion); three plugins would freeze an unstable cross-plugin API from the first day.
- The core tables would be owned by one plugin while holding the data of the others; uninstalling it would destroy or orphan that data. WordPress 6.5+ `Requires Plugins` declares dependencies but does not constrain versions.
- Costs triple with three plugins: translations, `uninstall.php`, schema migrations, admin screens, CI and releases.
- Against: the boundaries are enforced by discipline and tests rather than by packaging, which is the reason for the rules below. Eric's earlier plugins (`wp-scatter-elsewhere`, `wp-media-helper`) are separate repositories.

Namespaces are necessary but not sufficient. The eight rules that keep the split cheap:

1. **One-way dependencies, checked by an automated test** (`deptrac` or equivalent): `Triples` knows nothing, `Modes` sees only `Triples`, `Books` sees only the other two.
2. **Talk through a public API, not concrete classes.** `Modes` never writes SQL in the `Triples` tables; it goes through interfaces and hooks (for example registering the "mode" qualifier type).
3. **Each module owns its data:** creation and migration of its tables, its schema version in its own option, its own cleanup in `uninstall`.
4. **Names belong to the module, not to the umbrella:** table names (`triples_statements`, not `bindly_statements`), options, hooks, REST namespace (`triples/v1`), capabilities and text domain. This is the costliest to fix afterwards, because renaming stored data and settings needs a migration.
5. **A directory layout that lets a module be lifted out:** `plugin/modules/triples/`, `modules/modes/`, `modules/books/`, each with its own sources, tests, translation files and admin scripts.
6. **Tests per module** that run without loading the other modules (except the ones it depends on). This is the real proof that the separation exists.
7. **A module loader:** each module has its own bootstrap, and the plugin loads the list of enabled modules.
8. **No catch-all "common" directory.** If two modules need the same utility, either it belongs to the core or it is duplicated.

Work left at split time: plugin headers and `Requires Plugins`; a runtime check of the `Triples` API version (WordPress does not enforce plugin versions); taking over existing data and the activation order; CI, releases and translations in three copies; regrouping the admin menus. For Eric's own blog this is almost immediate; if other people install the plugin first, taking over their data needs more care.

## The current hack: `wp-pdf-helper`

Found on 2026-10-09: the hack is a small WordPress plugin, "WP PDF Helper", in the public repository <https://gitea.dyomedea.com/vdv/wp-pdf-helper> (one commit, May 2025, titled "Regression"; empty README; one PHP file of 124 lines and a few assets). The Gitea host was reachable from the session without any network change because the repository is public; private repositories there would still need a read-only token.

What it does:

- **Mode trigger.** At load time, `if ( ! array_key_exists( 'print', $_GET ) ) return;`. Everything below runs only when the query string has a `print` key (any value, even empty). The mode is decided once, at plugin load, and is not propagated to links.
- **Template swap, block themes only.** A `get_block_templates` filter takes the id of the first `wp_template` in the result (`theme//single`), appends `-print`, loads it with `get_block_template()` and, if it exists, replaces the first element. Classic PHP themes are not supported. No check that the result is non-empty, and the filter applies to every call of `get_block_templates`, not only the front-end template resolution.
- **Print stylesheet.** Enqueues `wp-pdf-helper-print.css`: hides header, navigation, footer, videos, comments, query loops and map controls with `display: none !important`, adjusts margins and font sizes, adds `page-break-*` rules. It depends on the theme's class names and on other plugins' blocks (`wp-block-wpprg-wp-printable-gallery`, Leaflet, WP GPX Maps). It has no `@page` rule, so page size and margins are not defined here.
- **A second query parameter.** `?gpxmap-size=small|large` sets the map and chart heights through the `wpagpx_shortcode_parameters` filter of WP GPX Maps, and turns off attachments and downloads in print mode.
- **A "Print" checkbox in the media library.** A column added to the media list saves the post meta `wpdfh.print = 'always'` through AJAX. Nothing in this plugin reads that meta; whatever uses it (a theme template or the printable gallery block) lives elsewhere. It is a per-attachment flag, global to all posts, which is the primitive version of a statement qualified by mode.
- **No PDF production.** Nothing here creates or assembles PDFs; the description says "helps to print". PDFs presumably come from the browser's print function or an external tool. The `.print-link` style exists but no code in the repository generates the link, so it is probably in the theme.

Problems visible in the code (useful as a checklist for the replacement):

- The AJAX handler `wpdfh_set_print_metadata` has **no nonce and no capability check** and does not validate `post_id`: any logged-in user, whatever the role, can set or delete the meta on any post.
- Debug `error_log` calls remain active, including a `print_r` of the template object on every filtered call.
- Global functions with generic names (`add_media_column`, `manage_custom_columns`, `ww_load_styles`): collision risk. Hard-coded English labels, no text domain, no `uninstall`, no version on enqueued assets, empty README, an editor configuration file committed.
- **Naming collisions in the template hierarchy.** Appending `-print` to a template slug produces names WordPress also uses itself: `page-print` is the template of a page whose slug is `print`, `category-print` and `tag-print` those of a term with that slug, `single-print` the template of a post type named `print`. Explicit relations between templates avoid this.

Implications for the design:

- Block themes (and block templates stored in the database or in theme files) must be supported; the existing hook point works only through `get_block_templates`, and whether better points exist is still to verify in WordPress core.
- A mode may need **parameters or options** (here the map size) and **hooks for third-party plugins** (WP GPX Maps) that adapt their output per mode. To design: a way for integrations to ask "which mode is active, with which options?".
- The per-attachment "Print" flag becomes a mode-qualified attachment per post (see the Media Helper section).
- Per-mode assets (stylesheets) are needed, as already listed; some of what the CSS hides could be removed from the print templates instead.
- The `-print` templates are `wp_template` posts created in the site editor (database), not theme files. Where the print links come from is still to find out.

Answers from Eric (2026-10-09):

- **The `-print` templates are created in the site editor**, so they are `wp_template` posts stored in the database, not theme files. Their ids still have the form `theme//slug`. They are tied to the active theme and are not versioned in git (export from the site editor is the only copy outside the database). The admin screen for relations should list templates from both sources (files and database).
- **The "Regression" commit** concerns the print CSS the plugin injects: a rule was removed. Which rule is not known, and the repository has a single commit, so there is no history to compare. It shows how fragile a global stylesheet full of `!important` overrides is.
- **PDFs are produced and assembled by hand today:** each page is printed to PDF with the Samsung Internet browser on Android (the only browser Eric found that does not add a header and footer), then the PDFs are arranged and merged with PDF Arranger on Ubuntu. Automation would save a lot of time.

### PDF production: automation options

**Proposed**, nothing decided.

- **Constraint.** A PHP PDF library (Dompdf, mPDF, TCPDF) is a poor fit: block themes rely on modern CSS (flex, grid, CSS variables), and the pages contain JavaScript-rendered maps (Leaflet, WP GPX Maps) and lazily loaded images. A real browser engine is needed. The plugin itself stays PHP-only (no `shell_exec`), so rendering happens outside WordPress.
- **What the plugin provides.** The `print`/`book` modes and their templates; a **single book page** (all posts of a book in order, table of contents, `@page` size and margins, continuous page numbers); a **manifest** of the book (ordered list of URLs with mode and options) through the REST API and WP-CLI.
- **What an external renderer does.** Opens the book page, or each URL of the manifest, in headless Chromium (for example Playwright, `page.pdf()` with header and footer disabled and the CSS page size preferred), waits for maps and images to finish loading, and merges the PDFs if there are several. With one book page, page numbering, table of contents and running headers become continuous, which per-post PDFs merged afterwards cannot give.
- **Where the renderer could run** (to decide): a small command-line tool on Eric's Ubuntu laptop; a CI job; the web host if it can run Chromium (unlikely on shared hosting); or in the browser with a paged-media library such as Paged.js (to verify with maps and the volume of a full book, and on a phone).
- **Open for Books:** where the renderer lives (this repository, a separate tool), and how far "Books" goes beyond composing the book and exposing the manifest.

## Integration with Media Helper

**Raised by Eric; to be studied, nothing decided.** [`wp-media-helper`](https://github.com/evlist/wp-media-helper) currently handles only native WordPress attachments. Statements will bring things it cannot express today:

- attaching a photo to **several posts** (a native attachment has a single `post_parent`);
- attachments **qualified by mode** (a photo for the web only, for print only, or for both), with a position that may differ per mode;
- **other kinds of attachments** than native ones (for example external resources such as `ext:youtube:ID`).

Media Helper must be able to manage these. Eric's idea: an extension mechanism (hooks) in Media Helper.

Directions to examine (**proposed**, not yet checked against Media Helper's code, which was not read for this note):

- **Direction of dependency.** Media Helper should stay usable alone, so it must not depend on this project. It exposes extension points; this project plugs an adapter into them. The reverse (Media Helper using the `Triples` API when present) is the alternative to weigh.
- **Extension points needed.** Probably: a filter resolving "the media attached to a post, for a given mode"; a filter or action for attaching and detaching, so that statements replace `post_parent`; a hook to display and edit the extra attachment data (modes, position) in Media Helper's screens; a hook for the type of an attached item (native attachment or other).
- **Galleries.** The gallery shortcode currently builds its HTML from the attachments of a post; it would have to ask for the attachments of the current mode.
- **Image sizes per mode.** Print needs different (larger) sizes than the web. Media Helper's configurable thumbnail sizes (its slice 033, not implemented) could depend on the mode.
- **Compatibility with core.** What `post_parent` and the "Attached to" column of the media library show when a photo belongs to several posts; keep `post_parent` as a primary parent, or leave it empty.
- **Lifecycle.** Deleting a media item or a post must remove or flag the matching statements (a missing-file report already exists in Media Helper's planned slice 036).

First step: read Media Helper's attachment model and the hooks it already offers, then decide whether the change belongs in Media Helper (new hooks), in this project (adapter), or both.

## Open questions

1. ~~Packaging~~: decided, see "Packaging" above. Still open within it: the plugin's final name.
2. **Names**: candidates `wp-triples` or `wp-relations`, `wp-template-modes`, `wp-bindly` or `wp-books`. "Template Modes" no longer describes the whole.
3. **PDF automation**: the current manual chain and the options are described above. To decide: where the renderer runs (laptop, CI, host, browser), whether it lives in this project, and where the print links come from.
4. **Page formats and printers** for the books.
5. **Domain 1 scope**: what is built first, and whether the triples component is extracted as a standalone plugin later.
6. **Gitea**: Eric's private repositories on his Gitea server may contain the hack. Reaching them requires allowing the server's domain in the environment's network settings and a read-only token stored as a secret, not pasted in chat.

## Reusable assets

From existing projects (see the repository `README.md` for the list):

- `wp-i18nly`: repository layout (`plugin/`, `tests/`, `docs/`, `scripts/`), PSR-4 code, PHPUnit and `phpcs`, REUSE/SPDX, CI, slice-by-slice development.
- `wp-media-helper` and `wp-scatter-elsewhere`: slice documents (`docs/slices/NNN-name.md`), admin Tools pages for long jobs run by WP-Cron with stop and resume, REST controllers and WP-CLI commands sharing one service layer.

## Suggested first steps (not started)

1. Write the slice plan for domain 1 (predicate registry, storage, API) in the project repository, following the `wp-media-helper` format.
2. ~~Study the real `?print` hack~~ (done, see "The current hack"); study the PDF chain once described.
3. Check in WordPress core how to hook template selection for both block and classic themes.
4. Study Media Helper's attachment model and extension points (see "Integration with Media Helper").
