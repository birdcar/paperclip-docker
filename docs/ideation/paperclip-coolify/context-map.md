# Context Map: paperclip-coolify

**Phase**: 1
**Scout Confidence**: 88/100
**Verdict**: GO

## Dimensions

| Dimension | Score | Notes |
|---|---|---|
| Scope clarity | 19/20 | All 5 file changes fully specified with exact content in spec. Minor open item: health check endpoint path `/api/health` vs `/healthz` needs verification against upstream. |
| Pattern familiarity | 18/20 | Both pattern files read in full. Dockerfile is 41 lines; README uses ATX headings, fenced code blocks, and a table. Conventions are clear. |
| Dependency awareness | 18/20 | Dockerfile is referenced by README.md, the CI build workflow (`.github/workflows/build.yml`), and the ideation docs. No other consumers exist — this is a standalone Docker packaging repo. |
| Edge case coverage | 16/20 | Spec documents 6 error scenarios explicitly. Open items in spec flag two unverified behaviors: non-interactive `onboard --yes` and Coolify handling of `external: true` networks. |
| Test strategy | 17/20 | No automated test suite exists (expected for a Docker packaging repo). Validation is Docker-native: `docker compose config`, `docker build`, `docker run` with volume mount. Commands fully specified in spec. |

## Key Patterns

- `Dockerfile` — Multi-stage build (`base` → `build` → `production`). Final stage uses `node:lts-trixie-slim`. Key sequence: `COPY --chown=node:node`, global npm installs, `ENV` block, `VOLUME`, `EXPOSE`, `USER node`, then `CMD`. The `COPY` and `RUN` for `entrypoint.sh` must go after `VOLUME` and before `USER node`.

- `README.md` — Structure: H1 title, single-sentence description, H2 sections in order: "What's in the image", "Getting started", "Configuration", "Volumes", "Tags and updates", "Building locally", "Relationship to upstream". New "Deploying on Coolify" section belongs after "Getting started".

## Dependencies

- `Dockerfile` — consumed by → `.github/workflows/build.yml`, `README.md` (build instructions)
- `README.md` — consumed by → GitHub repository homepage display; no programmatic consumers

No external consumers — changes are self-contained within a Docker packaging repo.

## Conventions

- **Naming**: Files at repo root are lowercase. Docs under `docs/`.
- **Shell style**: POSIX `#!/bin/sh` with `set -e`. No bashisms.
- **Dockerfile**: `COPY --chown=node:node` pattern. JSON array exec form for `CMD`. `ENV` block with `\` continuation.
- **Docker Compose**: Spec uses list form (`- KEY=value`) for Coolify variable interpolation.
- **Documentation**: Prose-heavy, no emoji, backtick-fenced code, `**bold**` for key terms. Tables use `|---|---|`.

## Risks

- **Health check endpoint uncertainty** — `/api/health` vs `/healthz` unverified against upstream.
- **Non-interactive `onboard --yes`** — May need `ONBOARD_ENV_KEYS` set. `set -e` ensures exit rather than hang.
- **`USER node` placement** — `COPY` and `RUN chmod` must go before `USER node` line.
- **No automated tests** — All validation is manual Docker commands.
