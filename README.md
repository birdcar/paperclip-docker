# paperclip-docker

Unofficial Docker image for [paperclipai/paperclip](https://github.com/paperclipai/paperclip), an open-source AI agent orchestration platform. The upstream project ships no official container image, so this repo builds one and publishes it to GHCR on a schedule that tracks upstream releases.

## What's in the image

The image clones and builds Paperclip from source at the latest upstream tag, then installs the three CLI agent runtimes Paperclip can invoke:

- `@anthropic-ai/claude-code`
- `@openai/codex`
- `opencode-ai`

It runs the Paperclip server with its bundled React UI on port `3100`, and stores all persistent data (config, database, secrets) under `/paperclip`.

## Getting started

```sh
docker run -d \
  --name paperclip \
  -p 3100:3100 \
  -v paperclip-data:/paperclip \
  ghcr.io/birdcar/paperclip-docker:latest
```

Then open `http://localhost:3100`.

A minimal Compose setup:

```yaml
services:
  paperclip:
    image: ghcr.io/birdcar/paperclip-docker:latest
    ports:
      - "3100:3100"
    volumes:
      - paperclip-data:/paperclip
    environment:
      PAPERCLIP_DEPLOYMENT_MODE: authenticated
      PAPERCLIP_DEPLOYMENT_EXPOSURE: private

volumes:
  paperclip-data:
```

## Deploying on Coolify

For a production deployment on [Coolify](https://coolify.io) with Claude Max subscription auth
and OpenClaw integration, see [docs/coolify-setup.md](docs/coolify-setup.md).

The included `docker-compose.yml` is designed for Coolify's Docker Compose deployment mode
and uses Coolify's magic variables for auto-generated secrets and domain routing.

## Configuration

All configuration is done through environment variables. The image sets a handful of defaults that make sense for a containerized deployment; override them as needed.

| Variable | Default in image | Description |
|---|---|---|
| `HOST` | `0.0.0.0` | Bind address |
| `PORT` | `3100` | HTTP port |
| `SERVE_UI` | `true` | Serve the React frontend |
| `PAPERCLIP_HOME` | `/paperclip` | Root for all persistent data |
| `PAPERCLIP_INSTANCE_ID` | `default` | Instance identifier |
| `PAPERCLIP_CONFIG` | `/paperclip/instances/default/config.json` | Path to instance config file |
| `PAPERCLIP_DEPLOYMENT_MODE` | `authenticated` | `local_trusted` or `authenticated` |
| `PAPERCLIP_DEPLOYMENT_EXPOSURE` | `private` | `private` or `public` |

**`PAPERCLIP_DEPLOYMENT_MODE`** controls auth behavior. `local_trusted` skips authentication entirely — only use it on a machine that's already access-controlled. `authenticated` requires users to log in and is the right default for anything reachable over a network.

**`PAPERCLIP_DEPLOYMENT_EXPOSURE`** tells Paperclip whether to expect traffic from the public internet. `private` is appropriate for local or VPN-only deployments; `public` enables additional hardening and is required if you're exposing the instance to the internet.

Beyond these, Paperclip supports a number of other variables for external Postgres, S3-compatible storage, secrets management, and heartbeat scheduling. See the [upstream server config source](https://github.com/paperclipai/paperclip/blob/master/server/src/config.ts) for the full list.

## Volumes

Mount `/paperclip` to persist data across container restarts. This directory holds the instance config, the embedded SQLite/Postgres database, encrypted secrets, and file attachments. Losing it means losing your Paperclip state.

## Tags and updates

The build workflow runs every 6 hours and checks the latest tag on the upstream repo. When a new tag appears that hasn't been built yet, it builds and pushes two tags:

- `ghcr.io/birdcar/paperclip-docker:latest`
- `ghcr.io/birdcar/paperclip-docker:<upstream-version>` (e.g. `v2026.318.0`)

The version-pinned tags are immutable once built. The last built version is tracked in [`.last-built-tag`](./.last-built-tag).

You can also trigger a manual build from the Actions tab if you need to force a rebuild.

## Building locally

```sh
docker build \
  --build-arg PAPERCLIP_REF=master \
  -t paperclip:local \
  .
```

`PAPERCLIP_REF` accepts any git ref from the upstream repo — a tag, branch name, or commit SHA.

## Relationship to upstream

This repo only contains the `Dockerfile` and the GitHub Actions workflow. All Paperclip application code lives in [paperclipai/paperclip](https://github.com/paperclipai/paperclip). Bugs in Paperclip itself should be reported there; issues specific to the Docker packaging belong here.
