---
name: mcp-add
description: Add or configure an MCP (Model Context Protocol) server for flockctl — picking the right scope (global/workspace/project), wiring secrets via ${secret:NAME} placeholders, and verifying the reconciled .mcp.json. Use this skill whenever the user asks to add, register, configure, install, wire, enable, or rotate an MCP server, or asks how to store tokens/API keys that an MCP server needs. Keywords, both RU and EN are welcome — mcp, MCP, Model Context Protocol, сервер, добавить mcp, подключить mcp, настроить mcp, токен, secret, секрет, env, переменная окружения, rotate token, api key, github token.
---

# Adding an MCP Server (with secrets)

flockctl lets you declare MCP servers at three scopes — **global**, **workspace**, and **project** — and reference secret values by placeholder so real tokens never land in committed config. This skill is the canonical playbook.

## The big picture

1. Server config (`command`, `args`, `env`) is stored as `.flockctl/mcp/{name}.json` at workspace/project scope, or in `${FLOCKCTL_HOME}/mcp/{name}.json` at global scope. **These files are committed to git.**
2. Secrets live in the flockctl SQLite DB, AES-256-GCM-encrypted with a master key at `${FLOCKCTL_HOME}/secret.key` (mode 600). They never sit in the repo.
3. At reconcile time, flockctl walks the scope chain (project → workspace → global), resolves every `${secret:NAME}` in env values, and writes a ready-to-use `.mcp.json` at the workspace/project root. `.mcp.json` is gitignored.

## Step 1: Decide the scope

| Scope | When to pick it |
|-|-|
| **global** | Tool is useful across every workspace (e.g. `fetch`, `time`). |
| **workspace** | Tool is for everyone on this repo (e.g. `github` with the team's shared repo). |
| **project** | Tool is for a single sub-project in a monorepo, or a config that shouldn't leak to siblings. |

When a server name exists at multiple scopes, the **more specific scope wins**.

## Step 2: Store the secrets the server needs first

Before wiring the server, stash each sensitive value. The UI has a **Secrets** card on every settings page; the API is also fine:

```bash
# Global secret (available everywhere)
curl -X POST http://localhost:8787/secrets/global \
  -H 'content-type: application/json' \
  -d '{"name":"GITHUB_TOKEN","value":"ghp_xxx","description":"GitHub API token for the github MCP server"}'

# Workspace-scoped secret
curl -X POST http://localhost:8787/secrets/workspaces/42 \
  -H 'content-type: application/json' -d '{"name":"GITHUB_TOKEN","value":"ghp_xxx"}'

# Project-scoped secret
curl -X POST http://localhost:8787/secrets/projects/7 \
  -H 'content-type: application/json' -d '{"name":"GITHUB_TOKEN","value":"ghp_xxx"}'
```

Rules for the secret **name**:
- Must match `[A-Za-z_][A-Za-z0-9_]*` (like a shell variable).
- Names are scope-unique — you can have a project `GITHUB_TOKEN` that shadows the global one.
- Reuse names across scopes on purpose — the resolver picks the closest match.

## Step 3: Declare the MCP server

Either use the **Add Server** dialog in the Skills/MCP page, or POST directly:

```bash
curl -X POST http://localhost:8787/mcp/workspaces/42/servers \
  -H 'content-type: application/json' \
  -d '{
    "name": "github",
    "config": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-github"],
      "env": {
        "GITHUB_TOKEN": "${secret:GITHUB_TOKEN}"
      }
    }
  }'
```

**Rules:**
- Never put a raw token, API key, or password in `env`. Use `${secret:NAME}` exclusively.
- `args` can reference placeholders too, but most MCP servers take config via `env`.
- Naming the env key the same as the secret name keeps the config readable.

## Step 4: Verify reconciliation

The POST kicks off an async reconcile. Check the generated file:

```bash
cat <workspace-or-project>/.mcp.json
```

You should see the **resolved** value inline. If you see the placeholder verbatim, the secret is missing at every scope — the server will start with an empty/broken env. The `.flockctl/mcp-state.json` manifest lists servers flockctl knows about.

If Claude Code is running, restart it so it picks up the new `.mcp.json`.

## Anti-patterns to catch

- **Raw token in `env`.** Move it to a secret and replace with `${secret:…}`.
- **Committing `.mcp.json`.** It's gitignored by default; if the user disabled the ignore, revert that — it will leak resolved tokens.
- **Storing a secret at the wrong scope.** If three teammates need different `GITHUB_TOKEN`s, keep them personal — store at workspace scope and let each clone override via a workspace-local secret, not by editing the committed `.flockctl/mcp/github.json`.
- **Relying on `.local.json` overrides for secrets.** That pattern still works for non-sensitive overrides, but secrets belong in the encrypted store.

## Deleting / rotating

### Deleting an MCP server

`.mcp.json` is **a generated artifact, not the source of truth.** Deleting it by hand does nothing: the next reconcile (triggered by any MCP create/update/delete, or at task/agent startup) regenerates it from the per-scope source files. The same goes for the UI — it lists servers from those source files, so a deleted `.mcp.json` won't make a server disappear from the interface.

The source of truth lives in committed JSON files, one per scope:

| Scope | File |
|-|-|
| global | `${FLOCKCTL_HOME}/mcp/{name}.json` (defaults to `~/flockctl/mcp/{name}.json`) |
| workspace | `<workspace>/.flockctl/mcp/{name}.json` |
| project | `<project>/.flockctl/mcp/{name}.json` |

To actually remove a server, do **one** of:

- Call the API (preferred — triggers reconcile, updates `.flockctl/mcp-state.json`):

  ```bash
  # Global
  curl -X DELETE http://localhost:8787/mcp/global/<name>

  # Workspace
  curl -X DELETE http://localhost:8787/mcp/workspaces/<wid>/servers/<name>

  # Project
  curl -X DELETE http://localhost:8787/mcp/workspaces/<wid>/projects/<pid>/servers/<name>
  ```

- Or delete the source JSON file directly from the right scope dir, then trigger a reconcile (any subsequent MCP API call works, or restart the daemon).

Do **not** "fix" stale entries by editing `.mcp.json` — your edit will be overwritten the next time the reconciler runs.

If you want to keep the server defined at one scope but suppress it in a narrower scope, use the **disable** mechanism instead (`PUT /mcp/workspaces/:id/disabled-mcp` or `PUT /mcp/projects/:pid/disabled-mcp` with `{name, level}`) — that records a disable entry in the scope's config so the server is filtered out at reconcile time without removing it from the source scope.

### Rotating / deleting secrets

- Rotating: re-POST to the same endpoint with a new `value`. Upsert replaces the encrypted blob and kicks a reconcile.
- Deleting a secret while an MCP config still references it leaves the placeholder unresolved — a warning is logged and the placeholder is kept verbatim in `.mcp.json`. Fix by removing the reference or recreating the secret.
- Deleting a workspace/project cascades: its scope-bound secrets are wiped from the DB automatically.

## If you get stuck

- Reconcile logs: `console.warn` lines tagged `[mcp-sync]` list unresolved placeholders with the server + env key.
- `GET /secrets/<scope>` never returns the plaintext value — only `name`, `description`, and timestamps. That's by design; to inspect, decrypt via the service layer.
- Master key lives at `${FLOCKCTL_HOME}/secret.key`. Losing it means every stored secret has to be re-entered. Back it up if you care about continuity.
