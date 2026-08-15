# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Relationship to `.claude/CLAUDE.md`

`.claude/CLAUDE.md` holds the portable Liferay ruleset (skill index, reference cards, agent conventions). It is a **copy** — the source is `.workspace-rules/liferay-rules.md`, mirrored byte-for-byte into `.claude/`, `.cursor/`, `.windsurf/`, and `.gemini/` (same for `rules/` and `skills/`). Change the ruleset in `.workspace-rules/` and re-mirror to all four; do not edit one copy in place.

This file holds what is specific to *this* workspace instance and is not in the template.

## Verified workspace state

| Fact | Value |
| --- | --- |
| Product | `portal-7.4-ga132` (`gradle.properties`); installed bundle `7.4.3.132-ga132` |
| Workspace Gradle plugin | `com.liferay.gradle.plugins.workspace:17.1.5` (`settings.gradle`) |
| Gradle wrapper | 8.9 — always go through `./gradlew` / `blade gw` |
| Blade / JDK on PATH | blade 8.0.2, OpenJDK 21 |
| Tomcat dir | `bundles/tomcat` (**unversioned** — the template's `bundles/tomcat*/` glob still matches, but literal `bundles/tomcat-9.x` paths do not exist) |
| HTTP port | 8080 (`bundles/tomcat/conf/server.xml`) |
| Version control | **not a git repo** — no diffs, no history, nothing to compare against |

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

The server is currently up on `http://localhost:8080` (admin: `test@liferay.com` / `test`).

## Verifying a deploy

A zero exit code from `blade gw deploy` means the jar was copied, not that anything started. Confirm activation in the logs:

```bash
tail -f bundles/tomcat/logs/catalina.out      # STARTED / Configuration deleted lines
tail -f bundles/logs/liferay.$(date +%F).log  # portal-level errors, stack traces
```

Deployed artifacts land in `bundles/osgi/modules/` (OSGi) and `bundles/osgi/client-extensions/` (CETs) — both currently empty.

Run deploys as background processes so log tailing can happen concurrently, per the agent guidelines in `.claude/CLAUDE.md`.
