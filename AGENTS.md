# Base44 sandbox notes for homehost

Monorepo with four runnable projects under `packages/`. `docker-compose.base44.yml`
is the sandbox runbook; everything runs from the cloned source (bind-mounted) with
its dev server, not from the repo's production Dockerfiles.

| Service | Port | What it is |
| --- | --- | --- |
| `client` | 3000 (preview) | React SPA built by webpack-dev-server |
| `server` | 5000 | Express API + Prisma/SQLite + media file watcher |
| `media-engine` | 5100 | Go data plane (ffprobe scanning, range streaming, HLS) |
| `control-plane` | 5200 | Rails 8 app (passwordless login, jobs, collections) |

Only `client` + `server` form the app the README describes; the client talks to the
Express API on 5000 and never to the Go/Rails services. Those two are wired up so
they boot, and are reachable on their own ports.

## Non-obvious setup details

- **Client API base is compile-time.** `webpack.common.js` feeds `dotenv.parsed`
  into `DefinePlugin`, so `process.env.REACT_APP_*` comes from
  `packages/client/.env` **at compile time**. The compose command generates that
  file from `HOMEHOST_API_BASE` before starting the dev server; changing the API
  URL therefore needs a client restart, not just a reload.
- **API CORS** is the single `Access-Control-Allow-Origin` value
  `CLIENT_BASE_URL` in `server.js`; it must match the preview origin
  (`https://3000-$BASE44_PUBLIC_HOST_SUFFIX`), otherwise every list stays empty.
- **Dev server host check.** webpack-dev-server rejects the preview proxy with
  `Invalid Host header` unless `.${BASE44_SANDBOX_HOST_DOMAIN}` is in
  `allowedHosts`; the compose command passes it via `--allowed-hosts`.
- **Database.** Prisma migrations are applied at container start
  (`prisma migrate deploy`); the SQLite file lives at `packages/server/data/media.db`
  (`DATABASE_URL=file:../data/media.db`) and is gitignored.
- **Media locations.** `MOVIES_PATH` / `TV_PATH` / `MUSIC_PATH` are host directories
  bind-mounted to `/media/{movies,tv,music}` (defaults under
  `/tmp/homehost-media`, override with those env vars). The server's chokidar
  watcher must see three *different* paths; the Go engine reads the same mounts.
- **Empty library.** A library with zero items used to crash every slider
  (`hooks/useSlide.js` read `firstChild.clientWidth`); it is now guarded, so empty
  sections simply render their headings.
- **Metadata needs external keys.** `TMDB_KEY` (movies/TV) and
  `SPOTIFY_CLIENT_ID`/`SPOTIFY_CLIENT_SECRET` (music) are delivered to
  `/run/base44/app.env`. Without a valid TMDb key a scanned file is recorded in
  `NotAvailable` and never appears in the UI, so an empty library with the keys
  missing is expected. File naming (see README): `<title> (<year>) <tmdbId>.mp4`,
  `<show> <tmdbId>/S01E02 name.mkv`, `<artist>/<album>/<disc>-<track> name.mp3`.
- **Pre-existing quirks.** `GET /` is the app's own 404 page (the app lives at
  `/movies`, `/tv`, `/music`). `/music/albums`, `/music/artists` and `/music/songs`
  read `location.state`, so a direct URL/reload of those paths renders blank — they
  only work through the `SEE ALL` links.

## Verify the stack

```sh
docker compose -f docker-compose.base44.yml ps                       # all healthy
curl -s localhost:5000/api/about                                     # {"name":"server"...}
curl -s localhost:5100/health/ready                                  # {"status":"ready"}
curl -s -o /dev/null -w '%{http_code}\n' localhost:5200/login        # 200
curl -s -o /dev/null -w '%{http_code}\n' localhost:3000/movies       # 200
```

Checks run inside the containers (the sandbox host has no runtimes):

```sh
docker compose -f docker-compose.base44.yml exec -T media-engine go test ./...
docker compose -f docker-compose.base44.yml exec -T control-plane bin/rails test
docker compose -f docker-compose.base44.yml exec -T client npx eslint .
```

The React client has no test suite.
