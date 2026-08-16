# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Relationship to `.claude/CLAUDE.md`

`.claude/CLAUDE.md` holds the portable Liferay ruleset (skill index, reference cards, agent conventions). It is a **copy** — the source is `.workspace-rules/liferay-rules.md`, mirrored byte-for-byte into `.claude/` (same for `rules/` and `skills/`). Change the ruleset in `.workspace-rules/` and re-mirror; do not edit the copy in place.

The template also ships `.cursor/`, `.windsurf/`, and `.gemini/` mirrors of the same content. They were removed here because this workspace only uses Claude Code, and an unscripted four-way manual mirror drifts silently. To restore one for a teammate on a different tool, copy `.workspace-rules/{rules,skills}` into the target directory and add that tool's entrypoint file (`.mdc` with YAML frontmatter for Cursor, `rules/liferay.md` for Windsurf, `GEMINI.md` for Gemini).

This file holds what is specific to _this_ workspace instance and is not in the template.

## Verified workspace state

| Fact                    | Value                                                                                                                                                                |
| ----------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Product                 | `portal-7.4-ga132` (`gradle.properties`); installed bundle `7.4.3.132-ga132`                                                                                         |
| Workspace Gradle plugin | `com.liferay.gradle.plugins.workspace:17.1.5` (`settings.gradle`)                                                                                                    |
| Gradle wrapper          | 8.9 — always go through `./gradlew` / `blade gw`                                                                                                                     |
| Blade / JDK on PATH     | blade 8.0.2, OpenJDK 21                                                                                                                                              |
| Tomcat dir              | `bundles/tomcat` (**unversioned** — the template's `bundles/tomcat*/` glob still matches, but literal `bundles/tomcat-9.x` paths do not exist)                       |
| HTTP port               | 8080 (`bundles/tomcat/conf/server.xml`)                                                                                                                              |
| Version control         | git — remote `github.com/TamarawGuy/devcon-liferay`, default branch `main`. GitHub flow: one branch per feature, PR review before merge, no direct commits to `main` |
| Feature flags           | `LPD-35443=true` (`configs/local/portal-ext.properties`) — headless-admin-site. `LPD-39244`, `LPD-74328` deliberately off                                            |

7.4 GA is not a quarterly release. Client Extensions, Objects, and Fragments all apply, but any API or feature flag introduced in a `-Qx` line may simply be absent. Verify against the running portal (`GET /o/<module>/v1.0/openapi.json`) before relying on an endpoint from the reference cards.

## Project layout

`modules/` and `themes/` exist but are **empty**, and `client-extensions/` does not exist yet — this is a bare workspace. Anything built here is the first artifact.

`gradle.properties` sets `liferay.workspace.modules.dir=modules`, `liferay.workspace.wars.dir=modules`, `liferay.workspace.themes.dir=themes`. Client extensions are not configured, so the plugin default `client-extensions/` applies: create that directory and the workspace plugin picks up each subproject automatically — no `settings.gradle` edit needed.

`/bundles` and `/gradle-*.properties` are gitignored: `bundles/portal-ext.properties` and `gradle-local.properties` are local-only, while `configs/<env>/portal-ext.properties` is the tracked, promotable copy.

## Commands

```bash
blade gw tasks                        # discover available tasks (dir contents drive the task list)
blade gw deploy                       # build + deploy everything to bundles/osgi/
blade gw :client-extensions:<name>:deploy   # deploy one project — preferred; full deploy rebuilds all
blade gw :modules:<name>:test         # unit tests for one module (no test sources exist yet)
blade gw :modules:<name>:test --tests '*MyTest.testFoo'   # single test method
./gradlew resolve                     # resolve modules against platform.bndrun for the target platform

blade server start --tail             # start Tomcat, tail catalina.out
blade server run                      # foreground; closing the terminal kills the server
blade server start --debug            # JDWP on port 8000
blade server stop

blade gw createDockerContainer && blade gw startDockerContainer   # Docker path (configs/docker)
blade gw distBundleZip                # distributable for the active environment
```

The server is currently up on `http://localhost:8080` (admin: `test@liferay.com` / `123456`).

Liferay forces a password reset at first login, so the template's documented `test` password is never valid after initial setup. A wrong value here surfaces as `403` on **every** `/o/` endpoint with an empty `{ }` body — which reads like an auth verifier or CORS problem and is not one. Check the credentials before diagnosing anything else. `configs/local/portal-ext.properties` now disables the forced reset, but only for bundles initialized after that property was added.

## Applying a config change

`configs/<env>/` is an overlay onto `LIFERAY_HOME` (= `bundles/`). Any path created under it lands at the same path in the bundle:

| Source | Destination |
| --- | --- |
| `configs/common/portal-setup-wizard.properties` | `bundles/portal-setup-wizard.properties` |
| `configs/local/portal-ext.properties` | `bundles/portal-ext.properties` |
| `configs/common/osgi/configs/*.config` | `bundles/osgi/configs/*.config` |

`configs/common` is applied first; the active environment (`liferay.workspace.environment`, default `local`) overrides it on filename collisions.

The overlay is applied by **`initBundle`** — at bundle creation, not on restart. A running bundle does not pick up `configs/` changes, and neither does `git pull`. To apply one to a live bundle, copy the single file across and bounce:

```bash
cp configs/local/portal-ext.properties bundles/portal-ext.properties
blade server stop && blade server start --tail
```

Anyone merging a config PR must do this locally. Verify the property actually reached the JVM's copy (`grep <key> bundles/portal-ext.properties`) before concluding a config change did not work.

## Verifying a deploy

A zero exit code from `blade gw deploy` means the jar was copied, not that anything started. Confirm activation in the logs:

```bash
tail -f bundles/tomcat/logs/catalina.out      # STARTED / Configuration deleted lines
tail -f bundles/logs/liferay.$(date +%F).log  # portal-level errors, stack traces
```

Deployed artifacts land in `bundles/osgi/modules/` (OSGi) and `bundles/osgi/client-extensions/` (CETs) — both currently empty.

Run deploys as background processes so log tailing can happen concurrently, per the agent guidelines in `.claude/CLAUDE.md`.
