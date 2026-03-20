# Implementation Spec: Paperclip on Coolify

**Contract**: ./contract.md
**Estimated Effort**: S

## Technical Approach

Add a Coolify-native Docker Compose file and a custom entrypoint script to the existing `birdcar/paperclip-docker` repo. The compose file uses Coolify's magic variables (`SERVICE_PASSWORD_64_*`, `SERVICE_FQDN_*`) for auto-generated secrets and domain routing. A shell entrypoint wraps the existing Node.js CMD to handle first-run setup: creating the Claude Code onboarding flag and running Paperclip's `onboard --yes` when no config exists yet.

The key auth decision is using `CLAUDE_CODE_OAUTH_TOKEN` instead of `ANTHROPIC_API_KEY`. This env var is checked last in Claude Code's auth precedence chain, so it's critical that `ANTHROPIC_API_KEY` is never set alongside it. The compose file explicitly documents this.

Networking uses a shared external Docker network (`paperclip-openclaw`) that both the Paperclip and OpenClaw Coolify stacks join. OpenClaw requires two env var changes in the Coolify UI (not file changes in this repo): `OPENCLAW_GATEWAY_BIND=lan` and `HOOKS_ENABLED=true`.

## Feedback Strategy

**Inner-loop command**: `docker compose -f docker-compose.yml config` (validates compose syntax and variable interpolation)

**Playground**: Local Docker — build and run the entrypoint to verify first-run behavior.

**Why this approach**: Most changes are config files (YAML, shell script). Compose config validation catches structural errors; manual container run validates the entrypoint logic.

## File Changes

### New Files

| File Path | Purpose |
|-----------|---------|
| `docker-compose.yml` | Coolify-native compose with env vars, volumes, networks, health check |
| `entrypoint.sh` | Hybrid first-run automation: Claude onboarding flag + Paperclip onboard |
| `docs/coolify-setup.md` | Step-by-step Coolify deployment guide including OpenClaw config changes |

### Modified Files

| File Path | Changes |
|-----------|---------|
| `Dockerfile` | Change `CMD` to use entrypoint script; copy `entrypoint.sh` into image |
| `README.md` | Add Coolify deployment section pointing to `docs/coolify-setup.md` |

## Implementation Details

### 1. Entrypoint Script (`entrypoint.sh`)

**Overview**: Shell script that runs before the Paperclip server starts. Handles two first-run concerns: Claude Code's onboarding wizard and Paperclip's own onboarding.

```bash
#!/bin/sh
set -e

CLAUDE_JSON="$HOME/.claude.json"
PAPERCLIP_CONFIG="${PAPERCLIP_CONFIG:-/paperclip/instances/default/config.json}"

# 1. Pre-seed Claude Code onboarding flag (prevents interactive wizard)
if [ ! -f "$CLAUDE_JSON" ]; then
  echo '{"hasCompletedOnboarding":true}' > "$CLAUDE_JSON"
fi

# 2. Run Paperclip onboard if no config exists yet (first boot)
if [ ! -f "$PAPERCLIP_CONFIG" ]; then
  echo "First run detected — running onboard..."
  npx paperclipai onboard --yes
  echo "Onboard complete. Run 'docker exec <container> npx paperclipai auth bootstrap-ceo' to create admin invite."
fi

# 3. Hand off to the original CMD
exec "$@"
```

**Key decisions**:
- Uses `exec "$@"` to replace the shell process with the Node.js server (proper signal handling)
- Checks for `config.json` existence as the first-run sentinel (idempotent)
- Runs as `node` user (inherited from Dockerfile `USER node`)
- Prints a reminder about `bootstrap-ceo` — the one manual step

**Implementation steps**:
1. Create `entrypoint.sh` at repo root
2. Ensure it's executable (`chmod +x`)

### 2. Dockerfile Changes

**Pattern to follow**: Current `Dockerfile` in the repo

**Overview**: Minimal changes — copy the entrypoint script and switch from `CMD` to `ENTRYPOINT` + `CMD` pattern.

```dockerfile
# Add after the VOLUME line, before USER:
COPY --chown=node:node entrypoint.sh /app/entrypoint.sh
RUN chmod +x /app/entrypoint.sh

# Change the CMD line to:
ENTRYPOINT ["/app/entrypoint.sh"]
CMD ["node", "--import", "./server/node_modules/tsx/dist/loader.mjs", "server/dist/index.js"]
```

**Key decisions**:
- `ENTRYPOINT` + `CMD` split allows overriding CMD (e.g., `docker exec ... bash`) while keeping the entrypoint
- Script is copied before `USER node` switch but owned by `node:node`
- Existing `docker run` invocations without explicit command still work (CMD provides default args)

**Implementation steps**:
1. Add `COPY` and `RUN chmod` lines after `VOLUME` declaration
2. Replace `CMD` with `ENTRYPOINT` + `CMD`

### 3. Docker Compose (`docker-compose.yml`)

**Overview**: Coolify-native compose file using magic variables for auto-generated secrets and FQDN routing.

```yaml
services:
  paperclip:
    image: ghcr.io/birdcar/paperclip-docker:latest
    environment:
      # Coolify auto-generated
      - SERVICE_FQDN_PAPERCLIP_3100
      - BETTER_AUTH_SECRET=${SERVICE_PASSWORD_64_AUTHSECRET}
      - PAPERCLIP_PUBLIC_URL=${SERVICE_FQDN_PAPERCLIP_3100}

      # Deployment mode
      - PAPERCLIP_DEPLOYMENT_MODE=authenticated
      - PAPERCLIP_DEPLOYMENT_EXPOSURE=private

      # Claude Code auth (Max subscription — do NOT set ANTHROPIC_API_KEY)
      - CLAUDE_CODE_OAUTH_TOKEN=${CLAUDE_CODE_OAUTH_TOKEN}

      # Server defaults (inherited from image, explicit for clarity)
      - HOST=0.0.0.0
      - PORT=3100
    volumes:
      - paperclip-data:/paperclip
    healthcheck:
      test: ["CMD", "curl", "-f", "http://127.0.0.1:3100/api/health"]
      interval: 30s
      timeout: 10s
      retries: 3
      start_period: 60s
    networks:
      - default
      - openclaw-bridge
    restart: unless-stopped

networks:
  openclaw-bridge:
    external: true
    name: paperclip-openclaw

volumes:
  paperclip-data:
```

**Key decisions**:
- `SERVICE_FQDN_PAPERCLIP_3100` — Coolify auto-generates the FQDN and routes Traefik to port 3100
- `SERVICE_PASSWORD_64_AUTHSECRET` — Coolify auto-generates a 64-char random secret
- `CLAUDE_CODE_OAUTH_TOKEN` is user-supplied (from `claude setup-token` on local machine)
- No `ANTHROPIC_API_KEY` — would override subscription auth and bill via API
- External network `paperclip-openclaw` must be created on the host before first deploy
- Health check uses `/api/health` with a 60s start period (onboard runs during startup)

**Feedback loop**:
- **Playground**: Run `docker compose config` to validate YAML and variable interpolation
- **Experiment**: Deploy in Coolify, check service status shows healthy
- **Check command**: `docker compose -f docker-compose.yml config --quiet && echo "valid"`

### 4. Coolify Setup Guide (`docs/coolify-setup.md`)

**Overview**: Step-by-step guide covering prerequisites, deployment, first-run, OpenClaw integration, and maintenance.

**Implementation steps**:
1. Write the guide covering these sections:
   - Prerequisites (Coolify server, Tailscale, Claude Max subscription)
   - Generate `CLAUDE_CODE_OAUTH_TOKEN` via `claude setup-token`
   - Create shared Docker network on host
   - Deploy in Coolify (Docker Compose resource, standard mode)
   - Set environment variables in Coolify UI
   - First boot (entrypoint auto-onboards, then manual `bootstrap-ceo`)
   - OpenClaw configuration changes (env vars to change in Coolify UI)
   - OpenClaw network attachment (add `paperclip-openclaw` network to OpenClaw stack)
   - Agent configuration in Paperclip UI (claude_local + openclaw_gateway adapters)
   - Maintenance (token rotation, image updates, backups)

**Key decisions**:
- Separate doc file rather than bloating README — this is Coolify-specific operational knowledge
- Includes the exact OpenClaw env var changes needed (not file changes, just Coolify UI settings)
- Documents `bootstrap-ceo` as the one manual step with clear instructions

### 5. README Updates

**Pattern to follow**: Current `README.md` structure

**Overview**: Add a "Deploying on Coolify" section that points to the detailed guide.

Add after the "Getting started" section:

```markdown
## Deploying on Coolify

For a production deployment on [Coolify](https://coolify.io) with Claude Max subscription auth
and OpenClaw integration, see [docs/coolify-setup.md](docs/coolify-setup.md).

The included `docker-compose.yml` is designed for Coolify's Docker Compose deployment mode
and uses Coolify's magic variables for auto-generated secrets and domain routing.
```

**Key decisions**:
- Brief pointer, not a duplication of the guide
- Mentions the two key differentiators (Max auth, OpenClaw) so users know if it's relevant

## Error Handling

| Error Scenario | Handling Strategy |
|----------------|-------------------|
| First-run onboard fails | Entrypoint exits with error (set -e), container restarts, retries onboard |
| `CLAUDE_CODE_OAUTH_TOKEN` not set | Claude Code agents fail at dispatch time with auth error — documented in setup guide |
| `ANTHROPIC_API_KEY` accidentally set alongside OAuth token | API key takes precedence, bills via API — documented as warning in compose and guide |
| Shared network doesn't exist | Compose fails to start — documented as prerequisite with create command |
| OpenClaw gateway unreachable | Paperclip logs adapter error — fix by setting `OPENCLAW_GATEWAY_BIND=lan` in OpenClaw |
| Health check fails during onboard | 60s `start_period` gives onboard time to complete before health checks begin |

## Validation Commands

```bash
# Validate compose syntax
docker compose -f docker-compose.yml config --quiet

# Build image locally with entrypoint changes
docker build -t paperclip:local .

# Test entrypoint first-run behavior (creates config in temp volume)
docker run --rm -v /tmp/paperclip-test:/paperclip paperclip:local

# Verify Claude onboarding flag was created
docker run --rm -v /tmp/paperclip-test:/paperclip paperclip:local cat /paperclip/.claude.json

# Verify compose deploys in Coolify
# (manual — deploy via Coolify UI, check service status)
```

## Rollout Considerations

- **Teardown first**: Delete the existing Paperclip deployment in Coolify before deploying the new compose
- **Network creation**: Run `docker network create paperclip-openclaw` on the host before deploying either stack
- **OpenClaw changes**: Update OpenClaw env vars in Coolify UI (`OPENCLAW_GATEWAY_BIND=lan`, `HOOKS_ENABLED=true`, add `HOOKS_TOKEN`) and redeploy before testing Paperclip→OpenClaw connectivity
- **Token generation**: Run `claude setup-token` on local machine and copy token to Coolify env vars before first deploy
- **Rollback**: If the new deployment has issues, the old image still works with `docker run` — the entrypoint is additive, not breaking

## Open Items

- [ ] Verify the exact health check endpoint path (`/api/health` vs `/healthz` — check upstream Paperclip server routes)
- [ ] Confirm `npx paperclipai onboard --yes` works correctly in the container's non-interactive environment (may need `ONBOARD_ENV_KEYS` set)
- [ ] Test whether Coolify's standard compose mode correctly handles the `external: true` network declaration

---

_This spec is ready for implementation. Follow the patterns and validate at each step._
