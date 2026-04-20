# Team Setup Guide — VS Code, Copilot, MCP, and Workspaces

**Purpose.** A portable guide for our team and close collaborators. Takes you from a fresh Windows machine to a working, consistent environment aligned with how we do development, database, cloud, and security work. Written so someone new to VS Code, Copilot, or MCP can follow it without outside context.

**Audience.** DBAs, developers, architects, and engineers on our team and adjacent teams we collaborate with closely. No prior experience required.

**How to use this.** Read section 1 first. Do section 2. Then pick the sections relevant to your role. The whole thing takes a few hours the first time. Updating later is quick.

**Companion document.** This guide focuses on the individual developer setup. For the enterprise-wide governance, rollout stages, and multi-org strategy that surrounds this work, see the *Enterprise Multi-Org DevSecOps Strategy* document.

-----

## 0. Table of Contents

1. Why we do this
1. Install the basics (Windows, Git, VS Code, WSL)
1. Sign in and verify Copilot access
1. VS Code Profiles — what they are and how we use them
1. Workspaces — one or many, and why
1. Copilot — getting more from it with instruction files, prompts, and Spaces
1. MCP — connecting Copilot to real systems
1. Extensions by role
1. Settings and repo scaffolding
1. Day-in-the-life examples
1. Troubleshooting and gotchas
1. Glossary — plain language
1. Where to ask questions
1. What’s next after this guide

-----

## 1. Why We Do This

Most of us have VS Code installed somewhere and use Copilot casually. This guide takes it one step further: a setup that’s **consistent across the team**, **tuned to our work**, and **portable** so a new teammate can be productive on day one.

The point isn’t to install every extension that exists. The point is to make the right choices the default — the right extensions load for the right work, Copilot knows our conventions, and our shared repos open the same way for everyone.

**Key idea: most of the value comes from four small things.**

- Profiles (so you don’t drown in extensions).
- Workspaces (so Copilot sees the right context).
- Instruction files and Spaces (so Copilot suggests code in our style and grounds answers in our docs).
- MCP (so Copilot can actually *see* GitHub, SQL, Azure, etc.).

The rest is supporting cast.

-----

## 2. Install the Basics

Do these in order on a fresh Windows machine. If something is already installed, skip it.

### 2.1 Required

Open PowerShell as Administrator (or use a regular terminal if your machine restricts admin). Install via `winget`:

```powershell
winget install --id Git.Git -e
winget install --id Microsoft.PowerShell -e
winget install --id Microsoft.VisualStudioCode -e
```

If `winget` isn’t available or allowed, download installers from:

- Git: https://git-scm.com/download/win
- PowerShell 7: https://aka.ms/powershell
- VS Code: https://code.visualstudio.com/

### 2.2 Recommended — WSL2

WSL lets you run Linux alongside Windows. You’ll want it for:

- Running MCP servers that behave better on Linux.
- Container and security scanning tools (Trivy, Checkov, tfsec).
- Anything Node or Python where Windows paths get awkward.

```powershell
wsl --install -d Ubuntu-24.04
```

Restart when prompted. Set a username and password inside Ubuntu.

**You don’t need WSL for everything.** SQL Server work, PowerShell, Azure/AWS CLI, SSMS, Azure Data Studio, and Windows-authenticated tooling all stay on the Windows side. WSL is a second environment, not a replacement.

### 2.3 Cloud and Data Tools (install what matters for your role)

On Windows:

```powershell
winget install --id Microsoft.AzureCLI -e
winget install --id Amazon.AWSCLI -e
winget install --id Microsoft.SQLServerManagementStudio -e
winget install --id Microsoft.AzureDataStudio -e
```

The `SqlServer` PowerShell module:

```powershell
Install-Module -Name SqlServer -Scope CurrentUser
```

Inside WSL (Ubuntu terminal):

```bash
# Node via nvm — check https://github.com/nvm-sh/nvm for the current version before pasting
curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/master/install.sh | bash
# reopen terminal
nvm install --lts

# Python via uv
curl -LsSf https://astral.sh/uv/install.sh | sh

# Azure CLI, AWS CLI per the vendor docs for Ubuntu
```

> Pin install URLs to known-good versions in our internal copy of this guide. The `master` branch is fine for first install, but versioned tags are safer for reproducible onboarding scripts.

### 2.4 Git Credential Manager — The One Thing That Trips People Up

If you have both a personal GitHub account and our enterprise EMU account on the same machine, Git Credential Manager can get confused about which one to use. Fix this up front.

In your Windows `~/.gitconfig`, add:

```
[credential "https://github.com"]
    provider = github
    useHttpPath = true
[credential "https://github.com/YourEnterprise"]
    username = your_emu_username
```

Replace `YourEnterprise` with our actual enterprise slug and `your_emu_username` with your EMU account (EMU usernames typically look like `firstname-lastname_enterpriseslug`).

If you’re using WSL, you can share the Windows credential manager so you don’t have to log in twice:

```bash
git config --global credential.helper "/mnt/c/Program\ Files/Git/mingw64/bin/git-credential-manager.exe"
```

-----

## 3. Sign In and Verify Copilot

Open VS Code. Click the account icon in the bottom-left corner. Sign in with GitHub using your **EMU identity** (not your personal account). For Commonwealth work specifically, this means your `.pennsylvania.gov`-linked EMU account.

Install the Copilot extensions:

- **GitHub Copilot** (code suggestions)
- **GitHub Copilot Chat** (the chat panel, agent mode, and Spaces access)

To verify Copilot is active:

- Open a new file, type a function signature, see if you get a gray suggestion.
- Open the Chat panel (`Ctrl+Alt+I`), ask it something.
- Check the model picker in the Chat input — you should see multiple models available (GPT, Claude, Gemini variants). If the picker only shows one model, your org may need to enable additional models in Copilot policy.

If either doesn’t work, check `Settings → GitHub → Copilot` and confirm you’re signed in with the enterprise account. If you still can’t sign in, the org may need to add you to the Copilot seat list — ask the GitHub admin rotation.

**A note on premium requests.** Copilot Enterprise and Business seats include a monthly allocation of “premium requests,” which power Chat, agent mode, code review, and model selection. Most normal use stays well inside the allocation; heavy agent-mode or code-review usage can consume it faster. The Copilot usage dashboard shows consumption; the COE publishes guidance on when to switch to a faster/cheaper model for routine work.

-----

## 4. VS Code Profiles — What They Are and Why They Matter

**A profile is a bundle of extensions, settings, keybindings, snippets, and UI state.** You switch between profiles depending on what you’re doing. Loading a profile is nearly instant. The alternative — one giant VS Code with 60 extensions installed — is slow and noisy, and Copilot’s suggestions get worse because it’s wading through irrelevant context.

### 4.1 Profiles We Recommend

Create these from `File → Profiles → New Profile`:

- **Default** — nearly empty. For quick file edits.
- **SQL & Data Platform** — SQL Server, database projects, Kusto, dbt.
- **Cloud & IaC** — Azure, AWS, Bicep, Terraform, Kubernetes, Docker.
- **PowerShell & Automation** — PowerShell, Pester.
- **DevSecOps** — GitHub Actions, security scanners, OPA, dev containers.
- **Docs & Architecture** — Markdown, Mermaid, Draw.io, PlantUML.

You don’t have to create all six. Create the ones matching your role. You can always add more.

**Tip:** Our team publishes exported profile templates (`.code-profile` files) in the `workspaces` repo. You can import them directly instead of hand-curating each one.

### 4.2 What Goes In Every Profile

Core extensions that are always useful:

- GitHub Copilot
- GitHub Copilot Chat
- GitLens
- Error Lens
- EditorConfig
- Code Spell Checker
- Markdown All in One
- Remote - WSL
- Remote - SSH

### 4.3 Why This Matters

Switching profiles takes a second. Switching between “I’m editing T-SQL” and “I’m writing Terraform” without changing profile means Copilot sees both contexts and guesses worse. Profiles are how you keep each workspace focused.

-----

## 5. Workspaces — One or Many?

A **workspace** is a `.code-workspace` file that groups one or more folders together. Open the workspace and you get all those folders in one VS Code window.

### 5.1 The Answer: Both

- **For casual work** — just open a folder. No workspace needed.
- **For connected work across several repos** — use a multi-root workspace.

### 5.2 Why Multi-Root Workspaces Are Useful

When you ask Copilot Chat a question like “how does our backup script call the shared logging module,” Copilot’s `@workspace` agent searches across all folders in the workspace. If you only have one repo open, it can’t find the other one.

A multi-root workspace is one of the easiest ways to give Copilot the right context.

### 5.3 Suggested Workspaces

Stored in a team `workspaces` repo so everyone gets the same view:

- `data-platform.code-workspace` — SQL backup automation, BCP pipeline, shared PS modules, data platform ADRs.
- `cloud-governance.code-workspace` — landing zone, control mapping, policy-as-code, cost reporting.
- `devsecops-coe.code-workspace` — org-level `.github` repo, reusable workflows, CODEOWNERS templates, security baselines.
- `enterprise-architecture.code-workspace` — governance artifacts, RACI docs, operating models, ADRs.

Short-lived ones we create as needed:

- `security-integration-<topic>.code-workspace` — when we’re working directly with Enterprise Security on something.
- `agency-onboarding-<agency>.code-workspace` — during an onboarding engagement with another Commonwealth agency. Archived after.

### 5.4 Anatomy of a `.code-workspace` File

```json
{
  "folders": [
    { "name": "🗄️ SQL Backup Automation", "path": "../repos/sql-backup-automation" },
    { "name": "📦 BCP Export Pipeline",    "path": "../repos/bcp-export-pipeline" },
    { "name": "🔧 Shared PS Modules",      "path": "../repos/ps-shared-modules" },
    { "name": "📐 ADRs",                   "path": "../repos/data-platform-adrs" }
  ],
  "settings": {
    "github.copilot.chat.codeGeneration.useInstructionFiles": true,
    "files.exclude": { "**/bin": true, "**/obj": true }
  },
  "extensions": {
    "recommendations": [
      "ms-mssql.mssql",
      "ms-vscode.powershell",
      "github.copilot",
      "github.copilot-chat",
      "eamodio.gitlens"
    ]
  }
}
```

The `extensions.recommendations` array matters. When anyone opens this workspace, VS Code prompts them to install the right tools. Combined with a profile, onboarding becomes automatic.

### 5.5 Rule of Thumb

One workspace per *coherent problem space*, not per team and not per repo. If you’d naturally ask questions that span several repos together, they belong in the same workspace. If you rarely need both open, keep them separate.

-----

## 6. Copilot — Getting More From It

Most of the value of Copilot Enterprise is in features people don’t know about. This section is the most important one in this guide.

### 6.1 Instruction Files — Shape Every Suggestion

Create `.github/copilot-instructions.md` in a repo. Every Copilot Chat turn in that repo automatically loads this file, and code generation gets shaped by it.

Keep instructions short, specific, and honest. Example for a PowerShell module:

```markdown
# Copilot Instructions — SQL Backup Automation

## Context
PowerShell 7 module for SQL Server 2019/2022/2025. Supports local, UNC, and S3 destinations. Runs under a service account with minimal SQL perms.

## Conventions
- PowerShell: approved verbs, StrictMode v3, comment-based help on public functions.
- T-SQL: schema-qualify every object. No SELECT *. Use sys.* catalog views.
- No inline credentials. Use SqlCredential or Managed Identity.
- Error handling: try/catch around every external call. Surface NativeCommandError details.

## Don't
- Don't suggest Invoke-Sqlcmd without -TrustServerCertificate or cert validation.
- Don't use Write-Host. Use Write-Verbose or Write-Information.
- Don't wrap everything in a broad try/catch that swallows errors.
```

You can also put `.instructions.md` files in subfolders with frontmatter targeting specific file patterns:

```markdown
---
applyTo: "**/*.ps1"
---
Rules that apply only to PowerShell files.
```

This is the single most effective way to make Copilot feel like a teammate who knows how you work.

### 6.2 Prompt Files — Reusable Prompts

Create `.github/prompts/<name>.prompt.md`. Invoke from Chat with `/<name>`.

Example `review-tsql.prompt.md`:

```markdown
# Review T-SQL

Review the selected T-SQL for:
- Sargability (can indexes be used?)
- Implicit type conversions
- NOLOCK usage (and whether it's justified)
- SELECT * usage
- Missing or redundant indexes suggested by the query
- Schema qualification on every object

Return a short list of findings with line references.
```

Other good ones to have:

- `/harden-powershell`
- `/generate-adr`
- `/threat-model`
- `/security-review`

### 6.3 Copilot Spaces — Ground Answers in Our Docs

**Important update.** Copilot *Knowledge Bases* were retired on November 1, 2025 and fully replaced by **Copilot Spaces**. If you remember hearing about “knowledge bases” from older documentation, the modern equivalent is a Space.

A **Space** is a curated container that Copilot uses as grounding context for Chat. Unlike the old knowledge bases (which were Markdown-only and organization-owner-created), Spaces can include:

- Code files from one or more repos (auto-syncs as files change).
- Free-text content: specs, transcripts, runbooks, architecture notes.
- Markdown, JSON, images, file uploads.
- Issues and pull requests.
- Custom instructions attached to the Space itself.

Anyone with a Copilot license can create a Space. Organization-owned Spaces can be shared with the org, with specific teams, or with named individuals. For Copilot Business and Enterprise orgs, admins enable the Spaces policy in Copilot settings.

**Our Spaces (planned / in progress):**

- **Data Platform Standards** — our SQL, backup, and data platform conventions.
- **DevSecOps COE Patterns** — CI/CD standards, security baselines, reusable workflows.
- **Cloud Landing Zone Reference** — Azure and AWS governance patterns.
- **Enterprise Security Standards** — published by the Enterprise Security org.
- **ITP / Commonwealth Policy Reference** — the IT Policy catalog and Executive Order 2016-06 reference material.

Use these heavily. In Chat, select the Space from the picker (or reference it inline). Answers get grounded in *our* standards instead of generic internet content.

**Migration note.** If your team had older “knowledge base” content, the GitHub migration tooling released in late 2025 converted those into individual Spaces. Check the Spaces list for your org before creating new ones from scratch.

### 6.4 Model Selection

Copilot Chat lets you pick the model per conversation. The current roster (which evolves — check the model picker for what’s actually available to you) includes reasoning-heavy models like Claude Opus / Claude Sonnet and GPT-5.x-class models, plus faster/cheaper options like Gemini Flash.

Rough guidance:

- **Strongest reasoning model** — architecture, threat modeling, hairy debugging, reviewing design tradeoffs, writing ADRs, SQL query-plan analysis.
- **Faster models** — boilerplate generation, small edits, simple refactors, commit messages.
- **Auto** (if available) — lets Copilot pick based on the task. Fine as a default when you’re not sure.

Don’t set and forget. Switch based on the task. The COE publishes a “which model when” cheat sheet in the COE repo.

**For Enterprise admins:** Bring Your Own Key (BYOK) is available as a preview for Copilot Enterprise. It lets the organization supply its own API keys from Anthropic, OpenAI, Azure AI Foundry, etc. This matters for data-residency and contracting reasons (billing under an existing enterprise agreement, staying within a specific cloud boundary). Discuss before enabling.

### 6.5 Chat Modes

- **Ask mode** — questions and explanations. Nothing changes in your files.
- **Edit mode** — Copilot proposes edits to files you specify. You approve each change.
- **Agent mode** — Copilot can read, edit, and run tools across multiple steps autonomously within the session. Most powerful, most risky.
- **Coding agent** (separate from in-editor agent mode) — assign a GitHub issue to Copilot and it works in the background, opens a PR, and pings you for review. “Mission Control” is the dashboard for managing multiple agent tasks at once. Start with low-risk repos (docs, ADRs, small refactors); establish trust before letting it touch production IaC.

Start with Ask and Edit. Move to Agent once you trust the patterns in your repo. Use the Coding Agent deliberately — it’s most useful for well-scoped, self-contained issues.

### 6.6 Code Review

Copilot can act as a first-pass reviewer on PRs, catching the obvious stuff so human reviewers focus on architecture and judgment. Enable on low-risk repos first (docs, ADRs, runbooks) and widen based on measured signal-to-noise. The COE publishes suggested CODEOWNERS patterns for when Copilot review is sufficient versus when a human reviewer is still required.

-----

## 7. MCP — Connecting Copilot to Real Systems

MCP (Model Context Protocol) lets Copilot Chat talk to outside systems: GitHub, SQL Server, Azure, AWS, your CMDB, your ticketing tool. Without MCP, Copilot only sees files in your workspace. With MCP, it can query your database, check a PR status, read an Azure policy assignment — and use what it finds to answer.

### 7.1 How MCP Configuration Works in VS Code

Two scopes:

- **User-level** — your personal MCP servers (credentials go here). Open with command palette → `MCP: Open User Configuration`.
- **Workspace-level** (`.vscode/mcp.json`) — shared with the team (no secrets). Checked into the repo.

Secrets-bearing servers → user scope. Safe-to-share configurations → workspace scope, in source control.

> **The #1 setup mistake.** VS Code’s `mcp.json` uses the root key `"servers"`. Claude Desktop and Cursor use `"mcpServers"`. Copying a Claude/Cursor config straight into VS Code without changing this key is the single most common MCP setup bug on our team. If your servers aren’t appearing, check this first.

### 7.2 Servers Worth Setting Up

Priority order for most of us:

|Server                             |What It Does                                                                                                                                      |
|-----------------------------------|--------------------------------------------------------------------------------------------------------------------------------------------------|
|**GitHub MCP** (official, remote)  |Search issues, PRs, code across our enterprise. Create issues. Read Actions runs. Hosted by GitHub at `https://api.githubcopilot.com/mcp/`.       |
|**Azure MCP** (Microsoft, official)|Inspect resources, check policy, query Resource Graph, read costs. Published as `@azure/mcp`.                                                     |
|**Microsoft SQL MCP**              |Query schema, run read-only queries, introspect indexes and plans. Several implementations exist (see 7.4); the COE publishes the approved choice.|
|**AWS MCP**                        |Equivalent for AWS workloads.                                                                                                                     |
|**Filesystem MCP**                 |Scoped access to a folder — good for a docs or ADR tree.                                                                                          |
|**Fetch MCP**                      |Pulls web pages during a Chat — vendor docs, RFCs, etc.                                                                                           |

### 7.3 Example `.vscode/mcp.json` for the Data Platform Workspace

```json
{
  "inputs": [
    { "id": "sql-conn", "type": "promptString", "description": "Dev SQL conn string", "password": true }
  ],
  "servers": {
    "github": {
      "type": "http",
      "url": "https://api.githubcopilot.com/mcp/",
      "headers": {
        "X-MCP-Readonly": "true"
      }
    },
    "azure": {
      "command": "npx",
      "args": ["-y", "@azure/mcp@latest", "server", "start"]
    },
    "mssql-dev": {
      "command": "npx",
      "args": ["-y", "<approved-mssql-mcp-package>"],
      "env": {
        "MSSQL_CONNECTION_STRING": "${input:sql-conn}",
        "MSSQL_READONLY": "true"
      }
    },
    "filesystem-adrs": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-filesystem", "${workspaceFolder}/adrs"]
    }
  }
}
```

VS Code prompts for the connection string the first time and remembers it for the session.

> **Why `<approved-mssql-mcp-package>` instead of a concrete package name?** There are several community-maintained SQL Server MCP servers and no single “official Microsoft” one published under a canonical npm name at this time. The COE maintains the approved choice in the Approved MCP Registry — pin to that value. Don’t blind-install from npm search results.

### 7.4 A Note on GitHub MCP Options

The remote GitHub MCP server (hosted by GitHub) is the easiest path — no install, OAuth handles auth. For environments where remote hosting isn’t acceptable, a local version (Docker image `ghcr.io/github/github-mcp-server`) is also available. Useful flags:

- `--read-only` (or header `X-MCP-Readonly: true`) — read-only tools only.
- `--dynamic-toolsets` (or `X-MCP-Toolsets`) — enables toolset selection on demand, reducing tool-count noise in the model context.

### 7.5 Safety Rules (Non-Negotiable)

- **Start read-only.** Every SQL, Azure, AWS, or GitHub MCP server should start in read-only mode. Elevate only when you need writes, and document why in an ADR or PR.
- **Non-production targets first.** Dev databases, sandbox subscriptions, scratch AWS accounts.
- **No shared tokens.** Use your own credentials; never check them into a repo. Use VS Code’s `inputs` prompt pattern (shown above) or user-scope configuration for anything sensitive.
- **Check the approved registry.** Our enterprise maintains a list of approved MCP servers. If the server you want isn’t there, ask the COE before using it on sensitive targets.
- **Prefer remote MCP servers with SSO over local servers with static tokens** for any target that touches regulated data (CJIS, PHI, PII, etc.).

### 7.6 Reviewing Tools

For each MCP server, VS Code shows the individual tools it exposes. You can enable or disable tools per server. Default to enabling only what you need.

-----

## 8. Extensions by Role

Keep each profile lean. **Core-everywhere** extensions are already listed in section 4.2. Below is what goes into each domain profile.

### SQL & Data Platform

- `ms-mssql.mssql` (primary SQL tool)
- SQL Database Projects
- Data Workspace
- Kusto (if you work with ADX or Log Analytics)
- dbt Power User (if dbt is in scope)
- Database Client (Weijan Chen) — secondary visual client
- SQL Formatter

### Cloud & IaC

- Azure Account, Azure Resources, Azure Resource Manager Tools
- Bicep
- Azure Policy
- HashiCorp Terraform
- AWS Toolkit
- Kubernetes
- Docker
- YAML (Red Hat) — configure schemas for GitHub Actions, Azure Pipelines, k8s

### PowerShell & Automation

- PowerShell (Microsoft)
- Pester Test Explorer

### DevSecOps

- GitHub Actions
- GitHub Pull Requests
- GitHub Repositories
- Snyk
- Checkov
- Trivy
- Open Policy Agent
- Dev Containers

### Docs & Architecture

- Markdown All in One
- markdownlint
- Mermaid Preview
- Draw.io Integration (hediet.vscode-drawio)
- PlantUML
- Excalidraw
- Paste Image

**Prune ruthlessly.** Every installed extension is context Copilot wades through and startup time you pay.

-----

## 9. Settings and Repo Scaffolding

### 9.1 User `settings.json`

Open the command palette (`Ctrl+Shift+P`) → “Preferences: Open User Settings (JSON)”:

```json
{
  "editor.formatOnSave": true,
  "editor.codeActionsOnSave": { "source.fixAll": "explicit" },
  "files.trimTrailingWhitespace": true,
  "files.insertFinalNewline": true,
  "files.eol": "\n",
  "terminal.integrated.defaultProfile.windows": "PowerShell",
  "git.autofetch": true,
  "git.confirmSync": false,
  "github.copilot.chat.codeGeneration.useInstructionFiles": true,
  "chat.mcp.enabled": true
}
```

The `files.eol` setting matters. Mixed line endings have caused real bugs on our team (StreamWriter encoding, BCP output). Enforce `\n` universally.

### 9.2 Per-Repo `.gitattributes`

```
* text=auto eol=lf
*.ps1 text eol=crlf
*.sql text eol=crlf
```

This prevents line-ending churn in PRs. PowerShell and T-SQL tooling on Windows genuinely prefers CRLF; everything else is LF.

### 9.3 Per-Repo `.editorconfig`

```ini
root = true

[*]
indent_style = space
indent_size = 4
end_of_line = lf
charset = utf-8
trim_trailing_whitespace = true
insert_final_newline = true

[*.md]
trim_trailing_whitespace = false

[*.{yml,yaml,json}]
indent_size = 2
```

### 9.4 Recommended Repo Scaffold

For every new repo:

```
.github/
  copilot-instructions.md
  CODEOWNERS
  pull_request_template.md
  workflows/
    ci.yml
    codeql.yml
  prompts/
    review-tsql.prompt.md
    harden-powershell.prompt.md
.vscode/
  settings.json
  extensions.json
  mcp.json
.editorconfig
.gitattributes
.gitignore
README.md
SECURITY.md
adrs/
  0001-record-architecture-decisions.md
```

Our org publishes repo templates with this scaffold pre-wired. Use those instead of copying manually.

-----

## 10. Day-in-the-Life Examples

### 10.1 Fixing a Bug in the Backup Script

1. Open `data-platform.code-workspace`.
1. VS Code offers to switch to the SQL & Data Platform profile. Accept.
1. Recommended extensions prompt. Install if missing.
1. Copilot Chat opens with the repo’s instruction file already loaded.
1. Attach the **Data Platform Standards** Space for grounding.
1. Ask: “`@workspace` why is the BCP call returning NativeCommandError on rows with embedded quotes?”
1. Copilot answers grounded in the repo’s conventions and our Space, not a generic Stack Overflow thread.

### 10.2 Reviewing a Control Mapping

1. Open `cloud-governance.code-workspace`.
1. Switch to the Cloud & IaC profile.
1. Azure MCP is running. Ask Chat: “For each row in the mapping matrix, verify the listed Azure Policy definition actually exists in our tenant.”
1. Copilot queries Azure via MCP, reports findings.
1. Edit mode applies corrections to the matrix.

### 10.3 Drafting a COE Policy

1. Open `devsecops-coe.code-workspace`.
1. Switch to the Docs & Architecture profile.
1. In Chat, attach the **DevSecOps COE Patterns** Space. Ask: “Draft a standard for signed commits on repos with `compliance-scope = cjis`.”
1. Answer is grounded in our existing standards.
1. Run `/generate-adr` prompt to format as ADR-nnnn.

### 10.4 Delegating an Issue to the Coding Agent

1. In GitHub, pick a low-risk issue (a doc typo, a small refactor, a missing test).
1. Assign it to Copilot. Make sure the issue description has enough context — the agent’s only input is what’s written there plus the repo.
1. Check Mission Control for progress.
1. When the PR opens, review it like any other contribution. Leave feedback if needed; the agent can iterate.
1. Merge on success.

-----

## 11. Troubleshooting and Gotchas

### 11.1 Copilot Suggestions Feel Generic

Check:

- Does the repo have a `.github/copilot-instructions.md`?
- Is `github.copilot.chat.codeGeneration.useInstructionFiles` set to `true`?
- Are you in a multi-root workspace if the question spans repos?
- Have you attached a Space for broader grounding?

### 11.2 Copilot Can’t See the Repo

If `@workspace` gives weak answers:

- Confirm the repo is actually added to your workspace (check the left-pane folder list).
- Large repos take time to index on first open. Wait a few minutes.
- For very large monorepos, the inline-completion context window is limited. Consider a Space over selected folders as a complement.

### 11.3 MCP Server Won’t Start

- Check the Output panel → MCP (or use the “show output” link in the Chat error banner) for error messages.
- Confirm the command (`npx`, `node`, `uvx`, etc.) is in PATH in the environment VS Code launched it from.
- Most MCP servers need Node 20+ or Python 3.11+.
- Root key mismatch: remember, VS Code uses `"servers"`, not `"mcpServers"`.
- If changes to `mcp.json` don’t take effect, reload the window (`Developer: Reload Window`).

### 11.4 Git Credential Manager Picks the Wrong Account

- Check `git config --global --list | grep credential`.
- Clear credentials: `git credential-manager erase` then `https://github.com` + Enter twice.
- Re-clone a repo and let it prompt for the right account.

### 11.5 Extensions Slow Down Startup

- Command palette → “Developer: Show Running Extensions” shows load times.
- Move heavy extensions out of the Default profile into a domain profile.

### 11.6 Line-Ending Warnings on Every Commit

Section 9.2. Add `.gitattributes`. Commit it. Run `git add --renormalize .` to re-normalize the working tree.

### 11.7 Premium Request Quota Warnings

If Chat starts warning about quota:

- Check the Copilot usage page for current consumption.
- Switch to a faster model for routine questions; reserve the strongest reasoning model for hard problems.
- Agent-mode and coding-agent sessions consume more per interaction than Ask-mode questions — budget accordingly.

-----

## 12. Glossary — Plain Language

**ADR.** A short document recording one architecture decision. Numbered. Immutable once accepted.

**Agent mode.** Copilot Chat mode where it can read, edit, and run tools across multiple steps within the editor session.

**CODEOWNERS.** File that lists who reviews changes to specific paths. Auto-requests them on PRs.

**Coding Agent.** Copilot feature where you assign a GitHub issue to Copilot and it works autonomously in the background, opening a PR when done. Managed through Mission Control.

**Copilot Space.** An enterprise-curated (or individually-curated) container of code, docs, images, issues, and custom instructions that Copilot grounds answers in. Replaced the older “knowledge bases” feature on November 1, 2025.

**Dev container.** A container that defines the development environment for a repo. You open the repo “in” the container and everyone gets the same tools.

**EMU.** Enterprise Managed User. Our enterprise GitHub accounts are EMU accounts, not your personal GitHub.

**Inner source.** Open-source-style collaboration inside an enterprise. Repos are internal, but contributions work like public open source.

**Instruction file.** `.github/copilot-instructions.md` in a repo. Automatically loaded into every Copilot Chat turn for that repo.

**MCP.** Model Context Protocol. Lets Copilot talk to real systems (GitHub, SQL, Azure, etc.).

**Mission Control.** A dashboard for assigning, steering, and tracking Copilot coding-agent tasks across multiple issues at once.

**Multi-root workspace.** A `.code-workspace` file grouping several folders into one VS Code window.

**Premium request.** Copilot’s unit of paid usage for Chat, agent mode, code review, and model selection. Plans include a monthly allocation.

**Profile (VS Code).** A bundle of extensions, settings, and keybindings. Switch profiles to switch contexts.

**Prompt file.** `.github/prompts/<name>.prompt.md`. Reusable Chat prompt invokable with `/<name>`.

**Reusable workflow.** A GitHub Actions workflow defined once, called from many repos. How shared CI/CD logic is delivered.

**WSL.** Windows Subsystem for Linux. Linux running alongside Windows.

-----

## 13. Where to Ask Questions

- Team channel: post in our main channel.
- Cross-team questions: DevSecOps COE channel.
- Security-specific: Enterprise Security channel / Enterprise Information Security Office (EISO).
- GitHub admin issues: our admin rotation.
- Commonwealth IT policy questions (ITPs, Executive Order 2016-06, procurement alignment): OA/OIT channel.

Don’t self-serve on anything you’re unsure about if it touches production data, credentials, or shared tooling. Ask. Particularly for anything with `compliance-scope = cjis` or that processes data classified as restricted under Commonwealth ITP policy.

-----

## 14. What’s Next After This Guide

Once you have the basics working, explore in this order:

1. **Write your own instruction file** for a repo you own. See how Copilot behavior changes.
1. **Add one MCP server** (start with the remote GitHub MCP or Fetch — no credentials risk).
1. **Build a multi-root workspace** for something you work on regularly.
1. **Contribute a prompt file** to the shared repo. Small, reusable, saves everyone time.
1. **Create a personal Space** for an area you know well, then promote it to an org-shared Space when it’s useful to others.
1. **Read the companion *Enterprise Multi-Org DevSecOps Strategy* document** for the bigger picture.

You do not need to master all of this in a week. Small steps. The team is the point, not the tools.

-----

*Maintained by our team. Suggestions via PR. Updates tracked in the changelog.*