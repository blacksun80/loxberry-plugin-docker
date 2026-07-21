# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

A **LoxBerry plugin** (Plugin Interface V2) that installs the Docker engine and a Portainer container for graphical management, and integrates Portainer into the LoxBerry WebUI. This is a fork of `michaelmiklis/loxberry-plugin-docker`; `origin` is the fork `blacksun80/loxberry-plugin-docker`, `upstream` is the original.

There is **no build/lint/test toolchain**. The plugin is shipped as a ZIP of this repo and installed through the LoxBerry WebUI (*Plugin-Verwaltung → install*). "Running" it means installing it on a LoxBerry system; there is nothing to run locally.

## Packaging & installing

- Package = ZIP the repo contents. **Exclude the `.git` folder** (older test ZIPs dragged the whole `.git/` in, which just bloats the archive). GitHub's "Download ZIP" of the branch works and already excludes `.git`.
- Install/uninstall happens on the LoxBerry box; the result is a log at `/opt/loxberry/log/system/plugininstall/docker.log`. Read that log to verify — it is the primary feedback channel.
- Tested on LoxBerry 4.0 (DietPi/Debian Trixie, x86_64) and intended to also work on LoxBerry 3 (older Debian, **not** Trixie). `LB_MINIMUM=2.0.0.4`, `ARCHITECTURE="raspberry,x86"` (x86 covers x86_64).

## Plugin lifecycle & architecture

The LoxBerry installer drives everything from `plugin.cfg`. Understanding the install flow matters more than any single file:

1. **`plugin.cfg`** — identity + metadata. `NAME=docker` / `FOLDER=docker` are the immutable plugin id (used for install folder and script names); **never change them** or LoxBerry treats it as a different plugin and updates break. `[AUTOUPDATE]` `RELEASECFG`/`PRERELEASECFG` point to the `release.cfg`/`prerelease.cfg` that the auto-updater reads.
2. **`dpkg/apt`** — one apt package per line; installed via `apt-get -y install` **before** postroot. Trap: if *any* listed package name does not exist on the target Debian, the whole apt command aborts and the others are not installed either. Keep this list minimal and universally available across Debian versions (LB3→LB4).
3. **`postroot.sh`** — runs as root *after* file install. This is where Docker itself is installed (via `curl https://get.docker.com | sh`, not via apt) and the Portainer container is created (`-p 9000:9000`). It ends by **verifying** Docker + the Portainer container are up and does `exit 2` on failure. Exit code is the *only* thing that makes LoxBerry mark the install as failed — a failed apt package alone is just a warning, so without this check a broken install still reports "installed".
4. **`uninstall/uninstall`** — extensionless script in the `uninstall/` dir, run as root on uninstall (not on update). Full removal: stops/removes the Portainer container, purges the Docker engine packages, removes `/var/lib/docker`, `/opt/portainer`, the docker apt repo/key and the docker group. It purges only *installed* packages (filtered via `dpkg-query`) so unknown package names on older systems don't abort the purge. LoxBerry sets the executable bit itself on install.

### WebUI

`webfrontend/htmlauth/index.cgi` (Perl) renders `templates/index.html` with the LoxBerry header/footer and language strings from `templates/lang/`. The plugin appears in LoxBerry at `/admin/plugins/docker/`.

`index.html` shows a **button that opens Portainer in a new browser tab** (via `webfrontend/htmlauth/forward.html`, a small JS redirect to `http://<host>:9000`). It intentionally does **not** embed Portainer in an iframe: Portainer running on port 9000 cannot be framed inside the LoxBerry page (browser shows "refused to connect"), and it only speaks HTTP on 9000, so `forward.html` hardcodes the `http:` protocol rather than inheriting the WebUI's (which breaks if the WebUI runs over HTTPS).

## Line endings

`.gitattributes` normalizes everything to **LF** (`eol=lf`). LoxBerry converts to Unix format on install, but keeping LF in the repo prevents Windows editors/GitHub Desktop from flipping shell scripts (`postroot.sh`, `uninstall/uninstall`, `dpkg/apt`) to CRLF, which would break them. After changing `.gitattributes`, run `git add --renormalize .`.

## Known open items (auto-update chain — deferred)

These are real but intentionally left unfixed until the upstream author (michaelmiklis) is contacted (to decide fork-vs-upstream ownership), because the "correct" values depend on which repo hosts releases:

- `release.cfg` `ARCHIVEURL`/`INFOURL` point to the **wrong repo** (`loxberry-plugin-xiaomi-miflora`) — a copy-paste bug.
- Version mismatch: `plugin.cfg` is `2.2.0` but `release.cfg`/`prerelease.cfg` say `2.1.0`.
- `plugin.cfg` `RELEASECFG`/`PRERELEASECFG` point to **upstream** michaelmiklis, not this fork, so fork changes are never delivered by auto-update.
