# Deploy Bifrost on Heroku (buildpacks)

This repo is set up for **classic Heroku buildpacks** (not the container stack).

## Buildpack chain (required)

You need **all four** buildpacks. If Heroku only detects Node.js, the Go binary never builds.

Declared in `.buildpacks` / `app.json`:

1. **heroku-buildpack-apt** — CGO deps (`libsqlite3-dev`, gcc, …)
2. **heroku-buildpack-nodejs** — builds `ui/` into `transports/bifrost-http/ui` (for `go:embed`)
3. **subdir-heroku-buildpack** — promotes `transports/` (`PROJECT_PATH=transports`) so `go.mod` is at the build root
4. **heroku-buildpack-go** — builds `./bifrost-http` with `CGO_ENABLED=1`

## One-time setup

```bash
heroku create your-bifrost-app
heroku buildpacks:clear
heroku buildpacks:add https://github.com/heroku/heroku-buildpack-apt
heroku buildpacks:add heroku/nodejs
heroku buildpacks:add https://github.com/timanovsky/subdir-heroku-buildpack
heroku buildpacks:add heroku/go

heroku config:set \
  PROJECT_PATH=transports \
  GO_INSTALL_PACKAGE_SPEC=./bifrost-http \
  CGO_ENABLED=1 \
  NPM_CONFIG_PRODUCTION=false \
  BIFROST_HOST=0.0.0.0 \
  APP_DIR=/app/data \
  LOG_LEVEL=info \
  LOG_STYLE=json

git push heroku dev:main   # or your deploy branch
```

Or use the Deploy button / `app.json` defaults.

### Why `NPM_CONFIG_PRODUCTION=false`?

`vite` and `typescript` live in `ui` **devDependencies**. With `NODE_ENV=production`, `npm ci` skips them and `heroku-postbuild` fails with `vite: not found`. The root script also uses `npm ci --prefix ui --include=dev` as a belt-and-suspenders fix.

## Runtime notes

- The web process listens on Heroku’s **`$PORT`** via `transports/bin/start-bifrost`.
- **`APP_DIR` is ephemeral** on dynos (SQLite under `/app/data` is wiped on restart). For production persistence, point Bifrost at external Postgres/stores and treat local SQLite as disposable, or attach durable storage outside the dyno filesystem.
- Set provider keys as config vars (`OPENAI_API_KEY`, `ANTHROPIC_API_KEY`, etc.) — never commit them.
- Prefer at least a **Basic/Standard-1X** dyno; cold starts and memory matter for the gateway + UI.

## Smoke check

```bash
heroku open
curl "https://your-bifrost-app.herokuapp.com/health"
```
