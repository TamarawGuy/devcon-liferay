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

> **`bundles/` is gitignored.** It holds the Liferay install and the database, and never
> travels through git. A `git pull` therefore never changes what your running portal serves —
> see [How to change things](#how-to-change-things).

## What is here

```
client-extensions/
  devcon-global-css/     globalCSS   — stylesheet injected on every page
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
  layouts/01_home/
    page.json                            page metadata, incl. Guest VIEW permission
    page-definition.json                 fragment composition
  fragments/group/devcon/
    collection.json
    fragments/hero/                      dir name is the fragment key
      fragment.json                      htmlPath / cssPath / icon / type
      index.html                         editable regions via data-lfr-editable-*
      index.css                          all rules prefixed #wrapper .devcon-hero
```

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

### Object-backed Collections cannot be expressed portably

`layouts/01_home/page-definition.json` contains this, and it **will not work on your
machine**:

```json
"className": "com.liferay.object.model.ObjectDefinition#A4A0"
```

That `#A4A0` suffix is generated **randomly per object definition**. Measured directly:
creating an object, deleting it, and recreating it with an identical ERC, name, and fields
produced `#U9Q3` and then `#H1B0`. It is not derived from the ERC, the name, or anything you
control.

A site initializer resolves exactly **eight** tokens, and none yields a class name:

```
[$COMPANY_ID  [$GROUP_FRIENDLY_URL  [$GROUP_ID    [$GROUP_KEY
[$LAYOUT_ID   [$LIST_TYPE_DEFINITION_ID  [$OBJECT_DEFINITION_ID  [$PORTAL_URL
```

Four source forms were tried. Only the literal works:

| Form | Result |
| --- | --- |
| `[$OBJECT_DEFINITION_CLASS_NAME:Session$]` | dropped — token does not exist |
| `[$OBJECT_DEFINITION_CLASS_NAME:DEVCON_SESSION$]` | dropped — same |
| `className` + `classPK` via `[$OBJECT_DEFINITION_ID:Session$]` | dropped |
| literal `…ObjectDefinition#A4A0` | **works** |

`OBJECT_DEFINITION_CLASS_NAME` appears in **no jar** in the portal. Several sources online
present it as standard; it may exist in a quarterly release, but not in 7.4 GA132.

**To make the collection work on your machine**, read your own suffix and paste it in:

```bash
curl -s -u <user>:<pass> \
  "http://localhost:8080/o/object-admin/v1.0/object-definitions?pageSize=50" \
  | grep -o '"className"[^,]*'
```

Then delete the DevCon site and redeploy. Symptom if you skip this: the page renders with
**"No Results Found"** under the hero, and nothing in any log explains why.

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

Not sure what a pull changed? `git log --stat HEAD@{1}..HEAD`.

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
