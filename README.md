# shared-room

Espace commun à mes projets : ce qui concerne plusieurs dépôts, et l'état de nos réflexions.

## Règle de rangement

- Ce qui est **transversal** (vision, conventions, décisions qui touchent plusieurs projets, outillage partagé) vit ici.
- Ce qui est **propre à un projet** vit dans le dépôt de ce projet. Ce dépôt n'en garde qu'un répertoire d'accompagnement : notes, décisions et questions ouvertes de ce projet (un répertoire par dépôt, à créer au fur et à mesure).
- Ne pas dupliquer : un lien vaut mieux qu'une copie.

## Contexte

Les projets WordPress ci-dessous servent un même but : produire un contenu unique (blog de voyage) et le diffuser sous plusieurs formes (web, impression, livres, YouTube). Présentation de référence : <https://evlist.github.io/balisage-2026/> (dépôt [`balisage-2026`](https://github.com/evlist/balisage-2026)).

## Mes dépôts

La colonne **Claude Code** indique si j'ai travaillé avec Claude Code sur le dépôt, d'après les sessions retrouvées au 2026-10-09 :

- **Oui** : au moins une session Claude Code liée à ce dépôt a été retrouvée.
- **Non observé** : aucune session retrouvée. Cela ne prouve pas que je n'y ai pas travaillé (anciennes sessions, autre outil) ; à corriger à la main.

### Plugins WordPress actifs

| Dépôt | Claude Code | Notes |
|---|---|---|
| [`wp-i18nly`](https://github.com/evlist/wp-i18nly) | Oui | Flux de traduction dans WordPress. Successeur de `wp-i18n-404-tools`. |
| [`wp-media-helper`](https://github.com/evlist/wp-media-helper) | Oui | Sources de médias, vignettes. Une session d'audit de sécurité y a eu lieu. |
| [`wp-scatter-elsewhere`](https://github.com/evlist/wp-scatter-elsewhere) | Oui | Envoi et mise à jour de vidéos YouTube depuis les articles. |
| `wp-scatter-everywhere` | Oui | Apparaît dans une session Claude Code, mais **absent de la liste des dépôts accessibles** : dépôt privé, renommé ou autre compte ? À éclaircir. |
| [`wp-whobird`](https://github.com/evlist/wp-whobird) | Non observé | À compléter. |

### Autres dépôts

| Dépôt | Claude Code | Notes |
|---|---|---|
| [`balisage-2026`](https://github.com/evlist/balisage-2026) | Non observé | Présentation Balisage 2026. |
| [`codespaces-grafting`](https://github.com/evlist/codespaces-grafting) | Non observé | Modèle de codespace utilisé par `wp-i18nly`. |
| [`wp-i18n-404-tools`](https://github.com/evlist/wp-i18n-404-tools) | Non observé | Archivé. Ancêtre de `wp-i18nly`. |
| [`orbeon-forms`](https://github.com/evlist/orbeon-forms) | Non observé | Dépôt de 2013, sans activité depuis. Voir aussi [`orbeon/orbeon-forms`](https://github.com/orbeon/orbeon-forms). |

### Forks

| Dépôt | Claude Code | Notes |
|---|---|---|
| [`wp-plugin-trackserver`](https://github.com/evlist/wp-plugin-trackserver) | Non observé | Fork. |
| [`fr-thumbnails-folder`](https://github.com/evlist/fr-thumbnails-folder) | Non observé | Fork. |
| [`simple-image-sizes`](https://github.com/evlist/simple-image-sizes) | Non observé | Fork. |
| [`sample-wordpress-plugin`](https://github.com/evlist/sample-wordpress-plugin) | Non observé | Fork. |
| [`localized-strings`](https://github.com/evlist/localized-strings) | Non observé | Fork. |
| [`deplacement-covid-19`](https://github.com/evlist/deplacement-covid-19) | Non observé | Fork (2020). |
| [`Saxon-CE`](https://github.com/evlist/Saxon-CE) | Non observé | Fork (2013). |
| [`msv`](https://github.com/orbeon/msv) | Non observé | Dépôt de l'organisation `orbeon`. |

## Projets en réflexion

Pas encore de dépôt. Pistes en cours de discussion, à consigner ici sous forme de décisions (une fois tranchées) :

- **Triplets** : un registre de relations *(sujet, prédicat, objet)* entre templates, articles, médias, etc.
- **Templates multiples** : appliquer un template différent selon un *mode* (`print`, `book`…), via des relations entre templates et un paramètre de requête.
- **Livres** : constituer un livre (contenu ordonné) à partir d'articles et assembler les PDF.

## À faire

- [ ] Compléter les notes des dépôts marqués « à compléter » ou « Non observé ».
- [ ] Créer un répertoire par dépôt qui en a besoin.
- [ ] Ajouter un `CLAUDE.md` avec le contexte à charger dans chaque session.
- [ ] Choisir une licence pour ce dépôt.
