# Paperclip on Coolify Contract

**Created**: 2026-03-19
**Confidence Score**: 96/100
**Status**: Draft

## Problem Statement

The `birdcar/paperclip-docker` repo builds and publishes a Docker image for Paperclip (AI agent orchestration platform) but provides no Coolify-native deployment configuration. The current deployment at `paperclip.home.birdcar.dev` suffers from two issues: Claude Code cannot authenticate inside the container (it tries to use host credentials that aren't available), and networking between Paperclip and the existing OpenClaw Coolify deployment is broken (OpenClaw's gateway binds to loopback, making it unreachable from other containers).

The goal is to tear down the current deployment and replace it with a properly designed Coolify Docker Compose stack that uses Claude Max subscription auth (`CLAUDE_CODE_OAUTH_TOKEN`), automates first-run setup, and connects to OpenClaw over a shared Docker network.

## Goals

1. **Working Coolify deployment** — Paperclip accessible at `paperclip.home.birdcar.dev` via Coolify's Traefik proxy, with proper health checks and volume persistence.
2. **Claude Max subscription auth** — Claude Code inside the container authenticates via `CLAUDE_CODE_OAUTH_TOKEN` (from `claude setup-token`), billing against the $100/mo Max subscription instead of API pay-as-you-go.
3. **OpenClaw connectivity** — Paperclip can reach OpenClaw's gateway at `http://openclaw:18789` over a shared Docker network, with hooks enabled for wake payloads.
4. **Automated first-run** — Hybrid entrypoint that auto-runs `onboard --yes` on first boot (config, JWT secret, master key) but leaves `bootstrap-ceo` as a manual step (user needs to see the invite URL).

## Success Criteria

- [ ] `docker-compose.yml` deploys successfully in Coolify (standard mode, not raw)
- [ ] Paperclip UI loads at the configured FQDN with authenticated mode
- [ ] Claude Code agents can execute using `CLAUDE_CODE_OAUTH_TOKEN` (no `ANTHROPIC_API_KEY` set)
- [ ] Entrypoint auto-runs onboard on first boot, skips on subsequent boots
- [ ] `bootstrap-ceo` can be run manually via `docker exec` and produces a working invite URL
- [ ] Paperclip can reach OpenClaw gateway over the shared `paperclip-openclaw` network
- [ ] Health check passes and Coolify shows the service as healthy
- [ ] Data persists across container restarts via the `paperclip-data` volume

## Scope Boundaries

### In Scope

- `docker-compose.yml` for Coolify with all env vars, volumes, networks, health check
- Custom entrypoint script that handles first-run onboarding + Claude Code onboarding flag
- Documentation of OpenClaw configuration changes needed (env var changes in Coolify UI)
- Documentation of the `bootstrap-ceo` manual step and `setup-token` workflow

### Out of Scope

- Changes to the Dockerfile itself — the current build is correct and tracks upstream
- Changes to the GitHub Actions workflow — working as designed
- Changes to the OpenClaw container image or compose — only env var changes in Coolify UI
- Upstream Paperclip code changes
- Backup strategy implementation — noted as future consideration
- External Postgres setup — embedded PG is fine for single-operator use

### Future Considerations

- Automated backup of the `paperclip-data` volume
- Migration to external Postgres if embedded PG becomes a bottleneck
- Token rotation automation for `CLAUDE_CODE_OAUTH_TOKEN` (expires after 1 year)
- Additional agent runtime configuration (Codex, Gemini)

## Execution Plan

### Dependency Graph

```
Phase 1: Coolify Deployment Stack (single phase — all work is sequential)
```

### Execution Steps

**Strategy**: Sequential (single phase)

1. **Phase 1** — Coolify Deployment Stack
   ```bash
   /execute-spec docs/ideation/paperclip-coolify/spec.md
   ```

---

_This contract was generated from brain dump input. Review and approve before proceeding to specification._
