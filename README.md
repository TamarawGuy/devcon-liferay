# DevCon — a Liferay 7.4 learning project

A conference site built from scratch on Liferay 7.4 GA132, to learn how the pieces of a
Liferay project connect: client extensions, site initializers, fragments, objects, and the
workflow a team uses to ship them.

Everything is built **source-first**: the site, its pages, its fragments, and its data model
all live in this repository as files. Nothing important exists only inside someone's local
database. That is what makes a change reviewable in a pull request and reproducible on
another machine.

## Contents

- [Getting it running](#getting-it-running)
- [What is here](#what-is-here)
- [Build log](#build-log)
- [Things that cost us time](#things-that-cost-us-time)
- [How to change things](#how-to-change-things)

## Getting it running

Requires Blade 8.x and a JDK 21 on `PATH`. The Gradle wrapper (8.9) is committed, so go
through `./gradlew` or `blade gw` rather than a local Gradle.

```bash
blade server init          # downloads the Liferay bundle into bundles/ (~2 GB)
blade server start --tail  # starts Tomcat, tails the log
```

Wait for `Server startup in [N] milliseconds`, then log in at `http://localhost:8080`.
Credentials are in [`CLAUDE.md`](CLAUDE.md).

Then deploy everything:

```bash
blade gw deploy
```

The site initializer autoprovisions the DevCon site on deploy, so `/web/devcon/home` should
render afterwards. Deploy a single extension instead with
`blade gw :client-extensions:<name>:deploy` — a full deploy rebuilds all of them.

**One step cannot be automated.** Go to **Site Builder → Pages → Configuration** on the DevCon
site and select the `DevCon Theme Css` theme. Until you do, the site renders with Liferay's
Classic theme and none of the branding applies. You will have to repeat this after every
theme redeploy — see
[Theme selection is manual, and every redeploy loses it](#theme-selection-is-manual-and-every-redeploy-loses-it).

> **`bundles/` is gitignored.** It holds the Liferay install and the database, and never
> travels through git. A `git pull` therefore never changes what your running portal serves —
> see [How to change things](#how-to-change-things).

## What is here

```
client-extensions/
  devcon-global-css/     globalCSS   — stylesheet injected on every page
  devcon-theme-css/      themeCSS    — the site theme; Clay build + design tokens
  devcon-site-init/      siteInitializer — the DevCon site, its pages and fragments
  devcon-batch/          batch       — object definitions and sample data
configs/                 per-environment portal properties (local, dev, uat, prod, docker)
.workspace-rules/        Liferay ruleset for AI agents; mirrored into .claude/
CLAUDE.md                workspace-specific facts, commands, credentials
```

### The site

`devcon-site-init` provisions a site with ERC `DEVCON` containing one Content Page, `Home`
at `/home`, carrying a `hero` fragment. Its source tree:

```
site-initializer/
  layout-set/public/metadata.json        theme assignment for the public layout set
  resource-permissions.json              role grants, e.g. Guest VIEW on Session entries
  layouts/01_home/
    page.json                            page metadata, incl. Guest VIEW permission
    page-definition.json                 fragment composition
  layout-page-templates/
    display-page-templates/session/      per-entry page for a Session
      display-page-template.json         which content type it renders
      page-definition.json               fragments, mapped via DisplayPageItem
  fragments/
    group/                               scope: this site. (company/ = whole instance)
      devcon-sections/                   fragment SET — dir name is its key
        collection.json                  the set's display label
        fragments/                       literal, required nesting level
          hero/                          FRAGMENT — dir name is its key
            fragment.json                htmlPath / cssPath / icon / type
            index.html                   editable regions via data-lfr-editable-*
            index.css                    all rules prefixed #wrapper .devcon-hero
      devcon-sessions/
        collection.json
        fragments/session-card/
```

**Directory names are the machine keys; the `name` in each JSON is the human label, and the two
are unrelated.** `page-definition.json` references the *fragment* key (`hero`, `session-card`)
plus `"siteKey": "[$GROUP_KEY$]"` — never the set key, which is why renaming a set costs
nothing outside its own folder.

### The data model

`devcon-batch` imports, in filename order:

| File | Creates |
| --- | --- |
| `00-object-folder` | `DEVCON` object folder |
| `01-object-definitions` | `Session`, then `Speaker` |
| `02-speakers` | 3 speakers |
| `03-sessions` | 6 sessions, linked to speakers by ERC |

```
Speaker  fullName (text, required) · company (text) · bio (long text)
Session  title (text, required) · abstract (long text) · startTime (datetime, UTC) · room (text)

Speaker  ──one-to-many──▶  Session      (deletionType: disassociate)
```

The relationship is declared on Speaker; Liferay auto-generates the foreign key column
`r_devconSpeakerSessions_c_speakerId` on Session. Entries are reachable at `/o/c/speakers`
and `/o/c/sessions`.

**Objects sit outside the site initializer on purpose.** They are company scoped, so they
survive the site deletion that a page change requires. Sample data is not lost every time a
fragment moves.

### The theme

`devcon-theme-css` replaces the Classic theme's stylesheets outright. Its source is four SCSS
partials; `blade gw deploy` runs Liferay's own theme builder against the `_styled` / `_unstyled`
parent themes and emits the compiled CSS into `static/`.

```
devcon-theme-css/
  client-extension.yaml            clayURL / mainURL / their RTL pairs / token JSON
  frontend-token-definition.json   which variables an editor may change, and their defaults
  src/css/
    clay.scss                      entry point — selects the custom-properties Clay build
    _clay_variables.scss           SCSS variables ($primary) — compiled in
    _clay_custom.scss              CSS emitted after Clay, used to force light mode
    _custom.scss                   theme-layer CSS (empty)
    _liferay_variables_custom.scss Liferay SCSS variables (empty)
```

Colour reaches the page by two routes, and the difference is the whole point of this extension:

```
_clay_variables.scss  →  $primary  →  compiled into clay.css   (build time, developer)
frontend-token-…json  →  :root { --… }  →  read by clay.css    (runtime, editor)
```

They meet at the fallback. Every Clay rule compiles to
`var(--btn-primary-background-color, #7b2ff7)` — the token wins if one is set, otherwise the
compiled SCSS value shows. Three tokens are exposed: `devconAccent`, `devconAccentAlt`, and
`btnPrimaryBackgroundColor`. The first two drive the gradient stripe that
`devcon-global-css` has painted since [#1](../../pull/1), so a colour picker in the Style Book
edits CSS written in the first pull request.

## Build log

One pull request per step. Each is small enough to read in a few minutes.

### [#1](../../pull/1) Global CSS client extension

A `globalCSS` CET injecting a stylesheet on every page — a gradient accent stripe, chosen as
a proof that the client-extension pipeline works end to end before anything real depended on
it. The stylesheet is additive only: new selectors, no redefinition of anything the theme
already styles.

### [#2](../../pull/2) Drop unused AI rule mirrors

`blade init` ships the Liferay ruleset five times: the source in `.workspace-rules/` plus
byte-identical copies in `.claude/`, `.cursor/`, `.windsurf/`, and `.gemini/`. Only Claude
Code is used here, so three copies were removed — 85 files. They are a correctness hazard,
not just clutter: `CLAUDE.md` requires re-mirroring on every ruleset change, and there is no
script or CI check enforcing it.

### [#3](../../pull/3) Fix globalCSS asset packaging

The extension from #1 was deployed but had **never worked**. Its zip contained no stylesheet,
so the asset 404'd — while Gradle reported `BUILD SUCCESSFUL` and the CET appeared in the
Control Panel. Cause: the `assemble` block was nested inside the CET entry, where the
workspace plugin does not read it. See [Things that cost us time](#things-that-cost-us-time).

### [#4](../../pull/4) Enable the site API, fix credentials and docs

Enables feature flag `LPD-35443`, which gates every operation in `headless-admin-site`. Also
corrects a wrong password in `CLAUDE.md`, and removes endpoints from
`.workspace-rules/rules/headless-apis.md` that do not exist on this version.

Adds a **Applying a config change** section to `CLAUDE.md`: `configs/<env>/` is an overlay
onto `LIFERAY_HOME`, applied by `initBundle` — **not** on restart, and not by `git pull`.

### [#5](../../pull/5) Site initializer with a Home page

The `siteInitializer` CET. Deploying it creates the DevCon site and a single empty Content
Page. Deliberately minimal — proving the provisioning pipeline before putting content
through it.

Unlike `globalCSS` and `batch`, a `siteInitializer` needs **no `assemble` block**; the plugin
packages its tree into a nested `site-initializer.zip` on its own.

### [#6](../../pull/6) Hero fragment

The first real frontend work: an HTML/CSS fragment with editable regions, placed on the Home
page through `page-definition.json`. Verified rendering for an anonymous visitor, not just
for an admin.

Fragment CSS is **inlined onto the page** at placement time rather than served as a file,
which is why fragments must be self-contained and why editing one does not update pages that
already exist.

### [#7](../../pull/7) Objects and sample data

`Speaker` and `Session`, their relationship, and sample data, as a `batch` CET.

Authored in the Liferay UI and then exported, rather than hand-written. Worth knowing:
definitions and entries export through **different buttons in different places**, and only
one produces batch format.

```
entries      →  {configuration, items} envelope — ready for a batch CET
definitions  →  raw DTO, no envelope — must be wrapped by hand
```

### [#9](../../pull/9) Session list on the Home page

A `session-card` fragment in a new `DevCon Sessions` set, rendered once per record by a
**Collection Display** bound to the `Session` object. Field mappings are declarative —
`ObjectField_title`, `ObjectField_startTime`, `ObjectField_room` map to the fragment's
editable regions — so a content editor can restyle or remap without touching code.

The fragment was authored in the Liferay fragment editor and exported. Fragments, unlike
objects, round-trip byte-for-byte.

**This step ships one line that is not portable.** See
[Object-backed Collections cannot be expressed portably](#object-backed-collections-cannot-be-expressed-portably).

### [#10](../../pull/10) Theme CSS client extension

A `themeCSS` CET carrying the site's brand, plus a frontend token definition that exposes
three colours to the Style Book editor. `devcon-global-css` stopped declaring
`--devcon-accent` itself and now consumes the theme's tokens with a hardcoded fallback.

Built in four moves, each verified before the next:

| | Change | Proved by |
| --- | --- | --- |
| 1 | Empty `src/` | Byte-identical to Classic — plumbing works before anything changes |
| 2 | `$primary: #7b2ff7` | `0b5fff` count → 0, `7b2ff7` → 116 in the compiled `clay.css` |
| 3 | `clay.scss` → custom-properties build | `var(--` count 0 → 9776, purple kept as the fallback |
| 4 | Token definition | Emitted `:root` properties 116 → 3 |

Step 3 is the one that matters. Without it the theme is frozen at build time and a style book
cannot change anything — see
[Design tokens do nothing unless the Clay build reads them](#design-tokens-do-nothing-unless-the-clay-build-reads-them).

Two manual steps survive this PR and cannot be removed: selecting the theme, and re-selecting
it after every redeploy.

## Things that cost us time

Each of these was a real dead end. They are recorded because none of them are documented
where you would look, and several are actively contradicted by the reference cards in
`.workspace-rules/rules/`.

### `BUILD SUCCESSFUL` means "file copied", nothing more

A CET deploy copies a zip into `bundles/osgi/client-extensions/`. The portal processes it
asynchronously afterwards. So a zero exit code tells you nothing about whether the extension
works, whether its assets are present, or whether provisioning succeeded. It does not even
tell you the server was running — **a deploy against a stopped Tomcat is a silent no-op that
looks exactly like success.**

Always verify the artifact and the runtime:

```bash
unzip -l bundles/osgi/client-extensions/<name>.zip     # are the files in there?
curl -s -o /dev/null -w "%{http_code}\n" <asset-url>   # does it serve?
tail -f bundles/tomcat/logs/catalina.out               # did it start?
```

### `assemble` is project scoped, and every CET type differs

`assemble` is a **top-level** key, a sibling of the CET entry — not a child of it. A project
builds one zip which may contain many entries, so packaging is project-level. Nested, it is
silently ignored: no error, no warning, and a zip with no assets.

| Type | Needs `assemble`? |
| --- | --- |
| `globalCSS` | yes |
| `batch` | yes |
| `siteInitializer` | no — packages itself |

Both `theme-and-design/SKILL.md` and `scaffold-client-extension/SKILL.md` document this
incorrectly (one nests it, the other omits it). Liferay's own extensions in
`liferay-portal/workspaces/` put it at the top level.

### A wrong password looks like a broken auth verifier

Liferay forces a password reset at first login, so the template's documented
`test@liferay.com` / `test` is invalid on any bundle anyone has logged into. The symptom is
`403` with an empty `{ }` body on **every** `/o/` endpoint — which reads like an auth
verifier or CORS fault. **Check the credentials before diagnosing anything else.**

### Feature-flagged endpoints fail with `400`, and the spec still lists them

With `LPD-35443` off, every `headless-admin-site` operation returns:

```json
{ "status": "BAD_REQUEST", "type": "UnsupportedOperationException" }
```

`400` normally means *you* sent something malformed, so the instinct is to re-read your
payload. The request is fine; the feature is off.

Worse, `openapi.json` returns `200` either way and advertises the gated operations. **Flag
state is not observable from `/o/api`** — only executing a call proves anything.

### `configs/<env>/` does not reach a running bundle

It is an overlay onto `LIFERAY_HOME`, applied by `initBundle` at bundle creation. Editing it
and restarting does nothing, and neither does `git pull`. See
[How to change things](#how-to-change-things).

### Optional references must be absent, not empty

Liferay's entry export is **not round-trippable**. An unlinked relationship exports as
`"r_..._speakerERC": ""`, and the importer treats an empty string as a lookup key rather than
as absence:

```
NoSuchObjectEntryException: No ObjectEntry exists with the key {externalReferenceCode=, ...}
```

With `importStrategy: ON_ERROR_FAIL`, that one row failed all six sessions —
`processedItemsCount: 0` of `6`. The fix is to omit the key entirely.

### A published object with no `panelCategoryKey` is invisible

Its entry-management screen exists but hangs off no menu, so it cannot be reached in the UI
at all. The object is correct and published; it just has no door. Set
`panelCategoryKey: applications_menu.applications.content`.

### Directory and file naming is inconsistent

In a site initializer, layout directories use **underscores** and their files use **hyphens**:

```
layouts/01_home/page-definition.json
```

### Modules disagree on how to address a site

Neither accepts the other's identifier:

```
headless-admin-site   /sites/{externalReferenceCode}   → DEVCON     (numeric id 404s)
headless-delivery     /sites/{siteId}                  → 20117      (ERC 404s)
```

`headless-admin-site` on this version also has **no** `/sites` collection and no bare
`/sites/{erc}` — every path is a subresource. So `DELETE /sites/{erc}` does not exist, and
the reprovision recipe in `rules/site-initializer-format.md` cannot be followed as written.
Delete the site from the Control Panel instead.

### Object tokens resolve only for object definitions the initializer owns

> **This section previously claimed `OBJECT_DEFINITION_CLASS_NAME` does not exist and that a
> site initializer resolves exactly eight tokens. Both were wrong.** The conclusion — that our
> collection is not portable — happens to be correct, but for a different reason. The wrong
> version was written from an incomplete search; this one is read from the importer's bytecode.

`layouts/01_home/page-definition.json` contains this, and it **will not work on your machine**:

```json
"className": "com.liferay.object.model.ObjectDefinition#A4A0"
```

The `#A4A0` suffix is generated **randomly per object definition**. Measured directly: creating
an object, deleting it, and recreating it with an identical ERC, name, and fields produced
`#U9Q3` and then `#H1B0`. It is not derived from the ERC, the name, or anything you control.

The importer resolves **25 argument-taking tokens**, plus argument-less ones such as
`[$COMPANY_ID$]`, `[$GROUP_ID$]`, `[$GROUP_KEY$]`, `[$GROUP_FRIENDLY_URL$]` and `[$PORTAL_URL$]`:

```
ASSET_LIST_ENTRY_ID   DDM_TEMPLATE_ID              OBJECT_DEFINITION_CLASS_NAME
BLOG_POSTING_ID       DOCUMENT_FILE_ENTRY_ID       OBJECT_DEFINITION_ID
CLASS_NAME_ID         DOCUMENT_FILE_ENTRY_TYPE_ID  OBJECT_DEFINITION_PORTLET_ID
CLIENT_EXTENSION_ENTRY_ERC  DOCUMENT_JSON          RELEASE_INFO
DATA_DEFINITION_ID    DOCUMENT_URL                 ROLE_ID
DDM_STRUCTURE_ID      KEYWORD_ID                   SEGMENTS_ENTRY_ID
                      LAYOUT_ID                    SITE_NAVIGATION_MENU_ITEM_ID
                      LAYOUT_PAGE_TEMPLATE_ENTRY_ID  TAXONOMY_CATEGORY_ID
                      LIST_TYPE_DEFINITION_ID      TAXONOMY_VOCABULARY_ID
                                                   TEMPLATE_ENTRY_ID
```

Read them yourself — this is how the list above was produced:

```bash
unzip -p bundles/osgi/portal/com.liferay.site.initializer.extender.jar \
  '*/BundleSiteInitializer.class' | LC_ALL=C grep -ao "[A-Z][A-Z_]\{4,40\}:" | sort -u
```

**So `[$OBJECT_DEFINITION_CLASS_NAME:Session$]` is a real token — and it still resolves to
nothing here.** `BundleSiteInitializer._addObjectDefinitions` builds the token map from:

```java
_objectDefinitionLocalService.getObjectDefinitions(companyId, true, 0)
                                                              ↑ system
```

**system object definitions only** — plus, separately, each definition the initializer creates
from its own `object-definitions/` folder. `Session` is `"system": false` and is created by
`devcon-batch`, so it is in neither set and the token is never registered. The literal
`[$OBJECT_DEFINITION_CLASS_NAME:Session$]` is then written into the page, the collection binds
to nothing, and the page renders **"No Results Found"** with nothing in any log.

The evidence that the boolean is `system` and not `active`: `Session` is `active: true` and
`approved`, so an `active` filter would have matched and the token would have worked.

**The real fix is to move the object definitions into the initializer's `object-definitions/`
folder**, which makes the token resolve and the collection portable. Both definitions and
entries are company-scoped, so that keeps the property [#7](../../pull/7) wanted — data
surviving site deletion — while removing the hardcoded class name. Not done yet.

**Until then, to make the collection work on your machine**, read your own suffix and paste it in:

```bash
curl -s -u <user>:<pass> \
  "http://localhost:8080/o/object-admin/v1.0/object-definitions?pageSize=50" \
  | grep -o '"className"[^,]*'
```

Then delete the DevCon site and redeploy.

### Liferay returns 404 for content you lack permission to see

Not `403`. A permission denial is indistinguishable from a missing route, a wrong URL, a
disabled feature, or an unpublished template — every one of which looks like `404`.

This cost an afternoon. A display page template was built, published, and marked default, and
every entry URL returned `404`. Three hypotheses were pursued and two were wrong: that the URL
format was wrong (it wasn't), and that company-scoped objects can't have display pages (they
can). The actual cause was that `Guest` had no `VIEW` permission on the entries.

**The diagnostic is one line — compare authenticated against anonymous:**

```bash
curl -s -o /dev/null -w "anon:  %{http_code}\n" "$URL"
curl -s -o /dev/null -w "auth:  %{http_code}\n" -u "$USER:$PASS" "$URL"
```

`404` / `200` means permissions, every time. Run this **before** questioning the URL.

### Object permissions have two layers, granted in different places

Granting `Guest` access to an object in **Control Panel → Roles → Guest → Define Permissions →
Objects** only grants the *definition* layer. Entries stay invisible:

```
GET /o/c/sessions        (anon) -> 200      definition layer: granted
GET /o/c/sessions/34061  (anon) -> 404      entry layer: not granted
totalCount as Guest: 0 of 6
```

The endpoint answers, and reports zero records. A public site with an empty collection.

The entry-layer resource name comes from `ObjectDefinitionImpl.getResourceName()`:

```
com.liferay.object#<objectDefinitionId>
```

Per-entry grants work over REST and are useful for diagnosis, but they don't scale and don't
cover entries created later:

```bash
curl -X PUT -u "$USER:$PASS" -H "Content-Type: application/json" \
  -d '[{"actionIds":["VIEW"],"roleName":"Guest"}]' \
  "http://localhost:8080/o/c/sessions/<id>/permissions"
```

The portable form is `site-initializer/resource-permissions.json`. The importer calls
`setResourcePermissions(companyId, name, scope, primKey, roleId, actionIds)`, so at `scope: 1`
(company) `primKey` must be the company id — `"0"`, which shipped examples use for *portal*
resources, is wrong here.

**DevCon was not publicly visible from [#9](../../pull/9) until this was found.** Every
screenshot until then was taken as an administrator. If you build a public site here, check it
signed out before believing it works.

### `numberOfItems` is mandatory, and omitting it destroys the whole site

With `displayAllItems: true` set, `numberOfItems` looks redundant. The importer reads it
unconditionally:

```
InitializationException: java.lang.NullPointerException:
Cannot invoke "java.lang.Integer.intValue()" because "numberOfItems" is null
```

Site initialization aborts entirely — no site, no pages, no fragments. The deploy still
reports success. **When a site fails to provision, `grep InitializationException` in
`catalina.out` first**; it names the exact field.

### Collections read the search index, not the database

A session existed, was `approved`, and was returned by `/o/c/sessions` — but never appeared
on the page. Elasticsearch confirmed only 5 of 6 documents were indexed, and "Reindex all
search indexes" did not fix it (it produced 4, then 5).

A single `PATCH` to the entry did, immediately:

```bash
curl -u <user>:<pass> -X PATCH ".../o/c/sessions/<id>" \
  -H "Content-Type: application/json" -d '{"room":"Hall A"}'
```

Bulk imports do not reliably trigger the same indexing path as UI writes, so **verify the
front end after any batch import**, not just the row count. A targeted update re-indexes one
document synchronously; a global reindex is asynchronous and can complete partially while
reporting nothing.

### A `themeCSS` CET replaces the theme's stylesheets — it does not add to them

The properties it accepts map one-to-one onto the files `classic-theme.war` ships:

```
clayURL / clayRTLURL   ↔  css/clay.css      css/clay_rtl.css
mainURL / mainRTLURL   ↔  css/main.css      css/main_rtl.css
frontendTokenDefinitionJSON  ↔  WEB-INF/frontend-token-definition.json
```

`ClientExtensionsServicePreAction` calls `themeDisplay.setClayCSSURL(cet.getClayURL())` with
**no blank check and no fallback**. So an omitted `clayURL` does not mean "keep Classic's" — it
means the page gets no Clay stylesheet at all and renders bare. The docs call `clayURL`
required; validation does not enforce it (`Validator.isBlank(x)` jumps past the check), so a
theme missing it builds, deploys, and destroys the site's styling in silence.

The same applies to the RTL pair: set only the LTR URLs and switching the portal to an RTL
language renders unstyled.

You do not have to author any of this. The workspace plugin's `ThemeCSSTypeConfigurer` applies
Liferay's own `BuildThemeTask` and `BuildCSSTask` against the `_styled` / `_unstyled` parents,
so an **empty** `src/` still produces a complete, Classic-equivalent theme. Start there and
confirm the site looks unchanged before writing a line of SCSS.

### YAML keys are passed through verbatim and never validated

The build copies each key into `typeSettings` as a string:

```json
"typeSettings" : [ "clayURL=css/clay.css", "mainURL=css/main.css" ]
```

Nothing checks that list against what the portal reads. `ThemeCSSCETImpl` calls
`getString("mainURL")`, so the `mainUrl` spelling in `skills/theme-and-design/SKILL.md` lands
as a `mainUrl=` entry that nothing ever reads, returns blank, and unstyles the site — with a
clean build and no error at any layer. The same card documents a `clayVersion` property that
exists in neither the portal nor the workspace plugin.

Check the generated config rather than the YAML:

```bash
unzip -p <project>/dist/<name>.zip <name>.client-extension-config.json
```

### Design tokens do nothing unless the Clay build reads them

The parent theme's `clay.scss` is `@import 'clay/base'`, which compiles SCSS variables to
literal values:

```css
.btn-primary { background-color: #7b2ff7; }
```

Classic instead uses the `clay/atlas-custom-properties` variant:

```css
.btn-primary { background-color: var(--btn-primary-background-color, #0b5fff); }
```

Count them: `clay/base` produced **0** `var(--` references, `atlas-custom-properties` produced
**9776**. With the first, a style book, a token definition, and every value in the emitted
`:root` block are inert — the CSS never looks at them.

`atlas-custom-properties.scss` imports its own variables file and **not** `../clay_variables`,
so switching to it naively discards your SCSS. Import both, yours first, so Clay's `!default`
declarations lose:

```scss
// src/css/clay.scss
@import "clay_variables";
@import "clay/atlas-custom-properties";
```

The fallback in `var(--btn-primary-background-color, $primary)` is what makes the two systems
compose instead of compete.

**This variant also brings dark mode**, via `color-scheme: light dark` and `light-dark()`
values. It cannot be switched off — `components/_root.scss` guards the dark block with
`@if (variable-exists(c-dark))` and Clay always defines `$c-dark`. Overriding `$c-root` does not
help, because the dark block is emitted *after* the first `:root`. Beat it on the cascade from
`_clay_custom.scss`, which is imported last:

```scss
:root { color-scheme: only light; }
```

DevCon is deliberately light-only. Dark mode is one deleted rule away, but the fragments
hardcode light backgrounds with inherited text colour and turn unreadable — fix those first.

### Theme selection is manual, and every redeploy loses it

Selecting a theme CSS CET writes a `ClientExtensionEntryRel` row binding the layout set to the
client extension entry. Redeploying the CET replaces that entry and orphans the row, so the
site silently reverts to Classic. **The theme must be re-selected in Site Builder → Pages →
Configuration after every deploy.**

It cannot be declared in the site initializer either. `layout-set/public/metadata.json` accepts
`themeName`, but `BundleSiteInitializer._getThemeId` resolves names through
`ThemeLocalService.getThemes()`, which contains **WAR themes only**. A CET theme is not a Theme
and never appears there. Setting `"themeName": "DevCon Theme Css"` is ignored without any
warning — the site initializes cleanly and comes up on Classic.

This is a known gap, not a misconfiguration:
[I would like the ability to set the Theme CSS Client Extension via Site Initializer](https://discuss.liferay.com/t/i-would-like-the-ability-to-set-the-theme-css-client-extension-via-site-initializer/230)
— open in Product Ideas since December 2025, no official response and no LPS/LPD ticket.

So this is the second place the initializer's "single source of truth" claim has a hole, after
[object-backed Collections](#object-backed-collections-cannot-be-expressed-portably). Both are
documented as procedure rather than papered over.

### Theme CSS is served with no cache headers

Classic's stylesheets are linked with a cache-busting timestamp
(`clay.css?browserId=other&…&t=1739913722000`). A CET's are linked bare:

```
/o/devcon-theme-css/css/clay.css
```

and the response carries no `Cache-Control`, no `ETag`, and no `Last-Modified`. Browsers
heuristically cache it, so after a redeploy you keep seeing the old stylesheet. **Hard-reload
before concluding a theme change did not work** — verify what the server sends, not what the
browser shows:

```bash
curl -s http://localhost:8080/o/devcon-theme-css/css/clay.css | grep -n "color-scheme:"
```

### Display page templates: the UI export is lossy

Exporting one from **Design → Page Templates → Display Page Templates** produces exactly the
initializer's layout, which is convenient:

```
display-page-templates/session/display-page-template.json
display-page-templates/session/page-definition.json
```

Three things the export gets wrong for source use:

1. **`defaultTemplate` is dropped.** You mark it default in the UI; the export omits the flag.
   Import it as-is and the template exists but nothing uses it — which presents as `404` on
   every entry URL, i.e. the failure above wearing a different hat.
2. **`settings` pins the theme** (`"themeName": "Classic"`). Remove it; `themeName` is ignored
   for CET themes anyway.
3. **`contentType.className` is the machine-specific `ObjectDefinition#A4A0`.** The two-part
   `ObjectEntry` + `contentSubtype.subtypeKey` form that shipped Liferay initializers use for
   web content **does not work for objects** — tried, and the template silently did not import.

Object entry URLs use the `/e/` separator, not the `/l/` default returned by
`ObjectEntryDisplayPageFriendlyURLResolver.getDefaultURLSeparator`:

```
/web/devcon/e/session/<displayPageTemplateId>/<entryId>
```

Don't derive it — map a fragment link to the item in the page editor and read the href Liferay
generates. Internally that link is stored as `LayoutPageTemplateEntry_<id>` with
`mapperType: "link"`, visible by exporting the page as a `.lar` (a zip) and decoding
`__editableValues` in `com.liferay.fragment.model.FragmentEntryLink/<id>.xml`.

### The authoritative list of client extension types

Not the reference card — read it from the workspace plugin itself:

```bash
unzip -p ~/.gradle/caches/modules-2/**/com.liferay.gradle.plugins.workspace-*.jar \
  '*client-extension.properties'
```

There is no `objectDefinition` type, despite online guides describing one. The real object
types are `objectAction`, `objectEntryManager`, and `objectValidationRule`.

## How to change things

Three things must be in sync, and `git pull` only moves the first:

```
git        →  source            client-extensions/, configs/
deploy     →  artifact          bundles/osgi/client-extensions/*.zip
provision  →  portal database   the site, its pages, the objects
```

After pulling someone else's change:

| Their diff touched | You run |
| --- | --- |
| A CET's source | `blade gw :client-extensions:<name>:deploy` |
| Initializer tree, and you have **no** DevCon site | `blade gw deploy` — it autoprovisions |
| Initializer tree: **pages, composition, or fragment content** | Delete the site in Control Panel, then deploy |
| `configs/<env>/` | `cp configs/local/portal-ext.properties bundles/portal-ext.properties` then restart |
| Object definitions or data | `blade gw :client-extensions:devcon-batch:deploy` |
| `devcon-theme-css` | Deploy, **re-select the theme**, then hard-reload the browser |

Not sure what a pull changed? `git log --stat HEAD@{1}..HEAD`.

The theme row has two manual steps for a reason: a redeploy orphans the theme binding, and the
CSS is served without cache headers. Neither is avoidable — see
[Theme selection is manual](#theme-selection-is-manual-and-every-redeploy-loses-it) and
[Theme CSS is served with no cache headers](#theme-css-is-served-with-no-cache-headers).

The third row is the expensive one. A site initializer runs **once, at site creation** —
redeploying will not retrofit a fragment or a composition change onto a page that already
exists, because fragment HTML and CSS are copied onto the page when placed. Recreating the
site from source is the only way to apply it, which is safe precisely because the source tree
is the truth. Object data is company scoped and survives.

## Conventions

- One branch and one pull request per change; no direct commits to `main`.
- Commit messages explain **why**, since the diff already shows what. Where a reference card
  was wrong, the commit says so — those notes are the most reusable thing here.
- Verify at runtime, not by exit code. A finding is not real until a `curl` or a log line
  confirms it.
- Reference cards in `.workspace-rules/rules/` have been wrong repeatedly. Treat them as
  hints and check paths, flags, and YAML shapes against the running portal
  (`GET /o/<module>/v1.0/openapi.json`, or `/o/api`) before building on them.
