# lobby

**One server, many apps, one front door.**

lobby turns a cheap VPS into a home for several apps. One [Traefik](https://traefik.io/) instance takes every request on :443, gets the certificates, and sends each visitor to the right app. Each app lives in its own repo and is deployed, rolled back or removed on its own: apps never depend on each other, and they don't depend on lobby's code. They only rely on the small **contract** below.

lobby is the hosting companion of [product development office](https://github.com/a1drich/product-development-office), the agent swarm that builds and ships a repo end to end. It provides the server half of the office's [Hetzner + Traefik stack](https://github.com/a1drich/product-development-office/blob/main/docs/stack/HETZNER-TRAEFIK.md): the office builds an app and runs its `make release` when you approve, and lobby is where that release lands. You don't need the office to use lobby, and an app on lobby works the same whether a person or the office releases it.

```
                         ┌────────────── lobby (Traefik) ──────────────┐
  https://  ────────────►│  www.app-one.example  → app-one             │
                         │  api.app-one.example  → app-one             │
                         │  app.app-two.example  → app-two             │
                         └───────────────┬─────────────────────────────┘
                                         │ network: proxy
          ┌──────────── app-one ─────────┴──┐   ┌──────────── app-two ────────────┐
          │ /opt/app-one   (own repo)       │   │ /opt/app-two   (own repo)       │
          │ web, api, db  (own network)     │   │ web, worker, db (own network)   │
          │ released with its own key       │   │ released with its own key       │
          └─────────────────────────────────┘   └─────────────────────────────────┘
```

Use it: click **Use this template** on GitHub, or clone it. Everything specific to your server (IPs, domains, secrets) stays on the server, never in the repo.

## The contract

Every app on a lobby server can rely on this, and nothing else:

| What | Value |
|---|---|
| Network for routed services | `proxy` (external) |
| HTTPS entrypoint | `websecure` (:443). Plain HTTP on :80 redirects to it. |
| Certificates | Resolver `le` (Let's Encrypt, TLS-ALPN). On by default for `websecure`, so routers need no TLS labels. |
| Routing | Docker labels. Only containers with `traefik.enable=true` on `proxy` get traffic. |
| Deploy user | `deploy`, in the `docker` group |
| App checkout | `/opt/<app>`, owned by `deploy` |
| Backups | `/var/backups/<app>`, mode 700, owned by `deploy` |
| Release access | A key that may only run `/opt/<app>/deploy/deploy.sh`. Its arguments arrive in `$SSH_ORIGINAL_COMMAND`. |
| Build lock | `flock /var/lock/host-docker-build.lock` around image builds, so two apps never build at once |

In return, every app keeps the house rules, so apps can't collide:

- **Prefix everything with the app's name:** the compose project (`name: <app>`), images (`<app>-<service>`), and Traefik routers, services and middlewares (`<app>-...`).
- **Publish no ports.** Only Traefik binds 80 and 443.
- **Keep databases and caches on the app's own network,** never on `proxy`.
- **Touch only your own project:** `docker compose` against your own project only, prune only your own images.
- **Keep secrets in untracked `.env` files** under `/opt/<app>`.

## What's here

```
lobby                         the command: setup, edge, add-app, list
edge/compose.yml              Traefik
edge/.env.example             ACME_EMAIL (optional)
services/<name>/compose.yml   your server-level services (monitoring, a status page, ...)
templates/app-compose.yml     starting point for an app's production compose file
templates/service-compose.yml starting point for a server-level service
```

```
lobby setup                                   provision the server (safe to rerun on a live one)
lobby edge                                    start or update Traefik
lobby add-app <app> <owner/repo> [--release-key '<public key>']...
lobby list                                    apps checked in, and what's running
```

Everything runs as root on the server. `setup` only installs packages that are missing, so rerunning it never upgrades Docker under running apps.

## Set up a server

1. **Get a server.** Any Ubuntu VPS works. 2 vCPU and 4 GB holds two or three small apps, and image builds are the memory peak. Note its IP.
2. **Provision it** from your clone of lobby:
   ```bash
   ssh root@<ip> 'bash -s' -- setup < lobby
   ```
3. **Put lobby on the server and open the front door:**
   ```bash
   ssh root@<ip>
   git clone https://github.com/<you>/lobby /opt/lobby   # private copy? see below
   ln -s /opt/lobby/lobby /usr/local/bin/lobby
   lobby edge
   ```
   If your copy of lobby is private, check it in like an app instead: `ssh root@<ip> 'bash -s' -- add-app lobby <you>/lobby < lobby`, add the key it prints as a deploy key, then run it again.
4. **Check:** `curl -sI http://<ip>` redirects to https, and `lobby list` shows the edge up.

## Add an app

Say the app is called `shop`, lives in `you/shop`, and serves `www.shop.example` and `api.shop.example`.

1. **DNS first.** Point an A record for every host the app serves at the server's IP, and wait until `dig +short A www.shop.example` returns it. Traefik asks for a certificate the moment a router appears. If DNS isn't live yet, the request fails, and Let's Encrypt rate-limits retries (5 failures per host per hour).
2. **Make a release key** on the machine that will release the app (your laptop, or the machine the office runs on):
   ```bash
   ssh-keygen -t ed25519 -N '' -f ~/.ssh/shop_deploy
   cat >> ~/.ssh/config <<'EOF'

   Host shop-deploy
     HostName <ip>
     User deploy
     IdentityFile ~/.ssh/shop_deploy
     IdentitiesOnly yes
   EOF
   ```
3. **Check the app in:**
   ```bash
   ssh root@<ip> "lobby add-app shop you/shop --release-key '$(cat ~/.ssh/shop_deploy.pub)'"
   ```
   The first run prints a deploy key. Add it to the repo read-only (`gh repo deploy-key add key.pub -R you/shop`), then run the same command again. It clones the repo into `/opt/shop`.
4. **Give the app's repo a production setup.** Each app owns its own:
   - `deploy/compose.yml`: start from [`templates/app-compose.yml`](templates/app-compose.yml) and keep the house rules.
   - `deploy/deploy.sh`: runs as `deploy` when the release key calls it, and reads `deploy <tag>` or `status` from `$SSH_ORIGINAL_COMMAND`. It should:
     - check out a tag that's on `main`
     - build under the build lock
     - migrate
     - `docker compose up -d --remove-orphans`
     - wait for health checks
     - prune only its own old images

     It must exit nonzero on failure.
   - `make release`: tags `main` and runs `ssh shop-deploy deploy <tag>`.
   - `make rollback REF=<commit>`: redeploys the release tag on that commit. These two are what the office calls.
   - A runbook listing its `.env` files and DNS records.
5. **Secrets.** As `deploy`, write the app's `.env` files under `/opt/shop`, mode 600.
6. **First release:** `ssh shop-deploy deploy v0.1.0`, or `make release` from the app's repo. Traefik picks up the new routers by itself, with no restart.
7. **Check:** `curl -sI https://www.shop.example` returns 200, and `docker logs traefik 2>&1 | grep -i acme` shows no errors.

## Add a server-level service

For something that belongs to the server rather than to one app, such as uptime monitoring, a status page or a shared tool:

1. Copy [`templates/service-compose.yml`](templates/service-compose.yml) to `services/<name>/compose.yml`, set `name: <name>`, pin the image version and set its host.
2. Point DNS for that host at the server.
3. Commit and push. Then on the server: `cd /opt/lobby && git pull && docker compose -f services/<name>/compose.yml up -d`.

## Update the edge

Change `edge/compose.yml`, push, then on the server run `cd /opt/lobby && git pull && lobby edge`. Recreating Traefik drops connections to every app for a few seconds, so pick a quiet moment. Certificates survive in the `traefik_letsencrypt` volume. Keep Traefik at 3.6 or later: older releases can't talk to Docker 29.

## Remove an app

1. Run `docker compose -f /opt/<app>/deploy/compose.yml down`. Add `-v` only once its data is backed up and no longer needed.
2. Remove its line from `~deploy/.ssh/authorized_keys`, plus `~deploy/.ssh/config.d/<app>` and `~deploy/.ssh/github-<app>*`. Delete the deploy key on GitHub.
3. Remove `/opt/<app>`. Keep `/var/backups/<app>` until you're sure.
4. Remove its DNS records.

## Before adding anything

Check headroom with `free -m` and `docker stats --no-stream`. Keep at least 1.5 GB available with everything running, because image builds need it. If there's less, move to a bigger server first.

## Coming from one app's own Traefik

If Traefik currently runs from one app's repo, move it to lobby so that app stops being the host:

1. `lobby setup`, then clone lobby to `/opt/lobby`. It reuses an existing `proxy` network and `traefik_letsencrypt` volume, so certificates carry over.
2. `docker rm -f traefik && lobby edge`. Sites are down for a few seconds.
3. Remove the Traefik stack from the app's repo, and point its runbook at this contract.
