---
name: agents-md-editing
description: Edit shared agent guidance (AGENTS.md / CLAUDE.md) in a Swarmctl project or workspace — knowing which file is the editable source vs. which is auto-generated, and how the workspace → project cascade works. Use this skill whenever the user asks to update CLAUDE.md, AGENTS.md, project rules, coding conventions, agent instructions, or agent guidelines. Keywords, both RU and EN: AGENTS.md, CLAUDE.md, agent rules, project conventions, системный промпт, инструкции агенту, правила проекта, readme для агента, отредактируй правила, обнови гайд.
---

# Editing AGENTS.md / CLAUDE.md in Swarmctl

In a Swarmctl workspace or project you will see **two** files named `AGENTS.md` — only one of them is editable.

| Path | What it is | Edit it? |
|-|-|-|
| `.flockctl/AGENTS.md` | **Source** — authored by humans, committed to git | Yes |
| `AGENTS.md` at the root | **Generated** — byte-stable output from the reconciler | No |
| `CLAUDE.md` at the root | **Symlink** → `AGENTS.md` | No |

The root `AGENTS.md` and `CLAUDE.md` symlink exist so agents that don't know about Swarmctl (Codex, Aider, Cursor, plain Claude Code) can still discover the same rules. Swarmctl regenerates them whenever a source changes, on project creation, and on server startup.

## Which source to edit

- **Project-specific rule** (e.g. "this service uses Fastify, not Hono") → edit `<project>/.flockctl/AGENTS.md`.
- **Workspace-wide rule** (e.g. "every service here is Drizzle + SQLite") → edit `<workspace>/.flockctl/AGENTS.md`. It cascades into every project's root `AGENTS.md` above the project block.
- **User-level preference** → belongs in the user's own agent memory/config, not in Swarmctl.

When in doubt, pick the narrower scope. A rule pinned at workspace scope affects every project in that workspace.

## How the merge works

For a project inside a workspace, the root `AGENTS.md` is assembled as:

```
<!-- AUTO-GENERATED header (says "do not edit") -->

<!-- BEGIN workspace AGENTS.md (from <workspace>/.flockctl/AGENTS.md) -->
{workspace source content}
<!-- END workspace AGENTS.md -->

<!-- BEGIN project AGENTS.md (from <project>/.flockctl/AGENTS.md) -->
{project source content}
<!-- END project AGENTS.md -->
```

Rules:
- Only the non-empty block(s) are emitted.
- If both sources are empty, the root `AGENTS.md` and the `CLAUDE.md` symlink are removed.
- Workspace root `AGENTS.md` is just its own source wrapped with a similar header — there is no parent to cascade from.
- Output is byte-stable: saving the same content twice doesn't change the file's mtime, so git stays quiet.

## Do / Don't

Do:
- Edit `.flockctl/AGENTS.md` at the right scope (project vs. workspace).
- Commit **both** `.flockctl/AGENTS.md` and the generated root `AGENTS.md` + `CLAUDE.md` symlink to git — teammates without Swarmctl running rely on the generated files.
- Let Swarmctl reconcile on its own; it runs on edit (via the UI or `PUT /projects/:id/agents-md`), on create, and on startup.

Don't:
- Edit the root `AGENTS.md` directly. The next reconcile overwrites your changes without warning.
- Replace `CLAUDE.md` with a regular file. Swarmctl detects that and leaves it alone (with a `[agents-sync]` warning in the log), but the symlink invariant breaks.
- Commit `.flockctl/.agents-reconcile` — it's a gitignored timestamp marker that Swarmctl adds automatically.

## API reference

- `GET /projects/:id/agents-md` → `{ source, effective }` (source = what's in `.flockctl/AGENTS.md`, effective = what's in the root `AGENTS.md` after merge).
- `PUT /projects/:id/agents-md` — body `{ content: string }`. Max 256 KiB. Triggers project reconcile.
- `GET /workspaces/:id/agents-md` → `{ source, effective }`.
- `PUT /workspaces/:id/agents-md` — body `{ content: string }`. Triggers **cascade** reconcile across every project in the workspace.

## If things look stale

- Force a refresh by re-saving the source (UI "Agent documentation" card, or the PUT endpoint with the same content). That fires a reconcile.
- Restarting the server also works — `reconcileAllAgents()` runs at startup and catches drift from `git pull`.
- Check `[agents-sync]` lines in the server log for warnings about non-symlink `CLAUDE.md` files or write failures.
