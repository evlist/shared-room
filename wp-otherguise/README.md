# wp-otherguise

The project that replaces the "`?print` hack" described in the [Balisage 2026 presentation](https://evlist.github.io/balisage-2026/) with a cleaner mechanism, and helps assemble PDFs into books. The name is a blend of *otherwise* and *guise*: the same content, in another guise. It was chosen on 2026-10-09; the short-lived code name was `wp-bindly`. `wp-otherguise` is the name of the single plugin (see "Packaging"); its modules keep their technical names `Triples`, `Modes` and `Books`.

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

### Reification: statements about statements

**Decided**, revised on 2026-10-09 by Eric (this replaces an earlier decision: qualifiers in a second table, with a fingerprint, a flag for repeated statements and a position column).

Eric's example: a photo attached to a post, where the attachment must say whether it concerns the web presentation, the print presentation, or both, and at which rank. That is an n-ary relation, so a plain triple is not enough. Options considered: one predicate per combination (unmanageable), quads with a context column (covers the mode but not the position), qualifiers stored apart from the triple, and **statements about statements**, which was chosen.

- Every statement has its own identity (its `id`), and `statement:ID` is a valid subject or object. **A qualification is just another statement** whose subject is the statement qualified. The unit is the triple; a statement stays the same whatever statements are later added about it.

```
statement:41: (post:12, illustrated-by, attachment:88)
statement:42: (statement:41, mode, mode:web)
statement:43: (statement:41, mode, mode:print)
statement:44: (statement:41, position, 5)
statement:45: (statement:43, position, 1)
```

- Here the photo is in the web and print versions; its rank is 5 for every mode (44), except in print where it is 1 (45): a statement about the print qualification. No splitting, merging or renumbering is ever needed to give a mode its own rank.
- **Reading rule** (proposed): no `mode` statement means "all modes". A position on a mode statement applies to that mode only and overrides the position on the statement itself; without any position the natural order (for photos, date and time computed by the consumer) applies. The order is computed on all the statements of the subject and predicate, then filtered by mode, so the relative order is the same in every mode.
- **One table.** Unique index on the whole triple: (subject type, subject, predicate, object type, object). A triple exists at most once; the index must include the object, otherwise statements 42 and 43 would clash.
- **Objects are entities or typed literals** (`5`, a string, a boolean). `mode:web` is an entity of type `mode`, registered by the Modes module, so that the validity of a mode is checked by its entity type. The type `statement` is an entity type of the Triples module.
- **A qualifier is a predicate** of the same registry, for example `modes/mode` or `triples/position`. A predicate declares which predicates may qualify its statements (`illustrated-by` accepts `modes/mode` and `triples/position`; `modes/mode` accepts `triples/position`), and the existing limits express "at most one position".
- **Repeated facts** need nothing special. "Worked at X from 2010 to 2012, then from 2015 to 2018" is `(1: A, worked-at, X)`, `(2: 1, from, "2010 to 2012")`, `(3: 1, from, "2015 to 2018")`: same subject and predicate, different objects, so the unique index accepts both. If each period has its own role, the role qualifies the statement of the period: `(2, role, engineer)`, `(3, role, manager)`.
- **Limits.** A qualifier has a single object: a structured value needs a typed literal (an interval) or an anonymous node (not planned). A literal string is limited to 191 bytes, which suits a mode, a rank or a short caption, not a long text.
- **Consequences.** Deleting a statement deletes, recursively, the statements about it (the same mechanism as the cleanup when a post or a media item is deleted). Reading a photo with its modes and ranks needs joins, which the indexes support and an object cache will absorb. This is also RDF reification with an identifier per triple, so the export is direct.

Not planned: named graphs, inference.

Risk: a generic relation engine can absorb all the development time. **Proposed** mitigation: deliver first only the two uses that serve the books (template modes and the ordered content of a book) behind a clean API; add media and post-to-post relations as real needs appear. The "Posts 2 Posts" plugin did something similar for posts and is no longer maintained, which shows both the need and the effort.

## 2. Multiple templates

**Decided**: replace the naming-convention hack with explicit relations between templates, and generalize "print" into a **mode**, with several modes definable.

- A **mode** is a declared entity: slug (`print`, `book`, `cover`...), label, query-string trigger.
- A **relation** *(source template, mode) → target template* says "template `bar` is the `print` version of template `foo`". One target can serve several sources. A default target per mode is possible: *(\*, print) → print-default*.
- **Decided**: a mode is *not* a predicate. The mode of a relation is given by a further statement about the relation (see "Reification" above): "`bar` is the `print` version of `foo`" is `(template:foo, has-variant, template:bar)` plus `(that statement, mode, mode:print)`.
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
- **Proposed**: `book` and `cover` are modes of domain 2, each with its own target templates. Each post is rendered in the `print`/`book` mode. A photo can then be attached to a post for the web only, for print only, or for both, through statements about the `mode`.
- **PDF production, proposed order**:
  1. A single "book" page: all posts concatenated with a table of contents, CSS `@page` rules and page breaks, turned into a PDF by the browser or a paged-media tool. This needs no external binary and keeps the "no `shell_exec`" rule used in `wp-i18nly`.
  2. Later, optionally, server-side rendering (headless Chrome or a service) to produce PDFs automatically.
- Final assembly (cover, continuous pagination, binding margins) could be done in PHP with a library or stay in an external tool.

Format today: A5, printed at coollibri.com (a detail for now; page size must stay configurable per book).

### The manual finishing chain (from Eric's how-to notes)

Eric shared a short LibreOffice document ("howto", November 2025) describing how the first book, six months on the Camino de Santiago, was finished. It supersedes the earlier mention of PDF Arranger for the final assembly. The steps, all command-line except the table of contents and index:

1. **Merge.** The monthly PDFs are merged with `pdfcpu merge`. Each monthly file is named `<start-date>_<end-date>_<title>.pdf` (for example `20250404_20250503_compostelle_1er_mois.pdf`). A book is therefore a sequence of **parts** defined by a date range and a title, each part printed to one PDF from the browser.
2. **Resize to A5.** `pdfcpu resize "formsize:A5"`. The pages are therefore *not* printed at A5 from the browser: the layout is designed at another size and scaled down afterwards (A4 to A5 is a factor of about 0.71), so the fonts and the layout shrink with it.
3. **Reduce the photos.** Ghostscript (`gs -sDEVICE=pdfwrite -dPDFSETTINGS=/printer ...`) downsamples the images to shrink the file, presumably for the printer's upload limit.
4. **Table of contents and index.** Built in a spreadsheet, exported as tab-separated CSV, inserted into Writer ("Insert / Text from File") and formatted with styles; the result becomes part of the PDF (the intermediate file is called `compostelle_wip.pdf`).
5. **Page numbers.** Stamped on the final PDF with `pdfcpu stamp add -mode text -- "%p" ...` (bottom centre, small grey label with a rounded border). The numbers are the **physical page numbers of the PDF**, front matter included, not CSS counters. This is why the table of contents and index could be computed from the position of each post.

Consequences for the design (**proposed**):

- Everything after the browser step is command-line and automatable. A **finishing pipeline** with replaceable steps (merge, resize or scale to the book format, reduce images, stamp folios) fits behind the renderer interface; a sidecar could ship a Chromium together with `pdfcpu` and Ghostscript, which are Eric's current tools.
- Stamping folios on the final PDF is robust: it does not depend on CSS page-margin support, and the physical page number matches what the table of contents and index cite. Roman numerals for the front matter would need stamping a page range (to verify in `pdfcpu`).
- The book format should be set where the page is laid out (`@page` size in the book template) rather than by scaling afterwards; the scaling step can stay as a compatibility option.
- A part is defined by a date range and a title: composing a book from "a period", as Eric did with database queries, should be a first-class way to build the list of posts, with parts as sections of the ordered `contains` statements.
- Eric's caveat: this chain describes how the **first book** was made; it is a record of needs and constraints, not necessarily what he wants to reproduce.
- Answers: the pages were printed from the browser at **A4** (then scaled to A5); in the first book the **table of contents is at the beginning and the index at the end**; the index was formatted in Writer (ODT) and is too large to share.

Data from the table of contents of the first book (a spreadsheet of 215 rows: title and page, analyzed on 2026-10-09):

- Six parts ("Premier mois" to "Sixième mois") starting on pages 8, 42, 79, 116, 153 and 190; the last entries are "Carte" (page 230) and "Index" (page 231). The body starts on physical page 8, so seven pages precede it (cover, title and the table of contents).
- Entries are post titles (not dates). Besides the daily posts, each part has recurring sections: an opener (the month), often a "Résumé", and at the end a "whoBIRD" section (bird detections) and a "Carte" (map). Other repeated titles: "Le camino" (4 times), "Gastronomie".
- **207 of 214 units take exactly one page.** Seven take more: five of two pages and two of three pages, almost all of them "whoBIRD" sections (their length depends on the number of detections), plus one "Gastronomie". With a default of one page per unit, only a handful of exceptions would have to be declared or measured, and they belong to a recognizable kind of content.
- The table of contents has about 5 pages at the front, and **its length does not depend on the page numbers it contains** (digits do not change the number of lines): rendering it once with placeholder numbers gives its page count, then the real numbers can be filled in. With the table of contents at the front, two steps are enough; the index at the end shifts nothing.
- The structure of a part is regular: opener, optional summary, daily posts, whoBIRD, map. This suggests a **kind** on each unit of the ordered `contains` statements (opener, summary, day, appendix...).

### Table of contents and index

Eric builds both by hand today and finds them useful features; the index "needs real thinking". **Proposed**, nothing decided.

- **The core difficulty is page numbers.** They exist only after pagination, which happens in the renderer, not in WordPress. A table of contents or index *with page numbers* therefore needs a renderer that can resolve them: a paged-media polyfill such as Paged.js (`target-counter()`), or a two-pass process (render, read the page of each anchor, inject the numbers, render again). Plain browser printing cannot do it. A table of contents *without* page numbers (sections, post titles, dates) is independent of the layout and can come first.
- **Table of contents.** Derived from the structure of the book: sections and ordered posts (the ordered `contains` statements and the statements about them that give their section). The simpler of the two features.
- **Where index entries come from** (three sources, which can be combined):
  1. **Taxonomy terms** (tags, categories such as places or species): each term used by the posts of the book becomes an entry, with the posts as locators. Automatic, but coarse.
  2. **Explicit marks in the text**: an inline "index entry" mark with an optional sub-entry, a sort key and a locator anchored in the passage (the idea of `\index` in LaTeX or `indexterm` in DocBook). Precise, but costly to author by hand; the step that generates the post HTML could also emit the marks.
  3. **Statements**: an entry is the object of a statement such as `(post:12, mentions, term:45)`; sub-entries come from a "broader" relation between terms, "see also" from a relation between entries. Statements about the mention carry what an index needs: the passage anchor, whether the mention is main or passing (bold page number), and the mode (print only). This ties the index to the triples domain.
- **What a real index needs:** entries and sub-entries; "see" and "see also"; sorting that follows the language (accents, ignored leading articles, explicit sort keys; PHP's `intl` `Collator` is optional, so availability is to verify); letter headings; page ranges collapsed (12-14); main references highlighted; display forms that differ from the sort form.
- **Eric's experience (one book printed so far).** He checked by hand that each daily post fits on one page. Database queries then extracted a CSV (without page numbers) of the index entries, and the page numbers were computed and added in a second step, since a post's page follows from its position. He indexed mainly place names, then itinerary names, and, occasionally, notable incidents. Page numbers in the table of contents and index are really useful if they can be included.
- **Page numbers without a layout engine (proposed).** Eric's trick generalizes: if every post of a book starts on a new page (CSS page break before each post) and each post declares how many pages it takes (default 1, a statement about the `contains` statement), then the first page of each post is the cumulative sum of the previous page counts plus the front matter. No renderer is needed. A verification step must catch a post that overflows its declared count: the renderer, when present, reports the real page count of each post and the plugin warns on a mismatch; without a renderer, the print preview is the check. Blank pages needed so that sections start on a right-hand page can be modeled the same way.
- **Page resolution as a replaceable strategy (proposed).** Two strategies behind one interface, so that the table of contents and the index do not care which one is used: *declared* (the counting above, no dependency, post-level locators) and *measured* (Paged.js or a two-pass render, which handles posts of arbitrary length and passage-level anchors).
- **What gets indexed, and where it lives today (Eric).** Itineraries (GR10, Via Tolosana...), countries, regions and incidents are all **tags** (`post_tag`). In the printed book an itinerary tag only says that the route was followed or crossed that day, so a route is *not* a richer entity (an earlier idea of stages `part-of` a route is dropped). Cities and villages were typed in by hand, not tagged. The printed book has a single combined index.
- **Consequences for the index (proposed).**
  - Entries come from **index sources**, each one a taxonomy (any taxonomy, not only tags). The first sources are taxonomy terms; statements and inline marks come later.
  - Cities and villages: a non-public **hierarchical taxonomy** (country, region, place) used from the normal editor box, with autocomplete on names already entered, is the cheapest way to capture them at writing time; its hierarchy gives sub-entries for free. A migration helper could fill it from existing posts.
  - Tags are not hierarchical and mix several meanings, so each term needs a **kind** (place, route, incident) to be indexed and presented, and terms with no kind stay out of the index. The kind can be a statement `(term:45, has-kind, kind:route)` or term meta. If country > region sub-entries matter for tags, a "broader" statement between terms provides them.
  - One combined index is the default; the kind can drive typography or a per-kind index later.
  - **Eric's answers (to be refined later):** the selection of tags for the first book's index was done by hand, and the types of tags (place, route, incident) were told apart by **typography** in the formatted index, not by data. A flat index was enough (no country > region > city hierarchy). Cities and villages could simply be added as tags too, which avoids a dedicated taxonomy. So the kind of a term and its inclusion in the index become explicit data (a term with no kind is excluded) and typography follows from the kind; a hierarchy stays an option.
- **Parts of variable length (Eric's objection).** The table of contents, the index and chapter openings have no fixed page count, so declaring counts for them by hand would burden the user. Proposed ways to remove most of that burden:
  - **Numbering design (optional, no longer needed).** An earlier idea was roman numerals for the front matter and the table of contents after the body. Eric's first book shows it is not needed: the table of contents is at the front, its length is independent of its page numbers and is found by a first pass, and the folios are the physical pages of the PDF. Roman numerals and a table of contents at the back stay possible as layout options.
  - **Chapter openings:** give each opening a declared page count (default 1), as for posts, and **calibrate instead of asking the user to count**: a renderer measures the real count and the plugin stores it; without a renderer, a "Pages" screen lists each unit with its status (declared, verified, mismatch) and lets the user type the count seen in the print preview once. An idea to evaluate: import the produced PDF and find the page of each post title to fill the counts.
  - With a measuring renderer none of this is needed; the declared strategy is the dependency-free baseline.
- **Export.** The table of contents and the index can also be exported as CSV or JSON (with page numbers when known), which replaces Eric's database queries and keeps external steps possible.
- **Delivery order (proposed):** table of contents without page numbers first; then page numbers by the declared strategy (page break per post, page counts, overflow check); then the index (places and routes from terms and statements, incidents from marks), as its own slice after a design note; the measured strategy later, when posts longer than a page or passage-level anchors are needed.

## RDF import and export

**Proposed**, **not a priority**. Eric's motivation: partly nostalgia (he worked on RDF in the 2000s) and, more seriously, the possibility of using semantic web tools if the need arises.

Why it fits:

- The core model is already a set of *(subject, predicate, object)* statements. Typed identifiers map to IRIs, and predicates of the registry can carry an optional IRI.
- Statements about statements map to RDF through classic reification (a statement identifier as subject), or RDF-star / RDF 1.2 annotations (more elegant, less well supported by tools; the current state of PHP library support was not checked).
- Serializations: **Turtle** for humans and diffs (N3 is a superset with rules, which are not needed), **N-Triples** for streaming, **JSON-LD** as the most natural for WordPress (it is JSON, close to the REST API, and needs no heavy parser to produce).

What it would bring: no lock-in, existing tools (SPARQL, SHACL, visualization), bulk editing of relations as text under version control, and links to external authorities (Wikidata, GeoNames for places, taxonomies for species) that would also give authority to the index and allow `schema.org` JSON-LD in pages.

Costs and pitfalls:

- **Stable IRIs.** Eric's remark: only media and posts have natural URIs. Public terms do have archive URLs, but they change with slugs and the permalink structure, and templates have none. A site-controlled IRI base independent of permalinks is needed, with a resolver per entity type.
- **Import is much harder than export.** A Turtle serializer is about a hundred lines; import needs a parser (a dependency, whereas the plugin has none), blank nodes, typed literals, mapping IRIs back to local identifiers, validation and size limits for untrusted input.
- **Out of scope:** an internal RDF store, a SPARQL endpoint, inference.

Direction:

1. JSON stays the main exchange format (exact round trip, statements about statements included), as decided.
2. Later, add a JSON-LD export, then Turtle, as a function of the `Triples` module, not a new module.
3. Consider RDF import only if a real need appears (for example seeding places from Wikidata).
4. Prepare the ground now at almost no cost: an optional IRI per predicate, an IRI resolver interface per entity type, stable statement identifiers, and typed literals (string, integer, boolean) so that they map to RDF datatypes.

## Packaging

**Decided: one plugin with three modules** (`Triples`, `Modes`, `Books`), built so that it can later be split into three plugins at little cost. Extract the core into its own plugin once its API has been stable for a while and a second consumer exists.

Reasons:

- The triples model is still moving (see the reification discussion); three plugins would freeze an unstable cross-plugin API from the first day.
- The core tables would be owned by one plugin while holding the data of the others; uninstalling it would destroy or orphan that data. WordPress 6.5+ `Requires Plugins` declares dependencies but does not constrain versions.
- Costs triple with three plugins: translations, `uninstall.php`, schema migrations, admin screens, CI and releases.
- Against: the boundaries are enforced by discipline and tests rather than by packaging, which is the reason for the rules below. Eric's earlier plugins (`wp-scatter-elsewhere`, `wp-media-helper`) are separate repositories.

Namespaces are necessary but not sufficient. The eight rules that keep the split cheap:

1. **One-way dependencies, checked by an automated test** (`deptrac` or equivalent): `Triples` knows nothing, `Modes` sees only `Triples`, `Books` sees only the other two.
2. **Talk through a public API, not concrete classes.** `Modes` never writes SQL in the `Triples` tables; it goes through interfaces and hooks (for example registering the `mode` entity type).
3. **Each module owns its data:** creation and migration of its tables, its schema version in its own option, its own cleanup in `uninstall`.
4. **Names belong to the module, not to the umbrella:** table names (`triples_statements`, not `otherguise_statements`), options, hooks, REST namespace (`triples/v1`), capabilities and text domain. This is the costliest to fix afterwards, because renaming stored data and settings needs a migration.
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
- **The "Regression" commit** concerns the print CSS the plugin injects: a rule was removed. Which rule is not known, and the repository has a single commit, so there is no history to compare. It shows how fragile a global stylesheet full of `!important` overrides is. Eric recalled the rule as `.skip-link.screen-reader-text { display: none !important; }`; it is present in the committed file (last selector of the first rule), so the repository copy already has it and the deployed copy may differ. It is very specific to the current theme.
- **The print links** are included discreetly in Eric's view templates so that a visitor can reach the print version.
- **PDFs are produced and assembled by hand today:** each page is printed to PDF with the Samsung Internet browser on Android (the only browser Eric found that does not add a header and footer), then the PDFs are arranged and merged with PDF Arranger on Ubuntu. Automation would save a lot of time.

### PDF production: automation options

**Proposed**, nothing decided.

- **Constraint.** A PHP PDF library (Dompdf, mPDF, TCPDF) is a poor fit: block themes rely on modern CSS (flex, grid, CSS variables), and the pages contain JavaScript-rendered maps (Leaflet, WP GPX Maps) and lazily loaded images. A real browser engine is needed. The plugin itself stays PHP-only (no `shell_exec`), so rendering happens outside WordPress.
- **What the plugin provides.** The `print`/`book` modes and their templates; a **single book page** (all posts of a book in order, table of contents, `@page` size and margins, continuous page numbers); a **manifest** of the book (ordered list of URLs with mode and options) through the REST API and WP-CLI.
- **What an external renderer does.** Opens the book page, or each URL of the manifest, in headless Chromium (for example Playwright, `page.pdf()` with header and footer disabled and the CSS page size preferred), waits for maps and images to finish loading, and merges the PDFs if there are several. With one book page, page numbering, table of contents and running headers become continuous, which per-post PDFs merged afterwards cannot give.
- **Eric's preference:** ideally on the server, but that means installing extra software, which could put off other users of the plugin. His own site runs under Docker, so it is not a problem for him. Serving the assembled result as HTML + CSS and printing from a browser (Samsung Internet while it adds no header, or a small dedicated Android app) is also considered.
- **Renderer backends** (**proposed** design): optional and pluggable behind one interface, all consuming the same book page and manifest.
  1. **Browser, the default, no dependency.** The plugin serves the book as HTML + CSS and the user prints it from a browser. `@page { margin: 0 }` with padding on the content normally hides the browser's own header and footer in Chromium-based browsers (to verify on Samsung Internet), which would remove the dependence on one browser. A small Android app (a WebView and the Android print framework) is possible but is a separate project and is not planned.
  2. **Sidecar HTTP renderer, optional.** A headless Chromium service in a container next to WordPress, called over HTTP by the plugin: no `shell_exec`, nothing installed in the WordPress container. It fits Eric's Docker setup and stays an opt-in for other users. Gotenberg is a known candidate (HTML or URL to PDF, and PDF merging); its current API and how to wait for the maps are to verify. The service must be able to reach the site, and the plugin only ever sends URLs of its own site.
  3. **External tool consuming the manifest, optional:** a command-line tool on a laptop or in CI.
- **Limitation to check:** Chromium does not support the paged-media features needed for a table of contents with page numbers and running headers (`target-counter()`, running elements). Paged.js polyfills them in the browser; a renderer based on Chromium shares the limitation unless it also runs such a polyfill.
- **Print link in the views** (**decided**): a block that builds the link to the current page in a given mode, shown only when a relation exists for the current template and that mode, replacing hand-written links in the templates.
- **Open for Books:** whether any renderer backend lives in this project or in a separate tool, and how far "Books" goes beyond composing the book and exposing the manifest.

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
2. ~~Names~~: decided, the plugin is `wp-otherguise`; the modules are `Triples`, `Modes` and `Books`. To check again if the plugin is submitted to WordPress.org: the slug `otherguise` was free on 2026-10-09.
3. **PDF automation**: the current manual chain and the options are described above. To decide: which backends to build first (browser output is the baseline) and whether any lives in this project.
4. ~~Page formats and printers~~: A5 at coollibri.com for now; keep the format configurable. Still open: index design (see "Table of contents and index").
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
