# scalingo-agent-platform-buildpack

Custom Scalingo-Buildpack fuer das `agent-platform`-Repo (Bun-Backend + Vite-Frontend).

Scalingo unterstuetzt offiziell weder Bun noch Multi-Stage-Dockerfiles. Dieses Buildpack installiert beides (Bun + Node, gepinnt) im Build-Container und baut die Monorepo-App in einem Rutsch:

1. **Frontend** (`frontend/`) wird mit Node 20 LTS + npm gebaut (`npm ci && npm run build`)
2. **Backend** (`backend/`) wird mit Bun 1.x installiert (`bun install --frozen-lockfile --production`)
3. **Bun-Binary** wird in `/app/.bun/bin/` kopiert, ins Runtime-PATH per `.profile.d/`
4. **Postgres-ENV-Aliasing** zwischen Scalingo's `SCALINGO_POSTGRESQL_URL` und unserem App-Code (`SCALINGO_POSTGRES`) plus Drizzle-Tools (`DATABASE_URL`)

## Verwendung im App-Repo

Im Root des `agent-platform`-Repos braucht es:

`.buildpacks` (Multi-Buildpack-Reihenfolge):
```
https://github.com/Scalingo/apt-buildpack.git
https://github.com/tilweb/scalingo-agent-platform-buildpack.git#v0.1.0
```

`Aptfile` (System-Pakete):
```
ffmpeg
ca-certificates
```

`Procfile` (Start-Command, optional — `bin/release` setzt den gleichen Default):
```
web: cd backend && bun run src/index.ts
```

## Update-Pfad

Bun- oder Node-Version aktualisieren:
1. `bin/compile`: `BUN_VERSION` oder `NODE_VERSION` anpassen
2. Commit + Tag (`git tag v0.2.0 && git push --tags`)
3. App-Repo: `.buildpacks`-Zeile auf neuen Tag (`#v0.2.0`)
4. App-Repo deployen — Buildpack-Cache wird automatisch invalidiert wenn die Versionen sich aendern

## Lokal testen

Saubere Snapshot-Kopie des App-Repos via `git archive`, dann in `scalingo/scalingo-22`:

```sh
TMP_BUILD=$(mktemp -d)
git -C /pfad/zu/agent-platform archive HEAD | tar -x -C "$TMP_BUILD"
mkdir -p /tmp/buildpack-test/{cache,env}

docker run --pull always --rm -it \
  --env STACK=scalingo-22 \
  --volume "$(pwd):/buildpack" \
  --volume "$TMP_BUILD:/build" \
  --volume /tmp/buildpack-test/cache:/tmp/cache \
  --volume /tmp/buildpack-test/env:/tmp/env \
  scalingo/scalingo-22:latest bash

# Im Container:
/buildpack/bin/detect /build       # → "Agent Platform"
/buildpack/bin/compile /build /tmp/cache /tmp/env
/buildpack/bin/release /build      # → YAML mit web: ...
```

## Lizenz

MIT (oder was auch immer das App-Repo nutzt).
