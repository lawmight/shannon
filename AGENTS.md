## Cursor Cloud specific instructions

### Project Overview

Shannon is an AI-powered penetration testing framework using TypeScript, Temporal.io for workflow orchestration, and Docker for containerization. See `CLAUDE.md` for full architecture details and code style guidelines.

### Services

| Service | How to run | Notes |
|---|---|---|
| **Temporal Server** | `docker compose up temporal -d` | Required. Exposes gRPC on `:7233`, Web UI on `:8233` |
| **Worker** (local dev) | `node dist/temporal/worker.js` | Connects to `localhost:7233` by default. Set `TEMPORAL_ADDRESS` to override |
| **Client** | `node dist/temporal/client.js <url> <repoPath> [options]` | Submits workflow to Temporal |
| **Router** (optional) | `docker compose --profile router up router -d` | Multi-model proxy on `:3456`. Required when using `ROUTER=true` with OpenRouter/OpenAI keys |

### Build

Two packages must be built in order — mcp-server first, then root:

```
cd mcp-server && npm run build && cd .. && npm run build
```

### Lint / Type-check

No ESLint or test framework is configured. TypeScript strict compilation (`npm run build` / `tsc`) is the primary correctness check.

### Docker in Cloud Agent VM

Docker must be installed and configured before Temporal can run. Key gotchas:
- Use `fuse-overlayfs` storage driver (`/etc/docker/daemon.json`)
- Switch to `iptables-legacy` (`update-alternatives --set iptables /usr/sbin/iptables-legacy`)
- Start dockerd manually: `sudo dockerd &>/dev/null &`
- Grant socket access: `sudo chmod 666 /var/run/docker.sock`

### Running a Pipeline

Supports two auth modes (see `.env.example`):
1. **Direct Anthropic**: Set `ANTHROPIC_API_KEY` in `.env`
2. **Router mode**: Set `OPENROUTER_API_KEY` (or `OPENAI_API_KEY`) + `ROUTER_DEFAULT` in `.env`, then pass `ROUTER=true` to the CLI

The full Docker-based workflow is:

```
./shannon start URL=<url> REPO=<repo-name>                # Direct Anthropic
./shannon start URL=<url> REPO=<repo-name> ROUTER=true     # Router mode (OpenRouter/OpenAI)
```

For local development (worker outside Docker), start Temporal via Docker Compose and run the worker directly with Node.js. The `.env` file is loaded automatically via `dotenv`.

### Key Caveats

- The `shannon` CLI script expects the worker to run inside Docker. For local dev, run `node dist/temporal/worker.js` directly instead.
- The `repos/` directory must exist for the worker to start workflows. Create it with `mkdir -p repos`.
- There is no hot-reload — rebuild TypeScript after code changes.
