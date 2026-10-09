<!-- SPDX-FileCopyrightText: 2026 Eric van der Vlist <vdv@dyomedea.com> -->
<!-- SPDX-License-Identifier: GPL-3.0-or-later -->

# wp-otherguise

**Otherguise** ("same content, another guise") is a WordPress plugin that lets a site present the same content in several forms (web, print, books) and helps assemble books from posts. It replaces the `?print` hack described in the [Balisage 2026 presentation](https://evlist.github.io/balisage-2026/).

The project lives in [`evlist/wp-otherguise`](https://github.com/evlist/wp-otherguise). **This directory only keeps the principles and the links.** The design notes, the decisions and the slices of work are in that repository, so that they stay next to the code and are maintained in one place. Update this file only when a principle changes.

## Principles

- **One plugin, three modules with one-way dependencies:** `Triples` (relations), `Modes` (alternative templates), `Books` (assembling books). Built so that it can be split into three plugins later if useful.
- **Everything is a statement.** A relation is a triple *(subject, predicate, object)* that has an identity; a qualification (the mode, the rank) is another statement whose subject is the statement qualified. One table, no special metadata.
- **Modes replace the naming hack.** A mode (`print`, `book`...) is selected by the query string (`?mode=print`); "template `bar` is the `print` version of template `foo`" is a declared relation, with a fallback to the normal template. Block themes and templates edited in the site editor must be supported.
- **Books are ordered lists.** A book is built from posts (by period, in order); a table of contents and an index are generated; page numbers do not need a layout engine when each post fills a declared number of pages.
- **Printable HTML and CSS is the baseline.** PDF rendering by a headless browser is an optional backend outside WordPress; the plugin does not run external commands.
- **Quality from the first slice:** small tested slices, WordPress coding standards, REUSE, capability checks, translation and uninstall handling built in, and an honest account of what was never run on a real WordPress site.
- **Media Helper stays independent;** this plugin plugs into extension points rather than the reverse.

## Where to read more

All in `evlist/wp-otherguise`:

| Subject | Document |
|---|---|
| Architecture and current state | [`docs/IA.md`](https://github.com/evlist/wp-otherguise/blob/main/docs/IA.md) |
| Slices of work, done and planned | [`docs/slices/README.md`](https://github.com/evlist/wp-otherguise/blob/main/docs/slices/README.md) |
| Design notes and open questions | [`docs/design/README.md`](https://github.com/evlist/wp-otherguise/blob/main/docs/design/README.md) |
| Statements, the registry, RDF | [`docs/design/triples.md`](https://github.com/evlist/wp-otherguise/blob/main/docs/design/triples.md) |
| Modes, templates, the current hack | [`docs/design/modes.md`](https://github.com/evlist/wp-otherguise/blob/main/docs/design/modes.md) |
| Books, table of contents, index, PDF | [`docs/design/books.md`](https://github.com/evlist/wp-otherguise/blob/main/docs/design/books.md) |
| Integration with Media Helper | [`docs/design/media-helper.md`](https://github.com/evlist/wp-otherguise/blob/main/docs/design/media-helper.md) |
| One plugin, three modules | [`docs/design/packaging.md`](https://github.com/evlist/wp-otherguise/blob/main/docs/design/packaging.md) |

Related projects: `wp-media-helper` and `wp-scatter-elsewhere` (media and YouTube), `wp-i18nly` (the model for the layout and the tooling), and the plugin `wp-pdf-helper` (<https://gitea.dyomedea.com/vdv/wp-pdf-helper>), which holds the hack being replaced.
