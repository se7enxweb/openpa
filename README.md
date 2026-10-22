# OpenPA

OpenPA is the toolkit OpenContent built for the websites of Italian public administrations
(municipalities, unions of municipalities, local agencies) on the Exponential legacy kernel. It brings the
building blocks such a site needs: the "Amministrazione Trasparente" (transparency) section and its admin tab,
an "Apps" section with its own admin tab, the organization chart (organigramma), roles and staff lookups,
SEO and cookie settings, reCAPTCHA keys, automatic changes of object state and section when content expires,
menu caching, a `robots.txt` and a `sitemap.xml`, plus many template operators and fetch functions used by
the OpenPA designs.

The extension, its settings and its templates are in Italian, as the project is. This guide is in English;
a short Italian summary is at the end ([Sommario in italiano](#sommario-in-italiano)).

Who it is for: integrators who run or migrate an OpenPA site on Exponential 6.x, and developers who want to
reuse its operators, fetch functions or workflow events.

## Contents

- [Requirements](#requirements)
- [Installation](#installation)
- [What it adds](#what-it-adds)
- [Configuration](#configuration)
- [Usage](#usage)
- [Permissions](#permissions)
- [Cron jobs and scripts](#cron-jobs-and-scripts)
- [Upgrading](#upgrading)
- [Known limits](#known-limits)
- [Troubleshooting](#troubleshooting)
- [Contributing](#contributing)
- [License and credits](#license-and-credits)
- [Sommario in italiano](#sommario-in-italiano)

## Requirements

- Exponential 6.x (legacy kernel) on PHP 8.0 or later.
- The extensions named in `extension.xml`: `ezmultiupload` (required); `ezwebin`, `ezjscore` and
  `wrap_operator` (extended).
- Optional, used by parts of the code when present: eZ Find / Solr (search operators, the `index_pending`,
  `update_class_pending` and `rename_object` cron jobs), eZ Flow (the block handlers and the
  `ezflowcustomblockupdate` workflow event), eZ Tags, and an extension that provides `OcReCaptchaType`
  (reCAPTCHA validation).

## Installation

### With Composer

```bash
composer require se7enxweb/openpa
```

The legacy installer places the package in `extension/openpa`.

### By hand

```bash
cd /path/to/exponential
git clone https://github.com/se7enxweb/openpa.git extension/openpa
```

### Activate it

Add the extension to `settings/override/site.ini.append.php`:

```ini
[ExtensionSettings]
ActiveExtensions[]=openpa
```

Then regenerate the autoloads and clear the caches:

```bash
php bin/php/ezpgenerateautoloads.php -e
php bin/php/ezcache.php --clear-all
```

## What it adds

| Part | Where | What |
|---|---|---|
| Admin tab "Apps" | `settings/menu.ini.append.php` `[Topmenu_apps]` | Opens `apps/dashboard`; shown with the `puzzle` icon in admin4. |
| Admin tab "Amministrazione Trasparente" | `[Topmenu_trasparenza]` | Opens the `Amministrazione-Trasparente` URL alias; shown with the `journal-text` icon in admin4. |
| Setup menu links | `[Leftmenu_setup]` | "Impostazioni SEO" (`openpa/seo`), automatic state change settings, automatic section change settings. |
| Modules | `modules/` | `openpa`, `apps`, `robots.txt`, `sitemap.xml` (see [Usage](#usage)). |
| Template operators | `autoloads/eztemplateautoload.php` | About 55 operators (see below). |
| Fetch functions | `modules/openpa/function_definition.php` | `fetch( 'openpa', ... )`, 21 functions. |
| Workflow events | `eventtypes/event/` | 8 event types (see below). |
| Cache items | `settings/site.ini.append.php` `[Cache]` | `openpamenu`, `openpadynamicini`, `openpacontextdata`, `openpaorganigramma`, all tagged `openpa`. |
| Translations | `translations/ita-IT` | Italian. |
| Designs | `design/standard`, `design/admin2` | Templates for the site and the admin interface. |

The `Icon=` lines name icons from the kernel's `design.ini` `[AdminDesignSettings] IconList[]`; a design that
does not read `Icon=` ignores them.

## Configuration

All settings follow the usual rule: never edit the files in `extension/openpa/settings`, put your changes in an
override such as `settings/override/openpa.ini.append.php` or a siteaccess file.

### openpa.ini

`settings/openpa.ini` holds the defaults; `settings/openpa.ini-dist` lists the node and link settings a site
usually overrides. The main blocks:

| Block | Purpose |
|---|---|
| `[ObjectHandlerServices]` | The services behind the `object_handler` operator (`content_main`, `control_menu`, `content_gallery` ...). |
| `[AttributeHandlers]`, `[BlockHandlers]`, `[DataHandlers]` | PHP classes per datatype/attribute, per block type/view, and per `openpa/data` handler (`politici`, `map_markers`, `partecipazioni`, `tree`). |
| `[CacheSettings]` | `Menu=enabled` turns the menu cache on. |
| `[ContentMain]`, `[Attributi]` | Which attributes count as image, abstract and full text. |
| `[Seo]` | Google Analytics, robots, meta tags; edited at `openpa/seo` and stored in the database (site data `SeoSettings`). |
| `[LinkSpeciali]`, `[Nodi]`, `[TopMenu]`, `[SideMenu]`, `[ExtraMenu]` | Node ids for special pages and the menus. |
| `[GestioneClassi]`, `[GestioneAttributi]`, `[GestioneSezioni]` | Class, attribute and section lists used by the templates. |
| `[MotoreDiRicerca]`, `[MotoreRicerca]` | Search form settings. |
| `[ChangeState]`, `[ChangeSection]` | Rules for the `change_state` and `change_section` cron jobs. |
| `[NetworkSettings]` | `PrototypeUrl`: where `openpa/class` and the class installer fetch class definitions from. |

Example override:

```ini
<?php /* #?ini charset="utf-8"?

[CacheSettings]
Menu=enabled

[LinkSpeciali]
NodoContattaci=1234
NodoPrivacy=1240

[ChangeSection]
ToSectionDefault=oggetti_scaduti
ToSection[bando]=oggetti_scaduti

*/ ?>
```

Settings are read through `OpenPAINI::variable()`, which serves some of them (the `Seo` keys,
`GeneralSettings::theme` and a few more) from the database instead of the INI file.

### Automatic state change

```ini
[ChangeState]
Rules[]=deliberazione|deliberazione_archivia

[ChangeStateRule-deliberazione_archivia]
CurrentState=albotelematico.visibile
DestinationState=albotelematico.archivioricercabile
# <attribute_identifier>;<operator>;<value>, operator eq, gt, ge, lt, le; value NOW, TODAY or a string
Conditions[]=data_finepubblicazione;lt;TODAY
```

The same rules can be maintained in the admin at `openpa/changestatesettings`.

### Automatic section change

`[ChangeSection]` maps each class to a target section (`ToSection[<class>]`, default `ToSectionDefault`), the
date attribute that decides expiry (`DataTime[<class>]`), an optional lifetime in seconds
(`ScadeDopoTotSecondi[<class>]`) and the subtree to scan (`RootNodeList[<class>]`). The admin view is
`openpa/changesectionsettings`.

### Other INI files

The extension also ships settings for `menu.ini`, `module.ini`, `site.ini`, `content.ini`, `design.ini`,
`override.ini`, `template.ini`, `workflow.ini`, `cronjob.ini`, `ezjscore.ini`, `ezfind.ini`, `image.ini`,
`dashboard.ini`, `zone.ini`, `squid.ini` and others in `settings/`. `block.ini.append.php_move_to_siteaccess`
is a block definition meant to be copied into a siteaccess, not read from the extension.

## Usage

### Admin tabs

After activation the admin header shows two extra tabs. In admin4 "Apps" carries the `puzzle` icon and
"Amministrazione Trasparente" the `journal-text` icon.

- **Apps** opens `apps/dashboard`, the full view of the site's Apps container. On first use it installs the
  `apps_container` class (from `PrototypeUrl`), creates the "Apps" section and publishes the container object.
- **Amministrazione Trasparente** opens the node with the URL alias `Amministrazione-Trasparente`.

### Module views

| URL | Policy | What it does |
|---|---|---|
| `openpa/seo` | `openpa/seo` | SEO settings: analytics ids, robots, meta tags. |
| `openpa/recaptcha` | `openpa/recaptcha` | reCAPTCHA v2 and v3 keys. |
| `openpa/theme` | `openpa/theme` | Choose the site theme. |
| `openpa/changestatesettings`, `openpa/changesectionsettings` | same names | Rules for the automatic changes. |
| `openpa/class/<ID>`, `openpa/classlist`, `openpa/classbydatatype/<datatype>`, `openpa/classsettings/<identifier>` | `openpa/class` | Compare and sync classes with the prototype. |
| `openpa/classdefinition/<ID>`, `openpa/classes`, `openpa/relations/<ID>` | `openpa/classdefinition` | Class definitions as data. |
| `openpa/roles` | `openpa/manage_roles` | Role management. |
| `openpa/add/<class>`, `openpa/addlocationto/<objectID>`, `openpa/refreshmenu`, `openpa/refreshorganigramma`, `openpa/block/<blockID>/<view>`, `openpa/settings` | `openpa/editor_tools` | Editor helpers. |
| `openpa/calendar/<nodeID>` | `openpa/calendar` | Calendar data for a node. |
| `openpa/data/<handler>`, `openpa/loadwt/<nodeID>` | `openpa/data` | Data from a `[DataHandlers]` handler. |
| `openpa/object/<objectID>` | `openpa/object` | Object data. |
| `openpa/signup`, `openpa/activate/...` | `openpa/signup` | User registration and activation. |
| `openpa/cookie` | `openpa/cookie` | Cookie consent. |
| `robots.txt`, `sitemap.xml` | none | Generated `robots.txt` and an essential `sitemap.xml`. |

### Template operators

`openpaini`, `get_main_style`, `has_main_style`, `is_area_tematica`, `get_area_tematica_style`,
`is_dipendente`, `openpa_shorten`, `has_abstract`, `abstract`, `rss_list`, `materia_make_tree`, `access_style`,
`unique`, `find_first_parent`, `current_object_id`, `fix_dimension`, `object_state_list`, `solr_field`,
`solr_meta_field`, `solr_subfield`, `solr_meta_subfield`, `search_exclude_class_facets`,
`search_exclude_classes`, `search_query`, `strReplace`, `organigramma`, `trasparenza_root_node_id`,
`openpa_instance_identifier`, `openpa_recaptcha_public_key`, `top_menu_cached`, `left_menu_cached`, `tree_menu`,
`slugize`, `cookieset`, `cookieget`, `check_and_set_cookies`, `checkbrowser`, `is_deprecated_browser`, `sort`,
`rsort`, `asort`, `arsort`, `ksort`, `krsort`, `natsort`, `natcasesort`, `find_global_layout`, `query_string`,
`openpapagedata`, `fill_contacts_matrix`, `contacts_matrix_fields`, `parse_contacts_matrix`, `openpacontext`,
`object_handler`, `apps_root`.

Examples:

```
{* a setting with a default, as the shipped page_head templates do *}
{if openpaini( 'Seo', 'GoogleTagManagerID', false() )}
    ...
{/if}

{* the services of a node or block *}
{def $openpa = object_handler( $node )}

{* strip tags and shorten a text to 120 characters *}
{$node.name|openpa_shorten( 120, '...' )}

{* a URL-safe slug *}
{'Amministrazione Trasparente'|slugize()}
```

### Fetch functions

`ruoli`, `homepage`, `nomi_ruoli_dirigenziali`, `dirigenti`, `footer_links`, `footer_notes`, `aree`, `servizi`,
`uffici`, `strutture`, `dipendenti`, `header_banner_background_style`, `header_logo_background_style`,
`calendario_eventi`, `faccette_classi_oggetti_correlati_inversi`, `list_count`, `list`, `tree`, `tree_count`,
`recaptcha_html`, `map_markers`.

```
{def $nomi  = fetch( 'openpa', 'nomi_ruoli_dirigenziali' )
     $ruoli = fetch( 'openpa', 'ruoli' )}

{fetch( 'openpa', 'recaptcha_html' )}
```

### Workflow events

Registered in `settings/workflow.ini.append.php`: `event_updatesection`, `event_ezflowcustomblockupdate`,
`event_addcalendarbyevent`, `event_ezprotectsubsite`, `event_reindexrelations`,
`event_reindexonupdatepriority`, `event_updateuseropenparole`, `event_refreshopenpapagedata`. Add them to a
workflow under Setup, Workflows, and attach it to a trigger.

## Permissions

The `openpa` module defines these functions (`modules/openpa/module.php`): `classdefinition`, `class`,
`manage_roles`, `data`, `calendar`, `editor_tools`, `object`, `signup`, `cookie`, `seo`, `recaptcha`,
`changestatesettings`, `changesectionsettings`, `theme`. The `apps` module defines `dashboard`; the "Apps" tab
is shown to users with `apps/dashboard`.

`site.ini.append.php` puts the public views (`openpa/classdefinition`, `calendar`, `object`, `data`, `signup`,
`cookie`, `loadwt`, `changestatedefinition`, `changesectiondefinition`, `robots.txt`, `sitemap.xml`) in
`[RoleSettings] PolicyOmitList[]`, so they need no policy.

## Cron jobs and scripts

`settings/cronjob.ini.append.php` defines these parts (run with `php runcronjobs.php <part>`):

| Part | Scripts |
|---|---|
| default | `change_section.php` |
| `frequent` | `clear_frontpages_cache.php`, `ezflowupdate.php`, `index_pending.php`, `update_class_pending.php` |
| `clearfrontpages` | `clear_frontpages_cache.php` |
| `changesection` | `change_section.php`, `change_state.php` |
| `changestate` | `change_state.php` |
| `generatemenu` | `generate_menu.php` (runs `bin/php/generatemenu.php` for each related siteaccess) |
| `rename` | `rename_object.php` |
| `rss` | `rssimport.php` |

`ezflowupdate.php` and `rssimport.php` come from eZ Flow and the kernel; the others are in `cronjobs/`.

```bash
php runcronjobs.php changesection
php runcronjobs.php frequent
```

`change_state.php` runs as the user `admin`. `index_pending.php`, `update_class_pending.php` and
`rename_object.php` need Solr as the search engine.

`bin/php/` holds maintenance scripts (class checks and sync, cache clearing, re-indexing, migrations, instance
creation). `bin/php/openpa.php` runs one of them on every OpenPA instance:

```bash
php extension/openpa/bin/php/openpa.php help
php extension/openpa/bin/php/openpa.php check_class --class=folder --exclude=siteA,siteB
```

## Upgrading

Update the package (`composer update se7enxweb/openpa`, or `git pull` in `extension/openpa`), then regenerate
the autoloads and clear the caches as in [Installation](#installation). `update/database/postgresql` holds a
PostgreSQL fix (`fix_integer_type.sql`). See `CHANGELOG.md` for what changed.

## Known limits

- The code was written for older PHP and kernel versions; this release changes only the admin tab icons and
  the release files, so test it on a copy of the site before going live.
- Class installation and `openpa/class` contact the prototype server named in `[NetworkSettings]
  PrototypeUrl`; without it they cannot fetch definitions.
- Many settings and templates assume the OpenPA content classes (`apps_container`, `dipendente`, `servizio`,
  ...).
- This repository's `master` branch is the 2.43 line; the 3.x line is on the `version3` branch.

## Troubleshooting

- **The tabs have no icon**: the icon appears only in designs that read `Icon=` (admin4). Clear the INI cache
  (`php bin/php/ezcache.php --clear-tag=ini`).
- **Menus do not change after editing**: clear the `openpa` cache tag
  (`php bin/php/ezcache.php --clear-tag=openpa`) or use `openpa/refreshmenu`.
- **"The current search engine plugin is not eZSolr"** from a cron job: that job needs Solr; remove it from
  the cron part on sites without Solr.

## Contributing

Issues and pull requests are welcome at https://github.com/se7enxweb/openpa. The upstream project is
https://github.com/OpencontentCoop/openpa.

## License and credits

GNU General Public License v2.0, see `doc/LICENSE`. Bundled third-party code in `lib/` keeps its own license
(listed in `extension.xml`).

OpenPA was designed, built and maintained by OpenContent (Opencontent SCARL, Trento, https://www.opencontent.it/)
from 2010, originally for eZ Publish, the CMS that preceded Exponential. 7x maintains this copy for
Exponential. The upstream README, with the full Italian notes on license, ethics and regulations, is in the
upstream repository.

## Sommario in italiano

OpenPA è la piattaforma di OpenContent per la comunicazione istituzionale e l'Open Government delle pubbliche
amministrazioni, qui distribuita per Exponential 6.x (PHP 8). Fornisce la sezione e la scheda "Amministrazione
Trasparente" (icona `journal-text`), la sezione e la scheda "Apps" (icona `puzzle`), l'organigramma, le
impostazioni SEO, cookie e reCAPTCHA, i cambi automatici di stato e di sezione per i contenuti scaduti,
`robots.txt`, `sitemap.xml`, operatori di template e funzioni fetch.

Installazione: `composer require se7enxweb/openpa` oppure copia in `extension/openpa`, poi
`ActiveExtensions[]=openpa`, `php bin/php/ezpgenerateautoloads.php -e` e
`php bin/php/ezcache.php --clear-all`. Le impostazioni si modificano con un override di `openpa.ini`.

Titolare della licenza e responsabile tecnico: Opencontent SCARL, https://www.opencontent.it/. Licenza GNU
General Public License v2.0.
