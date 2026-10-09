# shared-room

A common space for my projects: what concerns several repositories, and the state of our thinking.

## Filing rule

- What is **cross-cutting** (vision, conventions, decisions affecting several projects, shared tooling) lives here.
- What is **specific to one project** lives in that project's repository. This repository only keeps a companion directory for it: notes, decisions and open questions about that project (one directory per repository, created as needed).
- Do not duplicate: a link is better than a copy.
- Documents are written in English.

## Context

The WordPress projects below serve one goal: produce a single body of content (a travel blog) and publish it in several forms (web, print, books, YouTube). Reference presentation: <https://evlist.github.io/balisage-2026/> (repository [`balisage-2026`](https://github.com/evlist/balisage-2026)).

## My repositories

The **Claude Code** column says whether I have worked on the repository with Claude Code, based on the sessions found on 2026-10-09:

- **Yes**: at least one Claude Code session linked to this repository was found.
- **Not observed**: no session found. This does not prove that no work was done there (older sessions, another tool); correct by hand.

### Active WordPress plugins

| Repository | Claude Code | Notes |
|---|---|---|
| [`wp-i18nly`](https://github.com/evlist/wp-i18nly) | Yes | Translation workflow inside WordPress. Successor of `wp-i18n-404-tools`. |
| [`wp-otherguise`](https://github.com/evlist/wp-otherguise) | Yes | Same content in another guise: relations, alternative templates by mode, books. Replaces the `?print` hack. Work in progress. |
| [`wp-media-helper`](https://github.com/evlist/wp-media-helper) | Yes | Media sources and thumbnails. A security audit session took place there. |
| [`wp-scatter-elsewhere`](https://github.com/evlist/wp-scatter-elsewhere) | Yes | Upload and update of YouTube videos from posts. Formerly `wp-scatter-everywhere`. |
| [`wp-whobird`](https://github.com/evlist/wp-whobird) | Not observed | To be completed. |

### Other repositories

| Repository | Claude Code | Notes |
|---|---|---|
| [`balisage-2026`](https://github.com/evlist/balisage-2026) | Not observed | Balisage 2026 presentation. |
| [`codespaces-grafting`](https://github.com/evlist/codespaces-grafting) | Not observed | Codespace template used by `wp-i18nly`. |
| [`wp-i18n-404-tools`](https://github.com/evlist/wp-i18n-404-tools) | Not observed | Archived. Predecessor of `wp-i18nly`. |
| [`orbeon-forms`](https://github.com/evlist/orbeon-forms) | Not observed | 2013 repository, inactive since. See also [`orbeon/orbeon-forms`](https://github.com/orbeon/orbeon-forms). |

### Forks

| Repository | Claude Code | Notes |
|---|---|---|
| [`wp-plugin-trackserver`](https://github.com/evlist/wp-plugin-trackserver) | Not observed | Fork. |
| [`fr-thumbnails-folder`](https://github.com/evlist/fr-thumbnails-folder) | Not observed | Fork. |
| [`simple-image-sizes`](https://github.com/evlist/simple-image-sizes) | Not observed | Fork. |
| [`sample-wordpress-plugin`](https://github.com/evlist/sample-wordpress-plugin) | Not observed | Fork. |
| [`localized-strings`](https://github.com/evlist/localized-strings) | Not observed | Fork. |
| [`deplacement-covid-19`](https://github.com/evlist/deplacement-covid-19) | Not observed | Fork (2020). |
| [`Saxon-CE`](https://github.com/evlist/Saxon-CE) | Not observed | Fork (2013). |
| [`msv`](https://github.com/orbeon/msv) | Not observed | Repository of the `orbeon` organization. |

## Projects

- [`wp-otherguise/`](wp-otherguise/README.md): the plugin that replaces the `?print` hack and helps assemble books. Principles and links only; the project lives in [`evlist/wp-otherguise`](https://github.com/evlist/wp-otherguise).

## To do

- [ ] Complete the notes of the repositories marked "To be completed" or "Not observed".
- [ ] Create a directory for each repository that needs one.
- [x] Add a `CLAUDE.md` with the context to load in every session.
- [ ] Choose a license for this repository.
