# Agent MCP Integrations

[![License: MIT](https://img.shields.io/badge/license-MIT-green)](LICENSE)

Setup playbook for the Model Context Protocol (MCP) servers that connect AI coding agents — Claude Code, OpenAI Codex CLI, Gemini CLI — to the outside world: browsers, cloud platforms, databases, infrastructure, and domain APIs. One registration pattern across all three CLIs, the per-server specifics, and the supply-chain checks to run before enabling any of them.

> *If I were wiring a coding agent into real services — a browser, a database, a deploy host — what would I register, and what would I lock down first?*

**Scope.** Integration servers only — the ones that let an agent *do* things in external systems. Two sibling repos own the other layers:

- **Token-efficiency tooling** (RTK, LSP, Context7, `codebase-memory-mcp`, claude-mem) → [Claude Code Token Optimization](https://github.com/khasky/claude-code-token-optimization). Those servers earn their keep by cutting tokens, not by reaching outward.
- **The base, tool-agnostic ruleset** for how agents should behave → [Awesome AGENTS.md](https://github.com/khasky/awesome-agents-md). Register nothing here and that ruleset still works — these servers are optional accelerators.

---

## Contents

- [Servers at a glance](#servers-at-a-glance)
- [Prerequisites](#prerequisites)
- [Vet before enabling](#vet-before-enabling)
- [Registration pattern](#registration-pattern)
- [Credentialed servers](#credentialed-servers)
- [SSH host list](#ssh-host-list)
- [Unity — local build](#unity--local-build)
- [Cloudflare endpoints](#cloudflare-endpoints)
- [Verify registration](#verify-registration)
- [Safe defaults](#safe-defaults)
- [Related](#related)
- [License](#license)

---

## Servers at a glance

| Server | Command / endpoint | Requires | Reach for it when |
|---|---|---|---|
| `playwright` | `npx -y @playwright/mcp@latest` | — | browser automation, user-flow checks, forms, navigation, screenshots, responsive/regression, accessibility trees |
| `chrome-devtools` | `npx -y chrome-devtools-mcp@latest` | — (Claude Code also ships it as an official plugin) | console errors, network, performance, source maps, runtime profiling, layout/rendering analysis |
| `cloudflare` | `npx -y mcp-remote https://docs.mcp.cloudflare.com/mcp` | browser OAuth on first use | Cloudflare docs and product APIs |
| `neon` | remote `https://mcp.neon.tech/mcp` (native `http` type or `mcp-remote`) | Neon account (OAuth) or API key | Postgres — **prefer a read-only role** |
| `docker` | `uvx mcp-server-docker` | running Docker daemon | container state — mutates, confirm destructive actions |
| `ssh` | `npx -y @uarlouski/ssh-mcp-server@latest --configPath=<path>` | host list JSON (below) | remote hosts — mutates, confirm destructive actions |
| `trello` | `npx -y @delorenj/mcp-server-trello` | `TRELLO_API_KEY`, `TRELLO_TOKEN` | boards, cards, checklists |
| `resend` | `npx -y resend-mcp` | `RESEND_API_KEY` | transactional email |
| `steam` | `npx -y @tmhs/steam-mcp` | `STEAM_API_KEY` | Steam data lookups |
| `unity` | node bridge from [CoderGamester/mcp-unity](https://github.com/CoderGamester/mcp-unity) | Unity project + UPM package, server on port 8090 (setup below) | scene, GameObject, and asset operations in the Editor |

Register only the servers you actually use, at **user scope** so they are available in every project. An idle server is not free — every connected server adds tool names, descriptions, and schemas to the context, and a browser `snapshot` can return tens of thousands of tokens in a single tool result. Register narrowly; toggle per task in `/mcp`.

`trello`, `resend`, `steam`, and `unity` are domain-specific — no credentialed or production writes without explicit authorization. `neon`, `docker`, and `ssh` reach real state; treat every write as destructive until proven otherwise.

---

## Prerequisites

Most servers run through `npx`; the `docker` server needs `uvx`. On Windows (skip what you already have):

```powershell
winget install -e --id OpenJS.NodeJS.LTS   # npx — needed by every npx-based server below
winget install -e --id astral-sh.uv        # uvx — only for the docker MCP server
```

macOS/Linux: install Node.js (`npx`) from your package manager, and [uv](https://github.com/astral-sh/uv) if you want the docker server.

Windows tip — Python-based MCP tools print mojibake without UTF-8 forced:

```powershell
[Environment]::SetEnvironmentVariable("PYTHONUTF8", "1", "User")
[Environment]::SetEnvironmentVariable("PYTHONIOENCODING", "utf-8", "User")
```

---

## Vet before enabling

A third-party MCP server runs with your user's permissions — treat it like an npm package with install scripts:

- Check that it is maintained and open source; skim it for `eval`/`exec` of fetched content and for credential access beyond its stated purpose.
- Pin exact versions where the tool supports it — never blindly trust `latest` for something that can run commands on your machine or reach your data.
- Prefer a sandboxed first session and, for credentialed servers, the narrowest scope that works — a **read-only database user** for `neon`, read-only tokens where the service offers them.
- Everything a server returns is data, not instructions. An agent that expands its own permissions because a tool result told it to is the failure mode; keep permissions outside the model's reach.
- Re-check after major version bumps.

---

## Registration pattern

**Windows:** stdio servers need the `cmd /c npx.cmd` wrapper. On macOS/Linux drop it and call `npx` directly.

Canonical example — Playwright, registered in all three CLIs:

```powershell
claude mcp add --scope user playwright -- cmd /c npx.cmd -y @playwright/mcp@latest
codex  mcp add playwright -- cmd /c npx.cmd -y @playwright/mcp@latest
gemini mcp add -s user --timeout 30000 playwright cmd /c npx.cmd -y @playwright/mcp@latest
```

Claude Code syntax gotcha on Windows: use this positional form, not `add-json` (PowerShell strips the JSON quoting). The server name goes before any `-e` flags, and `--` comes right after the last flag.

Servers without credentials follow the same shape — swap the name and command from the table. Two of them differ:

- **`neon`** is a remote HTTP server, not a spawned process. Register it with the native `http` transport (`claude mcp add --transport http neon https://mcp.neon.tech/mcp`) or via `mcp-remote`; first use triggers a browser OAuth (or supply an API key).
- **`cloudflare`** is also remote, reached through `mcp-remote` with a browser OAuth on first use (see [Cloudflare endpoints](#cloudflare-endpoints) for the per-product URLs).

---

## Credentialed servers

Servers with credentials take them as environment variables at registration time — pass each key with `-e KEY=val` (Claude Code, Gemini) or `--env KEY=val` (Codex, stdio servers only). **Never write real keys into files tracked by git**; keep them in your user environment and reference them.

Trello, registered in all three CLIs:

```powershell
claude mcp add --scope user trello -e "TRELLO_API_KEY=$env:TRELLO_API_KEY" -e "TRELLO_TOKEN=$env:TRELLO_TOKEN" -- cmd /c npx.cmd -y @delorenj/mcp-server-trello
codex  mcp add trello --env "TRELLO_API_KEY=$env:TRELLO_API_KEY" --env "TRELLO_TOKEN=$env:TRELLO_TOKEN" -- cmd /c npx.cmd -y @delorenj/mcp-server-trello
gemini mcp add -s user -e "TRELLO_API_KEY=$env:TRELLO_API_KEY" -e "TRELLO_TOKEN=$env:TRELLO_TOKEN" trello cmd /c npx.cmd -y @delorenj/mcp-server-trello
```

Resend is the same shape with one key (`steam` follows identically with `STEAM_API_KEY`):

```powershell
claude mcp add --scope user resend -e "RESEND_API_KEY=$env:RESEND_API_KEY" -- cmd /c npx.cmd -y resend-mcp
codex  mcp add resend --env "RESEND_API_KEY=$env:RESEND_API_KEY" -- cmd /c npx.cmd -y resend-mcp
gemini mcp add -s user -e "RESEND_API_KEY=$env:RESEND_API_KEY" resend cmd /c npx.cmd -y resend-mcp
```

---

## SSH host list

The `ssh` server reads a host list — create `%USERPROFILE%\.mcp\ssh\ssh-mcp-config.json` (UTF-8 **without BOM**, otherwise the server fails to parse it):

```json
{
  "servers": {
    "my-host": {
      "host": "203.0.113.10",
      "port": 22,
      "username": "deploy",
      "privateKeyPath": "C:/Users/me/.ssh/id_ed25519"
    }
  }
}
```

Register it with `--configPath` pointing at that file, e.g.:

```powershell
claude mcp add --scope user ssh -- cmd /c npx.cmd -y @uarlouski/ssh-mcp-server@latest --configPath="$env:USERPROFILE\.mcp\ssh\ssh-mcp-config.json"
```

---

## Unity — local build

Unity MCP is the only server that needs a local build — a node bridge plus a UPM package inside your Unity project:

```powershell
git clone --depth 1 https://github.com/CoderGamester/mcp-unity.git "$env:USERPROFILE\.mcp\mcp-unity"
Push-Location "$env:USERPROFILE\.mcp\mcp-unity\Server~"
npm install
npm run build
Pop-Location
claude mcp add --scope user unity -e UNITY_PORT=8090 -- cmd /c node "$env:USERPROFILE\.mcp\mcp-unity\Server~\build\index.js"
codex  mcp add unity --env UNITY_PORT=8090 -- cmd /c node "$env:USERPROFILE\.mcp\mcp-unity\Server~\build\index.js"
gemini mcp add -s user -e UNITY_PORT=8090 unity cmd /c node "$env:USERPROFILE\.mcp\mcp-unity\Server~\build\index.js"
```

Then in Unity: Window → Package Manager → Add package from git URL → `https://github.com/CoderGamester/mcp-unity.git`, and start the bridge via Tools → MCP Unity → Server Window → Start Server (WebSocket, port 8090).

---

## Cloudflare endpoints

`docs.mcp.cloudflare.com` is one of several endpoints — other products (bindings, observability, builds, browser, radar, graphql, …) live at `https://<product>.mcp.cloudflare.com/mcp`. Register each the same way under its own name:

```powershell
claude mcp add --scope user cloudflare-docs -- cmd /c npx.cmd -y mcp-remote https://docs.mcp.cloudflare.com/mcp
claude mcp add --scope user cloudflare-observability -- cmd /c npx.cmd -y mcp-remote https://observability.mcp.cloudflare.com/mcp
```

Each triggers a browser OAuth on first use.

---

## Verify registration

```powershell
claude mcp list
codex  mcp list
gemini mcp list
```

or run `/mcp` inside an interactive session. If Claude Code gets stuck on MCP trust prompts: `claude mcp reset-project-choices`, then restart.

---

## Safe defaults

The rules that keep these servers from doing damage — same bar you would set for a capable new hire with no track record:

- **Read-only where possible.** A read-only database role for `neon`; read-only tokens for any service that offers them.
- **Confirm destructive actions.** `docker` and `ssh` mutate real state — pause and confirm before anything that deletes, overwrites, or restarts. Prefer a backup or rollback path first.
- **No production or credentialed writes without explicit authorization** — `trello`, `resend`, `steam`, `unity`, and any server pointed at a live system.
- **Secrets never in tracked files.** Keys live in the user environment and are referenced (`$env:API_KEY`), never inlined into configs, code, or commits.
- **Missing configuration fails loud** — no silent default endpoints or credentials for a server that isn't actually set up.

---

## Related

Three guides, one split — pick the layer you need:

- [Awesome AGENTS.md](https://github.com/khasky/awesome-agents-md) — the base, tool-agnostic ruleset every agent imports (one `AGENTS.md`). Start here; these servers are optional on top.
- [Awesome Agent Skills](https://github.com/khasky/awesome-agent-skills) — portable `SKILL.md` skills every agent loads: code review, debugging, security and leak audits, code and text cleanup.
- [Claude Code Token Optimization](https://github.com/khasky/claude-code-token-optimization) — the token-efficiency layer (RTK, LSP, Context7, `codebase-memory-mcp`, claude-mem, Caveman, Ponytail).
- **Agent MCP Integrations** — *this repo:* MCP servers that connect agents to browsers, cloud, databases, infra, and domain APIs.
- [Claude Code Security Audit](https://github.com/khasky/claude-code-security-audit) — the layered security-audit workflow (deep audit, continuous guardrails, scanners).

---

## License

Released under the [MIT license](LICENSE). The MCP servers referenced here are third-party projects under their own licenses — check each before use.
