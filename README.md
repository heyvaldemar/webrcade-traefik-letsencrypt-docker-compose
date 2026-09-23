# WebRcade + Traefik + Let's Encrypt on Docker Compose

[![Deployment Verification](https://github.com/heyvaldemar/webrcade-traefik-letsencrypt-docker-compose/actions/workflows/deployment-verification.yml/badge.svg?branch=main)](https://github.com/heyvaldemar/webrcade-traefik-letsencrypt-docker-compose/actions/workflows/deployment-verification.yml)

This repository deploys [webЯcade](https://github.com/webrcade/webrcade), a feed-driven game frontend that runs entirely in the browser, behind Traefik with automatic Let's Encrypt TLS. The emulators run in the visitor's browser; the server only hands out the application and whatever feeds and files you give it.

## Getting started

```bash
# 1. Clone
git clone https://github.com/heyvaldemar/webrcade-traefik-letsencrypt-docker-compose
cd webrcade-traefik-letsencrypt-docker-compose

# 2. Create the two Docker networks the stack expects
docker network create traefik-network
docker network create webrcade-network

# 3. Copy the environment template and fill in required values
cp .env.example .env
$EDITOR .env
# ^ Required: TRAEFIK_ACME_EMAIL, TRAEFIK_HOSTNAME, TRAEFIK_BASIC_AUTH,
#   WEBRCADE_HOSTNAME. See .env.example for generation commands.

# 4. Deploy
docker compose -f webrcade-traefik-letsencrypt-docker-compose.yml -p webrcade up -d
```

Within a minute or two, `https://${WEBRCADE_HOSTNAME}` serves WebRcade and `https://${TRAEFIK_HOSTNAME}` serves the basic-auth protected Traefik dashboard, both with fresh Let's Encrypt certificates.

### What success looks like

```bash
docker compose -f webrcade-traefik-letsencrypt-docker-compose.yml -p webrcade ps
# Expected: webrcade and traefik both show "(healthy)"

curl -fsS "https://${WEBRCADE_HOSTNAME}/" | grep -o "<title>[^<]*</title>"
# Expected: <title>webЯcade</title>
```

### Common first-deploy issues

- **Cert issuance fails.** DNS hasn't propagated to your server's IP yet, or port 80/443 isn't reachable from the internet. Confirm with `dig +short ${WEBRCADE_HOSTNAME}`.
- **`docker compose up` fails with `set in .env`.** A required variable is empty in `.env`; the error names it.
- **Network not found.** Step 2 (the `docker network create` commands) was skipped.
- **The library is empty.** That is expected on a fresh deploy: nothing is in it until you add a feed. See below.

### Apply `.env` or compose-file changes

```bash
docker compose -f webrcade-traefik-letsencrypt-docker-compose.yml -p webrcade up -d --force-recreate
```

## Your library

WebRcade is driven by feeds: JSON files that list what to show and where each item's files are. A feed can point anywhere the browser can reach, or at files this server hosts. Anything you want served from here goes into the `webrcade-data` volume, which WebRcade publishes under `/content/`:

```bash
docker compose -f webrcade-traefik-letsencrypt-docker-compose.yml -p webrcade \
  cp ./my-feed.json webrcade:/var/www/html/content/
# then load https://${WEBRCADE_HOSTNAME}/content/my-feed.json as a feed in WebRcade
```

The upstream [feed format](https://docs.webrcade.com/) describes what a feed may contain.

## What this repository does not contain

No ROM, BIOS, firmware or other copyrighted game file is included, linked to, or described how to obtain. WebRcade plays a library you already own: use dumps of cartridges and discs you have, and check that doing so is lawful where you live. This repository deploys the upstream [webrcade/webrcade](https://github.com/webrcade/webrcade) image (Apache-2.0) unmodified; the image is not made or maintained here, so verify its contents and licensing for your own use.

## Supply chain trust

This repository is a deployment template, not a custom image. It orchestrates two upstream images:

- [`webrcade/webrcade`](https://hub.docker.com/r/webrcade/webrcade): WebRcade upstream
- [`traefik`](https://hub.docker.com/_/traefik): reverse proxy, Docker Hub official image

Both are pinned to `tag@sha256:<digest>` as interpolation defaults in the compose file's `x-images` block. Compose pulls by digest, not by tag, so two users deploying on different days get byte-identical image manifests. And `git pull` alone delivers the version combination this repository has tested. Setting `WEBRCADE_IMAGE_TAG` or `TRAEFIK_IMAGE_TAG` in `.env` overrides the default when you deliberately want a different version.

Two override levels exist per image. `<PREFIX>_IMAGE_VERSION` in `.env` swaps only the version of that image (Compose then pulls the tag, without a digest) and leaves every other pin as tested; `<PREFIX>_IMAGE_TAG` replaces the whole reference, digest included. The variable names are listed in `.env.example`. Nested defaults need Docker Compose v2.5 or newer (2022).

The daily Pin Freshness workflow re-resolves each pinned tag against its registry and compares the pinned WebRcade version against the newest Docker Hub tag and the pinned Traefik version against the latest upstream release. Any drift fails that run and notifies the maintainer. GitHub Actions are pinned by commit SHA with version comments; Dependabot keeps those fresh.

## Production checklist

- [ ] **Generate your own `TRAEFIK_BASIC_AUTH` hash**: never deploy the example value from a guide.
- [ ] **Verify Let's Encrypt cert issuance.** Watch `docker compose -p webrcade logs traefik -f` on first start for `Adding certificate for domain(s)`.
- [ ] **Lock down the Traefik dashboard.** Basic auth is basic. Consider Traefik's `IPAllowList` middleware or not exposing the dashboard publicly at all.
- [ ] **Decide who may reach it.** Anything in `/content/` is served to anyone who has the hostname. If your feeds point at files here, keep the site behind a VPN or an authenticating middleware rather than on the open internet.

## Unattended updates

Releases are the update channel: a tag is cut only after CI has booted the pinned images, upgraded from the previous release on the same volumes, and passed the smoke tests. `update.sh` moves a deployment to the newest tag and nothing else:

```bash
./update.sh --dry-run   # show what would be applied
./update.sh             # update within the current major and redeploy
```

Put it on a timer for hands-off minor/patch updates:

```bash
# crontab -e
17 5 * * *  /opt/webrcade-traefik-letsencrypt-docker-compose/update.sh >> /var/log/webrcade-update.log 2>&1
```

The script refuses to cross a MAJOR template version on its own: majors are breaking by definition and their release notes exist to be read. After reading them, `./update.sh --allow-major` performs the jump. It also refuses to touch a checkout with local modifications: your customization belongs in `.env`, which updates never overwrite.

## Resource limits

Every service carries memory and CPU limits plus reservations as compose-level defaults: the same values CI boots the stack under. Override any of them in `.env` (the knobs and their defaults are listed in `.env.example`) and the override survives every `git pull`. If a service is OOM-killed under real load, `docker inspect <container> --format '{{.State.OOMKilled}}'` says so; raise its `_MEMORY_LIMIT` and recreate.

## Backups

The `webrcade-data` volume holds whatever you put in `/content/`. If those files exist only there, copy the volume out:

```bash
docker run --rm -v webrcade_webrcade-data:/data -v "$PWD":/backup alpine \
  tar -czf /backup/webrcade-content.tar.gz -C /data .
```

And to put it back, replacing what the volume holds now:

```bash
docker run --rm -v webrcade_webrcade-data:/data -v "$PWD":/backup alpine \
  sh -c 'find /data -mindepth 1 -delete && tar -xzf /backup/webrcade-content.tar.gz -C /data'
```

CI runs both commands exactly as written here, on every push: a file is served, backed up, deleted, restored, and served again.

Traefik's certificates live in the `traefik-certificates` volume and are re-issued automatically.

## Container hardening

Every service runs with `security_opt: no-new-privileges:true`, so a process cannot gain privileges through setuid binaries even if it escapes its initial capability set. The reverse proxy runs with `cap_drop: [ALL]` and adds back only `NET_BIND_SERVICE` to bind :80/:443. The application container keeps the default capability set on purpose: upstream images assume it, and a wrong guess there is a boot loop in production rather than a hardening win. CI boots the stack under exactly these settings on every push, so what ships is what was tested.

## Testing

The [Deployment Verification](https://github.com/heyvaldemar/webrcade-traefik-letsencrypt-docker-compose/actions/workflows/deployment-verification.yml?query=branch%3Amain) workflow runs on every push, pull request, and every day at 06:00 UTC: shellcheck and actionlint, a Trivy scan of each pinned image, and a deploy-and-test job that first starts the previous release on the same volumes, then upgrades it, requests real routing through Traefik, and requires the page answering over HTTPS to be WebRcade's rather than merely a 200. Pin freshness is its own daily workflow, so this badge says whether the stack works, not whether a pin is one version behind.

---

## About the maintainer

<div align="center">

**Maintained by [Vladimir Mikhalev](https://github.com/heyvaldemar)** · Docker Captain · IBM Champion · AWS Community Builder

[YouTube](https://www.youtube.com/channel/UCf85kQ0u1sYTTTyKVpxrlyQ?sub_confirmation=1) · [Blog](https://heyvaldemar.com) · [LinkedIn](https://www.linkedin.com/in/heyvaldemar/)

</div>
