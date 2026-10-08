# OpenDock NAS deployment

How Craig runs on the OpenDock Synology NAS. Each folder here is one Compose
project and lives at `/volume2/docker/<folder>` on the NAS.

| Folder | Host port | Public address | What it is |
|---|---|---|---|
| `craig` | none | | Bot, dashboard, ferret, kitchen, streamer and webapp server in one container, plus Postgres and Redis |
| `craig-dashboard` | 9200 | `craig.gifamily.synology.me` | Relay to the dashboard |
| `craig-dl` | 9100 | `craig-dl.gifamily.synology.me` | Relay to ferret (recording download pages) |
| `craig-ez` | 9001 | `craig-ez.gifamily.synology.me` | Relay to the Ennuizel streamer API (WebSocket) |
| `craig-web` | 9002 | `craig-web.gifamily.synology.me` | Relay to the webapp server (WebSocket) |
| `craig-app` | 9003 | `craig-app.gifamily.synology.me` | Webapp frontend (static site) |
| `craig-edit` | 9004 | `craig-edit.gifamily.synology.me` | Ennuizel editor frontend (static site) |

DSM's reverse proxy terminates TLS and forwards each address to its port.

## Why it is split up

- DSM recognizes one published port per Compose project, so every public port
  of the `craig` container gets a project of its own: a `socat` container that
  relays TCP to `craig` over the `craig_default` network.
- `craig/compose.yaml` is the repository's `docker-compose.yml` rewritten for
  the Compose v2.20 that DSM ships, which does not parse the
  `env_file: - path:` form.
- The webapp and the editor each need a frontend that is not part of this
  repository. `craig-web` and `craig-ez` are only their backends.

## Setting it up

1. Copy the folders to `/volume2/docker/`. Put the repository (with the
   `locale` submodule) in `craig/src`; it is only needed for building.
2. Copy `craig/env.example` to `craig/.env`, `chmod 600` it and fill in the
   empty values. `POSTGRES_PASSWORD`, `JWT_SECRET` and `WEBAPP_TOKEN` are
   random strings (`openssl rand -hex 32`).
3. Build the two frontends and copy each `dist/` to `site/` in its folder. The
   commits in use are in `craig-app/COMMIT` and `craig-edit/COMMIT`.
   - `craig-app`: https://github.com/CraigChat/webapp
   - `craig-edit`: https://github.com/CraigChat/ennuizel, with submodules

   Both build with `yarn install --frozen-lockfile && yarn build`.
4. Allow the Docker subnets in the DSM firewall: `172.16.0.0/12` and
   `192.168.0.0/16`. Without this the relays accept connections but never
   reach `craig`.
5. Start `craig` first, since it creates the network the relays join:

   ```sh
   cd /volume2/docker/craig && sudo docker compose pull && sudo docker compose up -d
   for n in dashboard dl ez web app edit; do (cd /volume2/docker/craig-$n && sudo docker compose up -d); done
   ```

Do not `docker compose down` the `craig` project while the relays are up: it
would try to remove the network they are attached to.

## Settings that differ from the defaults

- `CRAIG_IMAGE_TAG=master` pulls upstream's prebuilt image, which matches this
  fork as long as it has no changes of its own. `sudo docker compose build`
  builds `craig/src` instead.
- `REWARD_TIERS_JSON` gives the default tier every feature, so auto-record,
  mixes and MP3 work without Patreon.
- `WEBAPP_CONNECT_URL` points at `craig-app` and passes
  `custom=craig-web.gifamily.synology.me`, which makes the frontend use this
  server instead of the official ones.
- `PUBLIC_ENNUIZEL_URL` points at `craig-edit`; `PUBLIC_ENNUIZEL_API_HOSTNAME`
  is `craig-ez`.
