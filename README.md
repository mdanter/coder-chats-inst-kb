# Bringing institutional knowledge into Coder Agents

Coder Agents runs the agent loop in your Coder control plane rather than inside developer workspaces. That architectural choice makes "institutional knowledge" a first-class, governable concern rather than a per-developer configuration problem. This guide walks through the four composable layers Coder Agents provides for injecting company-specific knowledge into every agent session, and closes with a pattern for managing the deployment-wide system prompt as code.

Everything in this guide references only publicly documented Coder features. Links to public docs appear at the end.

## The four layers

Think of these as concentric rings, from deployment-wide down to per-repository.

### 1. Deployment-wide system prompt

An admin-set prompt that Coder appends to its built-in prompt for every Coder Agents chat, deployment-wide. Good uses:

- Company identity ("You are helping engineers at Acme Corp.")
- Coding standards, commit-message conventions, branch-naming rules
- Approved-vendor / approved-SaaS policy and escalation paths
- Template routing hints ("For payments work, prefer the `payments-python` template.")
- Environment-specific gotchas your engineers hit repeatedly

You can configure it two ways:

- **Dashboard**: **AI Settings → Coder Agents → Instructions**. Admin-only.
- **API**: the endpoints live under the chats-config route. Coder v2.36.0 documentation references `PUT /api/v2/chats/config/system-prompt`; earlier releases and current source expose the same route under `/api/experimental/chats/config/system-prompt`. Confirm the exact path in the docs for your Coder version before scripting against it.

The `PUT` payload accepts two fields:

- `system_prompt` — your prompt text.
- `include_default_system_prompt` — when `true`, your text is *appended* to Coder's built-in prompt rather than replacing it. This is the recommended setting.

`GET` on the same path returns the current `system_prompt`, the `include_default_system_prompt` flag, and the platform-provided `default_system_prompt` (for reference).

### 2. Skills — repository- and workspace-scoped instruction packs

Skills are structured, reusable instruction sets that live in the workspace filesystem and are auto-discovered by the agent when a chat attaches to a workspace. They are the right home for domain playbooks — `deep-review`, `release-checklist`, `payments-domain-conventions`, and similar.

Layout: place skill directories under `.agents/skills/` relative to the workspace working directory. Each directory contains a required `SKILL.md` file plus any supporting files the skill needs.

```
.agents/skills/
├── deep-review/
│   ├── SKILL.md
│   └── roles/
│       ├── security-reviewer.md
│       └── concurrency-reviewer.md
├── pull-requests/
│   └── SKILL.md
└── refine-plan/
    └── SKILL.md
```

`SKILL.md` starts with YAML frontmatter and a markdown body:

```md
---
name: deep-review
description: "Multi-reviewer code review with domain-specific reviewers"
---

# Deep Review

Instructions for the skill go here...
```

Mechanics worth knowing:

- **Lazy loading.** On the first turn of a workspace-attached chat, the agent scans `.agents/skills/` and builds an `<available-skills>` block in its system prompt listing each skill's name and description. Only frontmatter is read during discovery. The full skill content is loaded on demand when the agent calls the `read_skill` tool. You can ship many skills without inflating context.
- **Two tools are registered when skills are present:** `read_skill` (returns the `SKILL.md` body, the absolute skill directory for workspace skills, and a list of supporting files) and `read_skill_file` (returns the content of a supporting file, with path-safe resolution).
- **Naming and size constraints:** names must be kebab-case (`^[a-z0-9]+(-[a-z0-9]+)*$`) and match the directory name exactly. `SKILL.md` has a maximum size of 64 KB. Supporting files have a maximum of 512 KB; files exceeding that limit are silently truncated.
- **Path safety.** `read_skill_file` rejects absolute paths, paths containing `..`, and references to hidden files. All paths are resolved relative to the skill directory.

**Personal Skills** are the user-scoped counterpart, managed under **Agents → Settings → Personal Skills**. They use the same `SKILL.md` format, are portable to and from workspace skills, and are subject to two extra limits: **supporting files are not supported** (single `SKILL.md` file only), each `SKILL.md` is up to 64 KB, and each user can create up to 100 personal skills. Use these for individual preferences that shouldn't live in a shared repository.

### 3. Admin-registered MCP servers — deployment-wide knowledge systems

This is the canonical hook for wiring institutional knowledge systems into agents: wikis, service catalogs, ticket systems, internal RAG endpoints, vector databases, code search, and similar. If it has an API, an MCP server can front it.

Admins register MCP (Model Context Protocol) servers at **AI Settings → Coder Agents → MCP servers** (`/ai/settings/mcp-servers`). Key options:

- **Transports.** `streamable_http` or `sse`.
- **Availability policies:**
  - `force_on` — always injected into every chat; users cannot opt out. Appropriate for security/compliance tooling.
  - `default_on` — pre-selected in new chats; users can opt out. A good default for broadly useful org-wide knowledge.
  - `default_off` — available in the server list but users must opt in. Good for niche tools.
- **Authentication modes:** None, OAuth2 (with optional RFC 9728 / RFC 8414 / RFC 7591 auto-discovery and RFC 7009 token revocation), API key, custom headers, or **User OIDC Identity** (forwards the calling user's OIDC access token as `Authorization: Bearer <token>`; only works for users who authenticated to Coder via OIDC).
- **Tool governance:** per-server `tool_allow_list` and `tool_deny_list`. If `tool_allow_list` is non-empty, only the listed tool names are exposed; `tool_deny_list` blocks named tools even if they appear in the allow list. You do not have to expose every tool a server ships.
- **Identity forwarding (opt-in):** setting `forward_coder_headers = true` sends Coder identity headers on every outgoing request so a first-party MCP server can correlate a tool call back to the originating chat:
  - `X-Coder-Owner-Id` — the Coder user who owns the chat that issued the tool call.
  - `X-Coder-Chat-Id` — the top-level (parent) chat ID.
  - `X-Coder-Subchat-Id` — the subchat ID, only present when the request originates from a child chat.
  - `X-Coder-Workspace-Id` — the workspace associated with the chat, if any.

  Because these headers leak chat identity, the option is off by default and should only be enabled for first-party or trusted internal MCP servers.

Only enabled servers are visible to non-admins, and sensitive fields such as API keys and client secrets are never returned in API responses (Coder returns boolean flags indicating whether a value is set).

### 4. Workspace-scoped MCP (`.mcp.json`) — per-repo, per-template tools

For MCP servers that are specific to a repository or template (for example, a schema explorer for a particular service, or a repo-local internal API docs server), drop a `.mcp.json` at the workspace working directory root:

```json
{
  "mcpServers": {
    "github": {
      "command": "github-mcp-server",
      "args": ["--token", "..."]
    },
    "my-api": {
      "type": "http",
      "url": "http://localhost:8080/mcp",
      "headers": { "Authorization": "Bearer ..." }
    }
  }
}
```

- **Stdio transport:** set `command`, and optionally `args` and `env`. The agent spawns the process in the workspace.
- **HTTP transport:** set `url`, and optionally `headers`. The agent connects to the HTTP endpoint from the workspace.
- **Discovery:** the agent reads `.mcp.json` via the workspace connection on each chat turn, with a 5-second discovery timeout. Servers that fail to respond are skipped; partial success is acceptable.
- **Tool naming:** tool names are prefixed with the server name as `serverName__toolName` to avoid collisions between servers and with built-in tools.
- **Tool call timeout:** 60 seconds per invocation.

## Two related surfaces worth mentioning

**`AGENTS.md`.** Create an `AGENTS.md` file in the workspace agent's working directory (or `~/.coder/AGENTS.md`). It is automatically read and included in the system prompt for every conversation with a Coder Agent that uses that workspace. Good for repository-specific build and test instructions, architectural constraints, and links to runbooks.

**Template routing.** When the agent needs a workspace, it selects a template by reading the template's name, short description (limited to under 128 characters, sorted by active developer count), and a bounded README excerpt (roughly the first 1,000 characters in the listing view, up to roughly 8,000 characters in the detail view). The excerpt is reduced to plain text: frontmatter is stripped, link text is kept while URLs are dropped, images and badges are dropped entirely, and code blocks and tables are preserved as text. The agent does not read Terraform. Clear, specific template descriptions are the strongest routing signal you can give the agent. Admins can also restrict which templates the agent may use at **Agents → Settings → Manage Agents → Templates** without affecting manual workspace creation.

## A recommended composition

- **Layer 1 (system prompt):** company identity, standards, approved-SaaS list, template routing hints, escalation paths.
- **Layer 2 (`.agents/skills/`):** commit domain playbooks alongside code. Lazy loading means scaling to many skills is cheap.
- **Layer 3 (admin MCP servers):** connect the knowledge systems you already run. Use `default_on` for broadly useful org-wide knowledge, `force_on` for security/compliance tooling, `default_off` for niche integrations.
- **Layer 4 (workspace `.mcp.json`):** repo- or template-specific tools that don't make sense deployment-wide.
- **Personal Skills:** individual engineer preferences, without polluting shared repositories.

## Managing the system prompt as code

The dashboard is fine for iteration, but at enterprise scale the deployment-wide system prompt benefits from the same discipline you apply to any other production configuration: version control, peer review, CI-driven deploys, and drift detection.

### Why manage it as code

- **Auditability.** Every change is a git commit with an author, a reviewer, and a rationale.
- **Rollback.** `git revert` and redeploy.
- **Drift detection.** A change someone makes in the dashboard shows up as a diff on the next PR plan.
- **Environment parity.** The same prompt promotes cleanly across staging and production.

### Repository layout

A minimal shape:

```
company-coder-config/
├── coderd/
│   ├── terraform.tf          # coder/coderd provider pin
│   ├── backend.tf            # remote state (S3, GCS, etc.)
│   ├── main.tf               # AI providers + Agents model configs
│   └── system-prompt.md      # the system prompt, deployed byte-for-byte
├── .github/
│   └── workflows/
│       └── coderd.yaml       # plan on PR, apply on merge
└── .prettierignore           # include system-prompt.md so formatters leave it alone
```

### Terraform for providers and models

Use the [`coder/coderd`](https://registry.terraform.io/providers/coder/coderd) Terraform provider for AI providers and models. It authenticates against a Coder deployment via `url` and `token` (or the `CODER_URL` and `CODER_SESSION_TOKEN` environment variables). The provider requires **Coder v2.10.1 or later**; pin a specific provider version that matches your Coder version and check the provider's release notes when upgrading.

```hcl
# terraform.tf
terraform {
  required_providers {
    coderd = {
      source  = "coder/coderd"
      version = "~> 0.0.22" # pin to a version compatible with your Coder release
    }
  }
}

provider "coderd" {
  # url and token are read from CODER_URL and CODER_SESSION_TOKEN if omitted
}
```

An AI provider resource, using write-only credential arguments so the live API key value is never stored in Terraform state and is preserved across applies:

```hcl
# main.tf
resource "coderd_ai_provider" "openai" {
  type         = "openai"
  name         = "openai"
  display_name = "OpenAI"
  enabled      = true
  base_url     = "https://api.openai.com/v1"

  api_key_wo         = var.openai_api_key
  api_key_wo_version = 1 # bump to rotate
}
```

> Note: `coderd_ai_provider` is currently marked **experimental** by the provider — see its resource documentation for the latest schema and stability guidance before adopting it in production. Write-only arguments require Terraform 1.11 or later.

### The system prompt file

`system-prompt.md` is plain markdown. It is deployed byte-for-byte, so add it to `.prettierignore` to keep formatters from rewriting it. Structure it however maps best to your organization; a generic starting shape:

```md
You are Coder Agents, an agentic system designed to help <Company> employees.

Company Guidelines:
For <use cases relevant to your organization>, follow <Company AI Guidelines URL>.
This may require the <name> MCP server to be connected. If tools are missing,
encourage the user to connect it.

If a specific SaaS is not mentioned in the guide, do not treat it as a
recommended solution. Make it clear it would require coordination with
<IT channel or contact>.

Templates:
- Use "<Template A>" for <use case A>.
- Use "<Template B>" for <use case B>. Note any pre-wired credentials, ports,
  or apps exposed by the template.

Data sources:
- <Data source 1> and how to access it.
- <Data source 2> and how to access it.
- Credentials are pre-injected as <env var>. Do not ask the user for keys.

Previewing:
- View changes in the "Desktop" tab in the right sidebar.
- To open a URL in a browser inside the workspace: <exact command>
- Shareable port-forwarded URL pattern: <URL pattern>

Framework gotchas:
- Next.js: add the proxy host to `allowedDevOrigins` in `next.config.ts`
  and restart the dev server. Otherwise the page loads but looks broken
  with no visible error.
- Vite: add to `server.allowedHosts` in `vite.config.ts`.

Deploying:
- Use the <deploy tool> CLI. Walk users through auth via the terminal
  tab in the right sidebar.
- Always link deploys to the source repository (not one-off uploads) so
  auto-deploy and PR previews work.
- No personal accounts for company projects — per IT policy.
```

Every line above is doing work: routing the agent to the right template, pointing at real internal systems, warning about known footguns, and enforcing policy. Fill in the placeholders with your own systems and conventions; the shape is what matters.

### CI workflow

Two jobs: plan on PRs to review changes, and apply on merge to `main` to deploy. The example below is deliberately portable — swap the state-backend auth step for whatever your CI runner uses.

```yaml
# .github/workflows/coderd.yaml
name: coderd

on:
  pull_request:
    paths: ["coderd/**", ".github/workflows/coderd.yaml"]
  push:
    branches: [main]
    paths: ["coderd/**", ".github/workflows/coderd.yaml"]

concurrency:
  group: coderd-apply
  cancel-in-progress: false

env:
  CODER_URL: https://coder.example.com
  # CODER_SESSION_TOKEN comes from a repo secret. AI providers and models are
  # deployment-wide (site-level) resources, so this token needs owner scope.
  CODER_SESSION_TOKEN: ${{ secrets.CODER_SESSION_TOKEN }}
  # Adjust to match the path your Coder version exposes. See note below.
  SYSTEM_PROMPT_PATH: /api/experimental/chats/config/system-prompt

jobs:
  plan:
    if: github.event_name == 'pull_request'
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: hashicorp/setup-terraform@v3

      # Authenticate to your Terraform state backend here.

      - name: terraform init & plan
        working-directory: coderd
        run: |
          terraform init
          terraform validate
          terraform plan -no-color

      - name: Diff system prompt against live value
        working-directory: coderd
        run: |
          curl -fsS -H "Coder-Session-Token: $CODER_SESSION_TOKEN" \
            "$CODER_URL$SYSTEM_PROMPT_PATH" \
            | jq -r '.system_prompt' > /tmp/live-system-prompt.md
          diff -u /tmp/live-system-prompt.md system-prompt.md \
            || echo "::warning::system-prompt.md differs from live value"

  apply:
    if: github.ref == 'refs/heads/main' && github.event_name == 'push'
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: hashicorp/setup-terraform@v3

      # Authenticate to your Terraform state backend here.

      - name: terraform apply
        working-directory: coderd
        run: |
          terraform init
          terraform apply -auto-approve

      - name: Deploy system prompt via API
        working-directory: coderd
        run: |
          jq -Rs '{system_prompt: ., include_default_system_prompt: true}' \
            < system-prompt.md > /tmp/payload.json
          curl -fsS -X PUT \
            -H "Coder-Session-Token: $CODER_SESSION_TOKEN" \
            -H "Content-Type: application/json" \
            --data-binary @/tmp/payload.json \
            "$CODER_URL$SYSTEM_PROMPT_PATH"
```

A couple of things worth flagging honestly:

- **API path.** Coder is actively iterating on the chats-config API surface. As noted earlier, the v2.36.0 docs reference `PUT /api/v2/chats/config/system-prompt` while current source paths still expose `/api/experimental/chats/config/system-prompt`. Set `SYSTEM_PROMPT_PATH` to whichever one your Coder version exposes and be prepared to update it on upgrade.
- **Native Terraform resource for the system prompt.** As of writing, the `coderd` Terraform provider does not yet have a native resource for the deployment-wide chat system prompt, which is why this recipe deploys it via a workflow-driven API `PUT`. If and when a native resource ships, folding `system-prompt.md` into the Terraform module and dropping the workflow step is straightforward.

### CI role scoping

Two least-privilege OIDC roles is the tidy pattern:

- **`plan` role** — read-only against the state backend. Used on PRs.
- **`apply` role** — read/write against the state backend. Used on merge to `main`.

Guard the `apply` job on `github.ref == 'refs/heads/main'` and use a `concurrency` group so two applies never race for the state lock. The Coder session token in either job must be owner-scoped, because AI providers and models are deployment-wide resources.

## What Coder Agents deliberately does not ship

- **No built-in knowledge base or RAG.** Coder Agents is the integration and governance layer. You bring the knowledge system (wiki, code search, RAG endpoint, vector database); Coder exposes it via MCP with auth, tool governance, and identity forwarding.
- **No packaged starter-skills library.** Skills are authored per organization because they encode organization-specific conventions.

## Public documentation

- [Getting Started](https://coder.com/docs/latest/ai-coder/agents/getting-started), including "Set a deployment-wide system prompt"
- [Platform Controls](https://coder.com/docs/latest/ai-coder/agents/platform-controls)
- [Platform Controls → MCP Servers](https://coder.com/docs/latest/ai-coder/agents/platform-controls/mcp-servers)
- [Platform Controls → Template Optimization](https://coder.com/docs/latest/ai-coder/agents/platform-controls/template-optimization)
- [Extending Agents](https://coder.com/docs/latest/ai-coder/agents/extending-agents) — skills and workspace MCP
- [Best Practices](https://coder.com/docs/latest/ai-coder/best-practices)
- [`coder/coderd` Terraform provider](https://registry.terraform.io/providers/coder/coderd)
- [Model Context Protocol](https://modelcontextprotocol.io/introduction)
