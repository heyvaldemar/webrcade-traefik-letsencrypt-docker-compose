# Changelog

All notable changes to this project are documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.1.0/),
and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

## [Unreleased]

_(no unreleased changes yet)_

## [1.1.2] - 2026-10-09

### Security

- **`traefik:3.7` was rebuilt upstream**; the pin moved from `sha256:b588cb566045…` to `sha256:575fa15b1350…`. Same version, same tag, a rebuilt base image — the usual shape of a security fix in a base layer.

## [1.1.1] - 2026-10-07

### Security

- **`traefik:3.7` was rebuilt upstream**; the pin moved from `sha256:24841fe2de73…` to `sha256:b588cb566045…`. Same version, same tag, a rebuilt base image — the usual shape of a security fix in a base layer.

### Fixed

- **`update.sh` no longer stops without a word when a release adds a variable and no compose file requires one.** The search for `${VAR:?}` came back empty, and under `pipefail` that empty result ended the script with status 1 right after it listed the new variables.
- **`update.sh` stops on a `.env` it cannot read, before the checkout.** It used to fall through: every new required variable read as "not set", or, with none, the tree moved to the new tag and `docker compose up` failed on the permission. Now it names the file, its owner and mode, and changes nothing.

## [1.1.0] - 2026-09-26

### Added

- **Traefik's timeouts on the HTTPS entry point can be set from `.env`.**
  `TRAEFIK_READ_TIMEOUT`, `TRAEFIK_WRITE_TIMEOUT` and `TRAEFIK_IDLE_TIMEOUT`
  default to Traefik's own values (60s, 0s, 180s), so nothing changes unless
  you set them. Traefik reads its static configuration from one source, here
  the command in the compose file, and an override file can only replace that
  command whole; a variable is the way to tune it and keep taking updates.
  The same change was asked for in the [Keycloak template](https://github.com/heyvaldemar/keycloak-traefik-letsencrypt-docker-compose), and every template in the fleet gets it at once.

## [1.0.1] - 2026-09-23

### Fixed

- **The README's backup command had never been run, and there was no way back.** It documented how to copy the content volume out and not how to put it in. The README now carries the restore command too, and CI reads both out of the README and runs them as written on every push: a served file is backed up, deleted, restored and served again.

## [1.0.0] - 2026-09-23

### Added

- **Published, at the fleet's standard.** This repository was private while a
  committed `.env` sat in it; that file has been removed from every commit
  before publication, and a full-history secret scan finds nothing.
- **Pinned images.** `webrcade/webrcade` and `traefik` are pinned by digest in
  the compose file's `x-images` block, so `git pull` alone delivers the tested
  combination. The previous version took every image reference from `.env`,
  and `.env.example` left them empty while saying they were pinned in the
  compose file: a deployment made the documented way could not start.
- **Verification** on every push, pull request and every day: lint, a Trivy
  scan of each pinned image, and a deploy that first starts the previous
  release on the same volumes, upgrades it, and requires the page answering
  through Traefik to be WebRcade's rather than any 200.
- **Pin Freshness**, its own daily workflow, compares each pin with its
  registry and the WebRcade version with the newest Docker Hub tag.
- **`./update.sh`** moves a deployment between release tags, refuses a major
  version unattended and names any newly required variable first.
- **Both services restart on their own** after a reboot or a crash.
- **A health check that asks for a page** (`curl` against the web server)
  instead of only testing that port 80 is open.
- **Resource limits** and `no-new-privileges` on every service, Traefik with
  every capability dropped but `NET_BIND_SERVICE`.
- **What the repository does not contain** is stated in the README: no ROM,
  BIOS or firmware, and no pointer to any.
