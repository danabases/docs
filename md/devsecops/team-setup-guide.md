# Team Setup Guide

## VS Code, Copilot, MCP, and Workspaces — Draft v3

**Purpose.** A portable, step-by-step guide for our team and all collaborating teams within Enterprise Technology Services (ETS) and adjacent teams. Takes you from a fresh Windows machine to a working, consistent environment. Written for any skill level — no prior VS Code, Copilot, or MCP experience assumed.

**Audience.** Data platform / DBA (Database Administrator), Platform Engineering, IAM (Identity and Access Management), SecOps (Security Operations), Networking/Firewall, Backup, Automation, and adjacent teams collaborating through our shared GitHub org.

**How to use this.** Read §1 first. Do §2. Then pick the sections matching your role. The full setup takes a few hours the first time.

**Change log from v2:**

- Removed Azure Data Studio (retired by Microsoft — see §2.3)
- Audited and updated all tool recommendations
- Added Docker Desktop licensing note with alternatives
- Corrected branch strategy (§6)
- Added cross-delivery workspace framework (§5)
- Added on-prem tooling extensions: Ansible, Salt, VMware (§9)
- Added teams and permissions setup steps (§10)

-----

## 0. Table of Contents

1. Why we do this
1. Install the basics — Windows, WSL, Git, VS Code
1. Sign in and verify Copilot access
1. VS Code Profiles — what they are and how we use them
1. Workspaces — cross-team and cross-delivery framework
1. Branch strategy — how we use `main` and branches
1. The files that shape your experience
1. Copilot — getting more out of it
1. Extensions by role
1. Teams and permissions setup
1. Settings and repo scaffolding
1. Day-in-the-life examples
1. Troubleshooting and gotchas
1. Glossary
1. Where to ask questions
1. What to do next

-----

## 1. Why We Do This

Most of us have VS Code installed and use Copilot casually. This guide creates a setup that is **consistent across all ETS teams**, **tuned to our specific work**, and **portable** — so any teammate can be productive on day one.

**The goal is not to install everything. The goal is to make the right choices the default.**

```
FOUR THINGS THAT DELIVER MOST OF THE VALUE:

  1. PROFILES       → Right extensions load for the right work.
                       Copilot gives better suggestions without
                       irrelevant tools cluttering the context.

  2. WORKSPACES     → Group related repos across teams so Copilot
                       can see all relevant code when answering.
                       Critical for cross-team collaboration.

  3. INSTRUCTION    → Teach Copilot our conventions so it suggests
     FILES            code that matches how we actually work,
                       not generic internet examples.

  4. MCP            → Connect Copilot to live systems: SQL Server
                       schemas, Azure resources, GitHub issues,
                       VMware Aria inventory, CMDB data.
```

-----

## 2. Install the Basics

### 2.1 What Goes Where — Windows vs WSL

**WSL** = Windows Subsystem for Linux. Runs Ubuntu inside Windows. Not needed for everything — be deliberate.

```
KEEP ON WINDOWS                          PUT IN WSL (Ubuntu)
──────────────────────────────────       ──────────────────────────────────
VS Code (the editor itself)              MCP servers (run better on Linux)
PowerShell 7 (automation / SQL)         Container security scanning
Git + GCM (credential manager)           (Trivy, Grype)
SQL Server tools (SSMS, sqlcmd)         IaC security scanners (Checkov,
Azure CLI, AWS CLI                        tflint, KICS)
Windows-authenticated SQL connections   act (test GitHub Actions locally)
SSDT / SQL Database Projects            Node.js / Python dev tools
BCP (Bulk Copy Program)                  (where Windows paths cause issues)
PowerCLI (VMware PowerShell module)
```

### 2.2 Core Install — Windows

Open PowerShell (search “PowerShell” in Start — Admin if available):

```powershell
# Git — version control system
winget install --id Git.Git -e

# PowerShell 7.x — current version; keep 5.1 for legacy tools only
winget install --id Microsoft.PowerShell -e

# VS Code — the editor
winget install --id Microsoft.VisualStudioCode -e
```

Direct download links if winget is unavailable:

- Git: https://git-scm.com/download/win
- PowerShell 7: https://aka.ms/powershell
- VS Code: https://code.visualstudio.com/

### 2.3 Tool-by-Tool Reference — What to Install and Why

#### SQL Server and Data Tools

```powershell
# SSMS (SQL Server Management Studio) — primary GUI for SQL Server
# Still actively maintained by Microsoft. Install this.
winget install --id Microsoft.SQLServerManagementStudio -e

# NOTE: Azure Data Studio has been RETIRED by Microsoft.
# Do NOT install it. It will not receive updates or bug fixes.
# VS Code with the mssql extension covers most of what ADS did.
# For richer multi-database visual browsing, see DBeaver below.

# sqlcmd — the modern Go-based version (not the legacy C++ version)
# The new version supports MFA, service principals, and JSON output
winget install --id Microsoft.SQLCmd -e

# SqlPackage — for DACPAC (database schema package) deployments
winget install --id Microsoft.SqlPackage -e
```

**What to use instead of Azure Data Studio:**

|What you did in ADS           |Use instead                                               |
|------------------------------|----------------------------------------------------------|
|Query editor with results grid|VS Code + mssql extension (same team, actively maintained)|
|Notebook-style SQL            |VS Code + mssql extension supports notebooks              |
|Multi-database browsing       |DBeaver Community (free, excellent)                       |
|Schema compare                |VS Code SQL Database Projects extension                   |
|SSDT-style projects           |VS Code SQL Database Projects extension                   |

**DBeaver Community** (free, cross-platform, supports SQL Server, PostgreSQL, MySQL, Oracle, SQLite, and more):

```powershell
winget install --id DBeaver.DBeaver -e
```

#### Cloud Tools

```powershell
# Azure CLI — command-line tool for Azure
winget install --id Microsoft.AzureCLI -e

# AWS CLI v2 — command-line tool for Amazon Web Services
winget install --id Amazon.AWSCLI -e
```

#### Container Runtime

```powershell
# Docker Desktop — container runtime
# IMPORTANT: As of 2022, Docker Desktop requires a paid subscription
# for organizations with >250 employees or >$10M annual revenue.
# Check your enterprise license before installing.
winget install --id Docker.DockerDesktop -e

# FREE ALTERNATIVES (no subscription required):
# Podman Desktop — Docker-compatible, rootless, open-source
winget install --id RedHat.Podman-Desktop -e

# Rancher Desktop — includes containerd + nerdctl + optional dockerd
# Download from: https://rancherdesktop.io/
```

#### SqlServer PowerShell Module

```powershell
# Run in PowerShell 7 (not Windows PowerShell 5.1)
Install-Module -Name SqlServer -Scope CurrentUser -Force
```

### 2.4 WSL2 and Ubuntu

```powershell
# Install WSL2 with Ubuntu 24.04 LTS
wsl --install -d Ubuntu-24.04
```

Restart when prompted. Create a username and password inside Ubuntu — these are separate from your Windows credentials.

### 2.5 Tools Inside WSL

Open Ubuntu terminal for these:

```bash
# Node.js via nvm (Node Version Manager)
curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.39.7/install.sh | bash
# Close and reopen terminal, then:
nvm install --lts

# Python via uv (fast Python package/environment manager)
curl -LsSf https://astral.sh/uv/install.sh | sh

# Trivy — scans containers, file systems, IaC for vulnerabilities
# Now covers Terraform scanning (replaces standalone tfsec for most cases)
curl -sfL https://raw.githubusercontent.com/aquasecurity/trivy/main/contrib/install.sh \
  | sudo sh -s -- -b /usr/local/bin

# Checkov — IaC security scanner (Terraform, Bicep, CloudFormation, Ansible)
pip3 install checkov --user

# tflint — Terraform linter (catches provider-specific issues)
curl -s https://raw.githubusercontent.com/terraform-linters/tflint/master/install_linux.sh \
  | bash

# cfn-lint — AWS CloudFormation template linter
pip3 install cfn-lint --user

# KICS (Keeping Infrastructure as Code Secure) — multi-IaC scanner
# Download from: https://github.com/Checkmarx/kics/releases
# Or via Docker: docker run -v $(pwd):/path checkmarx/kics scan -p /path

# act — run GitHub Actions workflows locally for testing
curl https://raw.githubusercontent.com/nektos/act/master/install.sh \
  | sudo bash

# Ansible — agentless configuration management
pip3 install ansible --user
ansible --version  # verify

# ansible-lint — linter for Ansible playbooks and roles
pip3 install ansible-lint --user

# ansible-navigator — modern Ansible runner with execution environments
pip3 install ansible-navigator --user

# salt-lint — linter for SaltStack state files
pip3 install salt-lint --user

# Terraform (if needed inside WSL)
# See: https://developer.hashicorp.com/terraform/install#linux
```

**Note on tfsec:** Aqua Security (which maintains both Trivy and tfsec) has integrated Terraform scanning into Trivy. For new setups, Trivy alone covers Terraform IaC scanning. tfsec is still maintained and usable but Trivy is the forward-looking choice.

### 2.6 VMware / PowerCLI (Windows-side)

PowerCLI (VMware’s PowerShell module) must run on Windows because it uses Windows-native COM objects for some operations:

```powershell
# Install PowerCLI — VMware's PowerShell management module
Install-Module -Name VMware.PowerCLI -Scope CurrentUser
Set-PowerCLIConfiguration -InvalidCertificateAction Ignore -Confirm:$false

# Verify
Get-Module -Name VMware.PowerCLI -ListAvailable
```

### 2.7 Git Credential Manager — Prevent Account Confusion

**GCM (Git Credential Manager)** stores authentication tokens. Without correct configuration, it will use the wrong GitHub account on machines with both personal and enterprise EMU accounts.

**EMU** = Enterprise Managed User. Our enterprise GitHub accounts are EMU accounts — provisioned by our identity provider, not created personally. They look like `your_name_enterpriseslug` (note the underscore and enterprise suffix).

In your Windows `~/.gitconfig` (open with `notepad $HOME/.gitconfig`):

```gitconfig
[credential "https://github.com"]
    provider = github
    useHttpPath = true

[credential "https://github.com/YourEnterpriseSlug"]
    username = your_name_enterpriseslug
```

**Share GCM with WSL** (so you don’t authenticate twice):

```bash
# In WSL Ubuntu terminal:
git config --global credential.helper \
  "/mnt/c/Program\ Files/Git/mingw64/bin/git-credential-manager.exe"
```

**Verify:** Clone one of our enterprise repos. It should open a browser, sign in once with the EMU account, and never prompt again.

-----

## 3. Sign In and Verify Copilot

**Copilot Enterprise** is GitHub’s AI coding assistant at the enterprise tier. It includes inline suggestions, Chat (conversation interface), agent mode (multi-step autonomous editing), and knowledge bases (our own docs that ground its answers).

1. Open VS Code. Click the account icon (person silhouette) in the bottom-left.
1. Choose “Sign in with GitHub.” Sign in with your **EMU account** (the `name_enterpriseslug` format).
1. Install these extensions (`Ctrl+Shift+X` to open extensions):
- **GitHub Copilot** (`github.copilot`) — inline suggestions
- **GitHub Copilot Chat** (`github.copilot-chat`) — chat, agent, `@workspace`
1. Verify: Open a code file, type a function name, wait 2 seconds for a gray suggestion. Then `Ctrl+Alt+I` to open Chat and ask something.

If it doesn’t work: `Settings → GitHub → Copilot` — confirm the enterprise account is signed in.

-----

## 4. VS Code Profiles

### 4.1 What a Profile Is

A **profile** is a bundle of extensions, settings, keybindings, snippets, and UI state. Switching profiles is instant. The alternative — one VS Code with every extension installed — is slow and gives Copilot irrelevant context.

### 4.2 Profile Decision Tree

```
What are you working on?
│
├── Quick edit, README, ad-hoc file
│   └── DEFAULT profile (minimal)
│
├── SQL Server, databases, BCP, data migrations
│   └── SQL & DATA PLATFORM profile
│
├── Azure, AWS, Bicep, Terraform, Kubernetes
│   └── CLOUD & IaC profile
│
├── VMware, on-prem infra, Ansible, Salt, PowerCLI
│   └── ON-PREM PLATFORM profile
│
├── PowerShell scripts, modules, automation
│   └── POWERSHELL & AUTOMATION profile
│
├── GitHub Actions, security scanning, OPA policies
│   └── DEVSECOPS profile
│
├── IAM policy, Entra ID, SSO config, access reviews
│   └── IDENTITY & ACCESS (IAM) profile
│
└── Architecture docs, ADRs, Mermaid diagrams
    └── DOCS & ARCHITECTURE profile
```

Create profiles: `File → Profiles → New Profile`

### 4.3 Core Extensions — Install in Every Profile

|Extension          |ID                                     |Purpose                                    |
|-------------------|---------------------------------------|-------------------------------------------|
|GitHub Copilot     |`github.copilot`                       |Inline AI suggestions                      |
|GitHub Copilot Chat|`github.copilot-chat`                  |Chat, agent mode, `@workspace`             |
|GitLens            |`eamodio.gitlens`                      |Enhanced Git history, blame, comparison    |
|Error Lens         |`usernamehw.errorlens`                 |Error/warning messages inline on the line  |
|EditorConfig       |`editorconfig.editorconfig`            |Applies `.editorconfig` rules automatically|
|Code Spell Checker |`streetsidesoftware.code-spell-checker`|Catches typos in code and comments         |
|Markdown All in One|`yzhang.markdown-all-in-one`           |Preview and shortcuts for `.md` files      |
|Remote - WSL       |`ms-vscode-remote.remote-wsl`          |Open WSL folders in VS Code                |
|Remote - SSH       |`ms-vscode-remote.remote-ssh`          |Open remote server folders                 |

-----

## 5. Workspaces — Cross-Team and Cross-Delivery Framework

### 5.1 What a Workspace Is

A **workspace** (`.code-workspace` file) groups multiple repos into one VS Code window. When you use `@workspace` in Copilot Chat, it searches all repos in that workspace. This is the primary mechanism for cross-team collaboration context.

### 5.2 The Cross-Delivery Problem

Data platform, platform engineering, IAM, SecOps, networking, and backup all share the same GitHub org and often work on interconnected systems. Without workspaces, each team works in their own repo bubble. Questions like “how does the backup automation interact with the SQL service account that IAM provisions?” can’t be answered by Copilot because both repos aren’t open at the same time.

Cross-delivery workspaces solve this.

### 5.3 Workspace Types

```
WORKSPACE TAXONOMY

Stable (permanent, live in the shared workspaces repo)
│
├── Single-team workspaces
│   ├── data-platform.code-workspace
│   │   Repos: sql-backup-automation, bcp-pipeline,
│   │           shared-ps-modules, data-platform-adrs
│   │
│   ├── platform-engineering.code-workspace
│   │   Repos: landing-zone-azure, landing-zone-aws,
│   │           vmware-iac, shared-bicep-modules,
│   │           shared-terraform-modules
│   │
│   ├── iam.code-workspace
│   │   Repos: identity-policy, entra-id-config,
│   │           access-review-automation, pam-config
│   │
│   ├── networking.code-workspace
│   │   Repos: firewall-rules, nsx-iac,
│   │           dns-automation, network-baselines
│   │
│   └── devsecops-coe.code-workspace
│       Repos: coe-decisions, reusable-workflows,
│               template-library, docs-site
│
├── Cross-delivery workspaces (permanent — active partnerships)
│   │
│   ├── data-platform-partnership.code-workspace
│   │   Repos: [data repos] + [platform repos] + [iam repos]
│   │   When: Data team working on DB server provisioning
│   │         that involves IAM service accounts and Platform
│   │         landing zone configuration
│   │
│   ├── agency-onboarding-framework.code-workspace
│   │   Repos: [template library] + [landing zones] +
│   │           [security baselines] + [tag validation]
│   │   When: Building or improving the agency self-service process
│   │
│   └── security-integration.code-workspace
│       Repos: [security baselines] + [COE artifacts] +
│               [platform landing zones] + [iam policy]
│       When: Ongoing security posture alignment work
│
└── Short-lived workspaces (created for a specific effort, archived after)
    ├── customer-onboarding-<agency>.code-workspace
    ├── coe-workstream-<topic>.code-workspace
    ├── incident-response-<date>.code-workspace
    └── platform-migration-<project>.code-workspace
```

### 5.4 Cross-Delivery Workspace Design: Data + Platform + IAM

This example shows the most common cross-team workspace configuration — data platform engineers working alongside platform and IAM:

```json
{
  "folders": [
    {
      "name": "🗄️ SQL Backup Automation",
      "path": "../repos/sql-backup-automation"
    },
    {
      "name": "📦 BCP Export Pipeline",
      "path": "../repos/bcp-export-pipeline"
    },
    {
      "name": "🏗️ Azure Landing Zone",
      "path": "../repos/landing-zone-azure"
    },
    {
      "name": "🔑 IAM Service Accounts",
      "path": "../repos/identity-policy"
    },
    {
      "name": "🔧 Shared PS Modules",
      "path": "../repos/shared-ps-modules"
    }
  ],
  "settings": {
    "github.copilot.chat.codeGeneration.useInstructionFiles": true,
    "files.exclude": { "**/bin": true, "**/obj": true }
  },
  "extensions": {
    "recommendations": [
      "ms-mssql.mssql",
      "ms-mssql.sql-database-projects-vscode",
      "ms-vscode.powershell",
      "ms-azuretools.vscode-bicep",
      "github.copilot",
      "github.copilot-chat",
      "eamodio.gitlens"
    ]
  }
}
```

With this workspace open:

```
@workspace — how is the gMSA (Group Managed Service Account)
provisioned by the IAM team, and how does the backup automation
use it to connect to SQL Server?

Copilot searches all five repos and answers from the actual code
in identity-policy AND sql-backup-automation simultaneously.
```

### 5.5 Partnership Workspace Rules

**Who creates them:** Any team can create a cross-delivery workspace. The workspace file lives in the `workspaces` repo which all ETS teams have read access to.

**What goes in them:** Repos that are *actively* related for a joint effort. Don’t add every repo “just in case” — Copilot’s quality drops when the context is too broad.

**Who maintains them:** The team that initiated the collaboration. Archive when the joint effort ends.

**Branch strategy in workspace files:** The workspace file itself lives on `main` in the workspaces repo. Workspace file changes go through a PR — don’t edit workspace files directly on `main`.

-----

## 6. Branch Strategy — How We Use `main`

### 6.1 `main` Is the Production Version of Truth

In every repo we own, **`main` is the production-ready branch.** It is always deployable, always reviewed, always passing checks. It is what others clone, reference, and deploy from.

```
STANDARD DEVELOPMENT FLOW

  feature/add-retention-policy   ────── PR (reviewed + checks) ──► main
  fix/bcp-quote-handling         ────── PR (reviewed + checks) ──► main
  hotfix/cjis-audit-gap          ────── PR (reviewed + checks) ──► main

  main is always:
    ✓ Passing all required CI checks
    ✓ Reviewed and approved (CODEOWNERS + branch protection)
    ✓ Deployable to production
    ✓ The thing you clone to start work
    ✓ The version templates reference and pipelines deploy

  Nobody develops directly on main.
  Nobody force-pushes to main.
  No exceptions — including admins.
```

**Releases are tagged on `main`:** When you publish a version of a shared module or reusable workflow, you tag the commit on `main` (`v1.0.0`, `v1.1.0`). The tag is the release; `main` is always the latest production state.

### 6.2 When Version Pinning Applies (Consuming vs. Producing)

This is the distinction that caused confusion in earlier versions of this guide:

```
YOU OWN THE REPO (you are the producer):
  main = truth. Feature → PR → main. Tag releases from main.
  ✓ This is how all our repos work.

YOU ARE CALLING SOMEONE ELSE'S SHARED WORKFLOW (consumer):
  # Fragile — their next commit breaks you with no warning:
  uses: enterprise-security/security/.github/workflows/security-scan.yml@main

  # Stable — you update on your schedule:
  uses: enterprise-security/security/.github/workflows/security-scan.yml@v3
```

**Why pin when consuming?** If Enterprise Security pushes a breaking change to their `main`, every repo calling `@main` breaks immediately — no warning, no test period, no choice. Pinning to `@v3` means their `main` changes don’t affect you until you explicitly update your reference and test it.

This is exactly how package managers work: your project’s `main` branch is your production code; your `package.json` pins `"lodash": "4.17.21"`, not `"latest"`. Their `main` branch is not your business until you decide to update.

**Summary:**

|You are                          |Branch     |Rule                                         |
|---------------------------------|-----------|---------------------------------------------|
|Producer (your repo)             |Your `main`|Truth. Protect it. Require PRs. Tag releases.|
|Consumer (calling their workflow)|Their `@v3`|Pin to a version tag, not their `@main`      |

### 6.3 Branch Naming Convention

```
Type         Prefix    Example
──────────── ────────  ──────────────────────────────────
Feature      feature/  feature/add-monthly-backup-rotation
Bug fix      fix/      fix/bcp-null-handling
Hotfix       hotfix/   hotfix/cjis-audit-log-gap
Release prep release/  release/v2.1.0
Experiment   exp/      exp/salt-integration-test
```

### 6.4 Required Branch Protection on `main`

```
Required branch protection for every repo (minimum):
  ✓ Require pull request before merging
  ✓ Require 1 approving review (2 for compliance-scoped repos)
  ✓ Dismiss stale approvals when new commits are pushed
  ✓ Require status checks to pass before merging
      (security-scan, secret-scan, build, lint — not CodeQL
       unless the repo contains C#, Python, or JavaScript)
  ✓ Require branches to be up to date before merging
  ✗ Do NOT allow bypassing these settings (not even for admins)
```

The “no bypassing” rule applies to everyone. If an admin needs to make an emergency change, they still open a PR — it just gets expedited review, not a bypass.

-----

## 7. The Files That Shape Your Experience

### 7.1 File Map

```
A well-configured repo:

repo/
├── .github/
│   ├── copilot-instructions.md      ← Copilot reads this for every Chat turn
│   ├── CODEOWNERS                   ← Auto-requests reviewers on matching PRs
│   ├── pull_request_template.md     ← Checklist every PR author sees
│   ├── ISSUE_TEMPLATE/
│   │   ├── bug_report.md
│   │   └── feature_request.md
│   ├── workflows/
│   │   ├── ci.yml                   ← Build and test
│   │   ├── security-scan.yml        ← Semgrep + PSScriptAnalyzer (where applicable)
│   │   ├── secret-scan.yml          ← Gitleaks custom patterns (GitHub Secret Scanning always on)
│   │   ├── iac-scan.yml             ← Checkov + Trivy (IaC repos only)
│   │   └── tag-validation.yml       ← Required tag check
│   └── prompts/
│       └── [role-specific prompts]
│
├── .vscode/
│   ├── settings.json                ← Repo-level VS Code settings
│   ├── extensions.json              ← Recommended extensions
│   └── mcp.json                     ← MCP config (no secrets)
│
├── adrs/
│   └── 0001-record-architecture-decisions.md
│
├── .editorconfig                    ← Cross-editor formatting
├── .gitattributes                   ← Line ending rules
├── .gitignore                       ← Files Git never tracks
├── README.md                        ← What this is and how to use it
└── SECURITY.md                      ← How to report a vulnerability
```

### 7.2 Understanding Each File

**`.github/copilot-instructions.md`** — Copilot reads this automatically for every Chat turn in this repo. It shapes suggestions and generated code to match your conventions. Fill this in for every repo you own. See §8.2 for templates.

**`.github/CODEOWNERS`** — Lists who reviews changes to specific file paths. When a PR touches a matching path, GitHub automatically requests review from the listed team. Example:

```
/.github/workflows/     @ets-secops @ets-devsecops-coe
/**/*.sql               @ets-data/dba-team
/infra/                 @ets-platform
/adrs/                  @ets-devsecops-coe
SECURITY.md             @ets-secops
```

**`.vscode/extensions.json`** — When anyone opens this repo, VS Code prompts them to install listed extensions:

```json
{
  "recommendations": [
    "ms-mssql.mssql",
    "ms-vscode.powershell",
    "github.copilot",
    "github.copilot-chat"
  ]
}
```

**`.vscode/mcp.json`** — MCP server configuration. Safe to commit (no secrets). See §X for examples.

**`.gitattributes`** — Prevents line-ending mismatches between Windows and Linux teammates. Without this, files appear changed even when content is identical.

-----

## 8. Copilot — Getting More Out of It

### 8.1 The Value Ladder

```
  Basic use (inline suggestions as you type)
       │ +
  Chat (@workspace searches across all open repos)
       │ +
  Instruction files (Copilot follows your conventions)
       │ +
  Knowledge bases (answers grounded in our own standards)
       │ +
  MCP servers (Copilot sees live SQL, Azure, GitHub, VMware)
       ↓
  Maximum value for our kind of work
```

### 8.2 Instruction File Templates by Role

**For SQL / Data Platform repos:**

```markdown
# Copilot Instructions — [Repo Name]

## Context
PowerShell 7 module targeting SQL Server 2019/2022.
Supports local, UNC, and Azure Blob destinations.
Runs under a gMSA (Group Managed Service Account) with least-privilege.

## Conventions
- PowerShell: Verb-Noun naming, approved verbs only, StrictMode v3.
  Comment-based help with at least one .EXAMPLE on all public functions.
- T-SQL: schema-qualify all objects (dbo.TableName).
  No SELECT *. Use sys.* catalog views not INFORMATION_SCHEMA.
- Error handling: try/catch around BCP, 7-Zip, and CLI calls.
  Check $LASTEXITCODE after native commands.
- No inline credentials. Use SqlCredential or Managed Identity.
- Logging: Write-Verbose for diagnostics, Write-Information for events.
  Never Write-Host.

## Don't
- Don't suggest Invoke-Sqlcmd without -TrustServerCertificate
  or a cert validation note (required for SQL Server 2022+/ODBC 18+).
- Don't swallow errors in broad try/catch without re-throwing or logging.
- Don't generate SELECT * anywhere.
```

**For Platform / IaC repos (Azure Bicep):**

```markdown
# Copilot Instructions — [Repo Name]

## Context
Bicep modules for Azure landing zone deployment.
Targets Azure Government where applicable.
Managed identity authentication only.

## Conventions
- Bicep: use modules for reusable components.
  All parameters have decorators (@description, @allowed where relevant).
  No hardcoded resource IDs — use references or parameters.
- All resources must have required tags applied via a tags parameter.
- Private endpoints required for all PaaS services.
- Diagnostic settings required on all resources (logs to Log Analytics).

## Don't
- Don't generate public endpoints on storage, SQL, Key Vault, or App Service.
- Don't hardcode subscription IDs, tenant IDs, or resource IDs.
- Don't skip the tags parameter on any resource.
```

**For On-Prem / Ansible repos:**

```markdown
# Copilot Instructions — [Repo Name]

## Context
Ansible roles and playbooks for on-premises VMware infrastructure.
Targets RHEL 8/9 and Windows Server 2019/2022.
Runs via Ansible Navigator with execution environments.
No direct SSH to production — all runs via AWX/Ansible Automation Platform.

## Conventions
- Roles: follow ansible-galaxy structure (tasks, handlers, defaults, vars).
  Idempotent — running twice should produce no changes on second run.
- Variables: use defaults/main.yml for defaults, override in host_vars/group_vars.
- No hardcoded credentials. Use ansible-vault or AAP credentials.
- Tasks: always use FQCN (Fully Qualified Collection Name):
  ansible.builtin.package not just package.
- Tags: always tag tasks for selective execution.

## Don't
- Don't use shell or command module where an Ansible module exists.
- Don't hardcode IP addresses or hostnames — use inventory variables.
- Don't store secrets in plaintext anywhere in the repo.
```

**For Salt Formula repos:**

```markdown
# Copilot Instructions — [Repo Name]

## Context
SaltStack formulas for on-premises configuration management.
Manages VMware-hosted RHEL and Windows nodes.
Integrated with VMware Aria Automation for post-provisioning config.

## Conventions
- Formulas: follow Salt formula conventions (init.sls, map.jinja).
  Idempotent — states should be safe to run repeatedly.
- Use Jinja2 templating with map.jinja for OS-specific values.
- Pillar data for all secrets and environment-specific values.
- Requisites: use require, watch, onchanges appropriately.

## Don't
- Don't hardcode values that should come from pillar.
- Don't skip requisites where order matters.
- Don't use cmd.run where a Salt module handles the task.
```

### 8.3 Prompt Files

Create `.github/prompts/<n>.prompt.md` — invoke with `/<n>` in Chat.

**`/review-tsql`:**

```markdown
Review the selected T-SQL for:
1. Sargability — are predicates index-friendly? Flag functions around
   columns in WHERE clauses, implicit type conversions.
2. NOLOCK — is it present? Is it justified with a comment?
   NOLOCK can return dirty/phantom rows. Flag unjustified use.
3. SELECT * — flag all occurrences. Columns must be explicit.
4. Schema qualification — every object must be schema.Name.
5. Implicit conversions — VARCHAR compared to INT, etc.
6. Covering index candidates — based on WHERE/JOIN patterns.
Return numbered findings with line numbers.
```

**`/review-ansible`:**

```markdown
Review the selected Ansible for:
1. Idempotency — will running this twice cause problems?
2. FQCN — are all module names fully qualified (ansible.builtin.)?
3. Hardcoded values — credentials, IPs, hostnames that should be variables.
4. Shell/command usage — is there an Ansible module for this task instead?
5. Error handling — are failed_when/changed_when used appropriately?
6. Tags — are tasks tagged for selective execution?
Return numbered findings with line numbers.
```

**`/review-terraform`:**

```markdown
Review the selected Terraform for:
1. Required tags — are all required tags (environment, owning-team,
   agency-code, data-classification, compliance-scope) present on
   every resource?
2. Hardcoded values — credentials, account IDs, region names that
   should be variables.
3. State backend — is remote state configured? Not local.
4. Provider version pinning — is the provider version pinned to
   a minor version (~> 2.x)?
5. Security groups/NSGs — are overly permissive rules present (0.0.0.0/0)?
6. Public exposure — any resource with public access that shouldn't have it?
Return numbered findings with line numbers.
```

### 8.4 Knowledge Bases Available

```
@Data-Platform-Standards   → SQL Server conventions, backup policy, DBA standards
@Cloud-Landing-Zone        → Azure and AWS landing zone patterns, control mappings
@On-Prem-Platform          → VMware, Salt, Ansible runbooks and patterns
@Security-Standards        → Enterprise Security's required policies
@DevSecOps-COE-Patterns    → CI/CD standards, reusable workflows, templates
@State-Compliance          → CJIS, NIST, tagging standards, state policy
```

### 8.5 Chat Modes

|Mode     |When to use                                                            |
|---------|-----------------------------------------------------------------------|
|**Ask**  |Questions, explanations, research, design discussion                   |
|**Edit** |Targeted changes to specific known files                               |
|**Agent**|Complex multi-file tasks — review all proposed changes before accepting|

Agent mode can make sweeping changes. Start it on docs and low-risk repos. Review every diff. It does not know which repos are production-critical unless your instruction file tells it.

-----

## 8.6 Security Scanning — Practical Reference for Every Team

This section explains the scanning tools your team will encounter in CI/CD pipelines, why each one exists, and what it does or does not cover. Understanding this prevents the most common mistake: assuming one tool covers everything.

### Two Separate Problems, Two Tool Families

```
PROBLEM 1 — SECRETS IN CODE
─────────────────────────────────────────────────────────────────────
What it is:  Passwords, API keys, connection strings, certificates,
             tokens, and credentials that a developer typed or pasted
             into a script, config file, SQL file, or IaC template.

Why it matters: A SQL connection string committed to a repo —
             even briefly, even in a feature branch — is potentially
             exposed. GitHub history preserves it even after deletion.
             For CJIS-scoped systems, this is an automatic finding.

Tools:       GitHub Secret Scanning   ← always on, no workflow needed
             Push Protection          ← blocks commits before they land
             Gitleaks                 ← custom patterns, history scanning

These tools scan CONTENT — every file regardless of language.
A .ps1 file, a .sql file, a .yaml config, a .json settings file —
all scanned. Language is irrelevant for secret detection.

PROBLEM 2 — VULNERABLE CODE PATTERNS
─────────────────────────────────────────────────────────────────────
What it is:  Logic flaws, insecure function usage, missing error
             handling, injection risks — vulnerabilities in how the
             code works, not in what strings are embedded.

Why it matters: A PowerShell script that uses ConvertTo-SecureString
             with plaintext, or a SQL script that builds dynamic
             queries from unvalidated input, or an Ansible role that
             runs commands as root when it doesn't need to.

Tools:       PSScriptAnalyzer  ← PowerShell specific
             Semgrep           ← PowerShell, SQL, Terraform, YAML, more
             Checkov / Trivy   ← IaC specific
             CodeQL            ← C#, Python, JavaScript ONLY

These tools analyze CODE STRUCTURE. Language matters —
each tool only covers the languages it was built for.
```

### What Each Tool Covers

```
TOOL              COVERS                          DOES NOT COVER
────────────────  ──────────────────────────────  ──────────────────────
GitHub Secret     Every file type and language.   Nothing — it scans all.
Scanning          Built into GitHub. Always on.

Push Protection   Blocks at commit time. Every    Cannot catch patterns
                  file type.                      it doesn't recognize
                                                  (use Gitleaks for custom).

Gitleaks          Every file type. Custom regex   Requires workflow setup.
                  patterns. Git history scan.     Not on by default.

PSScriptAnalyzer  .ps1  .psm1  .psd1             All other languages.
                  Security rules, credential
                  patterns, deprecated cmdlets.

Semgrep           PowerShell, SQL, Terraform,     Deep semantic analysis
                  Bicep, Ansible, YAML, Python,   (use CodeQL for that on
                  JS, Go, and more via rules.     supported languages).

Checkov           Terraform, Bicep,               Runtime app code.
                  CloudFormation, Ansible,
                  Salt, Kubernetes, ARM.

Trivy             Containers, file systems,        Deep code analysis.
                  Terraform, Kubernetes.

cfn-guard         CloudFormation (AWS) only.       Everything else.

CodeQL            C#, Java, JavaScript,           PowerShell ✗
                  TypeScript, Python, Go,          T-SQL ✗
                  Ruby, Swift.                     Terraform ✗
                                                   Bicep ✗
                                                   Ansible ✗
                                                   Salt ✗
```

**Rule for this team:** CodeQL runs only on repos that contain C#, Python, or JavaScript — primarily Enterprise Development repos. It does not run on PowerShell, SQL, IaC, or configuration repos because it produces no findings for those file types. Running it is not harmful, but it creates false confidence that code has been scanned when it has not.

### How Findings Reach the Security Tab

Every scanner outputs **SARIF** (Static Analysis Results Interchange Format — a standard JSON schema for security findings). The GitHub Actions workflow uploads the SARIF file after the scan. All findings from all tools appear in the GitHub Security tab and in the org-wide Security Overview that Enterprise Security monitors.

```
WHAT YOUR PR PIPELINE RUNS
(selected based on repo content)

All repos:
  ├── GitHub Secret Scanning     (native, always on)
  ├── Push Protection            (native, always on)
  └── Gitleaks                   (custom patterns, SARIF upload)

Repos with .ps1 / .psm1:
  └── PSScriptAnalyzer           (SARIF upload)

Repos with any code (PS, SQL, YAML, etc.):
  └── Semgrep                    (community + custom rules, SARIF upload)

Repos with Terraform / Bicep / CloudFormation / Ansible / Salt:
  └── Checkov                    (SARIF upload)
  └── Trivy                      (SARIF upload)

Repos with CloudFormation (AWS):
  └── cfn-guard                  (policy gates, SARIF upload)

Repos with C# / Python / JavaScript (Enterprise Dev only):
  └── CodeQL                     (SARIF upload)

All findings → GitHub Security tab → Security Overview → SIEM
```

### What PSScriptAnalyzer Catches in PowerShell

PSScriptAnalyzer has dedicated security rules. The most important for our environment:

|Rule                                            |What it catches                                                        |
|------------------------------------------------|-----------------------------------------------------------------------|
|`PSAvoidUsingPlainTextForPassword`              |Parameters named `Password` accepting plain strings                    |
|`PSAvoidUsingConvertToSecureStringWithPlainText`|`ConvertTo-SecureString -AsPlainText` usage                            |
|`PSAvoidUsingUsernameAndPasswordParams`         |Functions accepting both `-Username` and `-Password`                   |
|`PSAvoidUsingWriteHost`                         |`Write-Host` (output bypasses pipeline, hard to suppress in automation)|
|`PSUseDeclaredVarsMoreThanAssignments`          |Variables assigned but never used (often a logic error)                |
|`PSAvoidUsingInvokeExpression`                  |`Invoke-Expression` (executes arbitrary strings — injection risk)      |
|`PSAvoidUsingCmdletAliases`                     |`ls`, `cat`, `curl` — aliases that break cross-platform compatibility  |
|`PSUseApprovedVerbs`                            |Non-standard verb-noun naming                                          |

Run PSScriptAnalyzer locally before pushing:

```powershell
# Install if not present
Install-Module -Name PSScriptAnalyzer -Scope CurrentUser

# Scan a single file
Invoke-ScriptAnalyzer -Path ./scripts/Export-TableData.ps1 -Severity Warning,Error

# Scan all PS files in a directory
Invoke-ScriptAnalyzer -Path ./scripts/ -Recurse -Severity Warning,Error

# Include security rules specifically
Invoke-ScriptAnalyzer -Path ./scripts/ -Settings PSGallery
```

### What Semgrep Adds for SQL and PowerShell

Semgrep uses pattern-matching rules rather than deep semantic analysis. The community registry includes rules for:

- PowerShell: hardcoded credentials, unsafe invocations, `Invoke-Expression` with dynamic content
- SQL: dynamic query construction that could enable injection, `NOLOCK` patterns, `SELECT *`
- Terraform: hardcoded secrets in resource definitions, overly permissive security groups
- Ansible: `shell` module usage where a built-in module exists, `no_log: false` on sensitive tasks

Custom rules for our environment (maintained in the COE repo):

- Flag SQL connection strings embedded anywhere in PowerShell
- Flag hardcoded server names that match our production naming conventions
- Flag `$env:` variable reads for known secret variable names

### Adding Scanners to a Workflow

Minimal working example for a PowerShell repo:

```yaml
name: Security Scan
on: [pull_request]

jobs:
  secret-scan:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0          # Full history for Gitleaks

      - name: Gitleaks secret scan
        uses: gitleaks/gitleaks-action@v2
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}

  code-scan:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: PSScriptAnalyzer
        shell: pwsh
        run: |
          Install-Module PSScriptAnalyzer -Force -Scope CurrentUser
          $results = Invoke-ScriptAnalyzer -Path . -Recurse `
            -Severity Warning,Error -ReportSummary
          if ($results | Where-Object Severity -eq 'Error') {
            Write-Error "PSScriptAnalyzer found errors"
            exit 1
          }

      - name: Semgrep
        uses: semgrep/semgrep-action@v1
        with:
          config: >-
            p/powershell
            p/secrets
        env:
          SEMGREP_APP_TOKEN: ${{ secrets.SEMGREP_APP_TOKEN }}
```

Our shared reusable workflows already include these steps. In most repos, you call the reusable workflow rather than writing this from scratch:

```yaml
# In your repo's .github/workflows/security-scan.yml:
name: Security Scan
on: [pull_request]

jobs:
  scan:
    uses: managed-services/.github/.github/workflows/security-scan-baseline.yml@v2
    secrets: inherit
```

-----

## 9. Extensions by Role

### Core Set — Every Profile (§4.3)

Already listed above. Install these first in every profile.

### SQL & Data Platform Profile

|Extension            |ID                                     |What it does                                        |
|---------------------|---------------------------------------|----------------------------------------------------|
|SQL Server (mssql)   |`ms-mssql.mssql`                       |Query editor, schema browser, SQL Server connections|
|SQL Database Projects|`ms-mssql.sql-database-projects-vscode`|SSDT-style projects, DACPAC build and deploy        |
|Data Workspace       |`ms-mssql.data-workspace-vscode`       |Manages multiple database project connections       |
|Kusto / KQL          |`ms-vscode.vscode-kusto`               |KQL query language for Azure Monitor, ADX           |
|DBeaver Integration  |—                                      |Use DBeaver standalone (not a VS Code extension)    |
|SQL Formatter        |`inferrinizzard.prettier-sql-vscode`   |SQL formatting on save                              |
|dbt Power User       |`innoverio.vscode-dbt-power-user`      |dbt (data build tool) development support           |

**Note:** Azure Data Studio was retired. The `ms-mssql.mssql` extension in VS Code is the direct successor — maintained by the same Microsoft team.

### Cloud & IaC Profile (Azure + AWS)

|Extension            |ID                                           |What it does                                     |
|---------------------|---------------------------------------------|-------------------------------------------------|
|Azure Account        |`ms-vscode.azure-account`                    |Azure sign-in, required by other Azure extensions|
|Azure Resources      |`ms-azuretools.vscode-azureresourcegroups`   |Browse Azure resources                           |
|Bicep                |`ms-azuretools.vscode-bicep`                 |Bicep language support, validation, what-if      |
|HashiCorp Terraform  |`hashicorp.terraform`                        |Terraform HCL language support                   |
|AWS Toolkit          |`amazonwebservices.aws-toolkit-vscode`       |AWS resource browser, CloudFormation support     |
|Kubernetes           |`ms-kubernetes-tools.vscode-kubernetes-tools`|K8s cluster management                           |
|Docker               |`ms-azuretools.vscode-docker`                |Container image management                       |
|YAML (Red Hat)       |`redhat.vscode-yaml`                         |YAML validation with schema support              |
|CloudFormation Linter|`kddejong.vscode-cfn-lint`                   |Validates CloudFormation templates               |

**YAML schema associations** (add to `settings.json`):

```json
"yaml.schemas": {
  "https://json.schemastore.org/github-workflow.json": ".github/workflows/*.yml",
  "https://json.schemastore.org/ansible-playbook.json": "**/playbooks/*.yml",
  "https://raw.githubusercontent.com/awslabs/goformation/master/schema/cloudformation.schema.json":
    "**/cloudformation/**/*.yaml"
}
```

### On-Prem Platform Profile (VMware, Ansible, Salt)

|Extension          |ID                           |What it does                                                    |
|-------------------|-----------------------------|----------------------------------------------------------------|
|Ansible            |`redhat.ansible`             |Ansible playbook/role language support, ansible-lint integration|
|YAML (Red Hat)     |`redhat.vscode-yaml`         |YAML validation (Ansible is YAML-based)                         |
|Jinja              |`wholroyd.jinja`             |Jinja2 template syntax highlighting (used in Ansible and Salt)  |
|HashiCorp Terraform|`hashicorp.terraform`        |Terraform for VMware vSphere/NSX providers                      |
|SaltStack          |`korekontrol.saltstack`      |Salt state file (.sls) syntax highlighting                      |
|Python             |`ms-python.python`           |Python support (Salt and Ansible use Python under the hood)     |
|Remote - SSH       |`ms-vscode-remote.remote-ssh`|Connect to VMware-hosted Linux nodes                            |

**Ansible extension setup:**

```json
// In .vscode/settings.json for Ansible repos:
{
  "ansible.ansible.path": "/home/your_user/.local/bin/ansible",
  "ansible.validation.lint.enabled": true,
  "ansible.validation.lint.path": "/home/your_user/.local/bin/ansible-lint"
}
```

### PowerShell & Automation Profile

|Extension           |ID                    |What it does                                              |
|--------------------|----------------------|----------------------------------------------------------|
|PowerShell          |`ms-vscode.powershell`|Language support, debugging, PSScriptAnalyzer             |
|Pester Test Explorer|`pspester.pester-test`|Run and visualize Pester (PowerShell test framework) tests|

### Identity & Access (IAM) Profile

|Extension          |ID                           |What it does                                   |
|-------------------|-----------------------------|-----------------------------------------------|
|Azure Account      |`ms-vscode.azure-account`    |Azure/Entra ID connectivity                    |
|Azure AD / Entra   |via Azure Resources extension|Entra ID management                            |
|HashiCorp Terraform|`hashicorp.terraform`        |IaC for IAM policy as code                     |
|YAML (Red Hat)     |`redhat.vscode-yaml`         |Policy files often YAML-based                  |
|REST Client        |`humao.rest-client`          |Test Microsoft Graph API and identity endpoints|

### DevSecOps Profile

|Extension           |ID                                        |What it does                                                                        |
|--------------------|------------------------------------------|------------------------------------------------------------------------------------|
|GitHub Actions      |`github.vscode-github-actions`            |Workflow editing, run history, inline log viewing                                   |
|GitHub Pull Requests|`github.vscode-pull-request-github`       |Review and merge PRs without leaving VS Code                                        |
|Checkov             |`bridgecrew.checkov`                      |IaC security scanner — inline findings for Terraform, Bicep, CloudFormation, Ansible|
|Snyk                |`snyk-security.snyk-vulnerability-scanner`|Vulnerability scanning for code, containers, dependencies, IaC                      |
|Open Policy Agent   |`tsandall.opa`                            |OPA/Rego policy language support for cfn-guard and Sentinel rules                   |
|PowerShell          |`ms-vscode.powershell`                    |Runs PSScriptAnalyzer inline — security findings appear as you type in `.ps1` files |
|Dev Containers      |`ms-vscode-remote.remote-containers`      |Reproducible containerized dev environments                                         |
|YAML (Red Hat)      |`redhat.vscode-yaml`                      |GitHub Actions workflow validation and schema support                               |

**Notes on the DevSecOps profile:**

The PowerShell extension is listed here as well as in the PowerShell profile because PSScriptAnalyzer runs as part of it — giving inline security findings in `.ps1` files as you type, before the pipeline runs. If you are doing security review work on PowerShell scripts, install this extension in the DevSecOps profile too.

Semgrep does not have a first-class VS Code extension for inline scanning; it runs in CI/CD. For local pre-push scanning, run it from the terminal:

```bash
# From WSL, scan current directory with PowerShell rules
semgrep --config=p/powershell .

# Scan with secrets rules
semgrep --config=p/secrets .
```

Gitleaks also runs from the terminal for local history scanning:

```bash
# Scan current repo for secrets (including history)
gitleaks detect --source . --verbose
```

### Docs & Architecture Profile

|Extension          |ID                              |What it does                        |
|-------------------|--------------------------------|------------------------------------|
|Markdown All in One|`yzhang.markdown-all-in-one`    |Preview, shortcuts, table formatting|
|markdownlint       |`davidanson.vscode-markdownlint`|Consistent Markdown style           |
|Mermaid Preview    |`bierner.markdown-mermaid`      |Renders Mermaid diagrams inline     |
|Draw.io Integration|`hediet.vscode-drawio`          |Edit `.drawio` diagrams in VS Code  |
|PlantUML           |`jebbs.plantuml`                |Sequence, class, deployment diagrams|
|Excalidraw         |`pomdtr.excalidraw-editor`      |Whiteboard-style diagrams           |
|Paste Image        |`mushan.vscode-paste-image`     |Paste clipboard images into Markdown|

-----

## 10. Teams and Permissions Setup

This section walks through how to set up GitHub teams correctly for the ETS org. Our org is the reference model for other orgs and teams to follow.

### 10.1 Create Teams in GitHub

`Your org → Settings → Teams → New team`

```
Teams to create in the ETS org:

ets-admins
  Description: Org administrators — emergency access and org config
  Privacy: Secret (not visible to external contributors)
  Members: Named individuals only (max 3). Not a group account.

ets-platform
  Description: Platform engineering — cloud and on-prem infrastructure
  Privacy: Visible
  Parent team: none

ets-data
  Description: Data platform and DBA team
  Privacy: Visible
  Parent team: none
  Child teams to create:
    ets-data-dba     (SQL Server DBAs)
    ets-data-eng     (Data engineers / pipeline builders)

ets-iam
  Description: Identity and Access Management
  Privacy: Visible

ets-secops
  Description: Security Operations
  Privacy: Visible

ets-networking
  Description: Networking and Firewall
  Privacy: Visible

ets-backup
  Description: Backup and Disaster Recovery
  Privacy: Visible

ets-automation
  Description: Build, ordering, and automation
  Privacy: Visible

ets-devsecops-coe
  Description: DevSecOps Center of Excellence participants
  Privacy: Visible
  Note: Cross-functional; members from multiple above teams
```

### 10.2 Assign Repo Access

For each repo, access is assigned at the team level (not per-person). Go to `Repo → Settings → Collaborators and teams → Add a team`.

```
REPO ACCESS ASSIGNMENT REFERENCE

sql-backup-automation:
  @ets-data           → Maintain
  @ets-platform       → Write
  @ets-backup         → Write
  @ets-secops         → Triage
  @ets-devsecops-coe  → Write

landing-zone-azure:
  @ets-platform       → Maintain
  @ets-iam            → Write
  @ets-secops         → Triage
  @ets-devsecops-coe  → Write

iam-policy:
  @ets-iam            → Maintain
  @ets-secops         → Write
  @ets-platform       → Read

reusable-workflows (shared CI/CD):
  @ets-devsecops-coe  → Maintain
  @ets-secops         → Write
  All other teams     → Read (inner source consumption)

coe-decisions (ADR repo):
  @ets-devsecops-coe  → Maintain
  All other teams     → Read (with PR submission rights)
```

### 10.3 Set Up CODEOWNERS in Every Repo

```
# .github/CODEOWNERS template for a data platform repo

# Default — DBA team reviews everything not otherwise specified
*                       @ets-data/dba-team

# Workflows — security and COE review
/.github/workflows/     @ets-secops @ets-devsecops-coe

# Infrastructure — platform team reviews
/infra/                 @ets-platform @ets-data/dba-team

# IAM / service accounts — IAM team reviews
/iam/                   @ets-iam
/service-accounts/      @ets-iam @ets-data/dba-team

# Architecture decisions — COE reviews
/adrs/                  @ets-devsecops-coe

# Security policy
SECURITY.md             @ets-secops
```

### 10.4 Org-Level Settings to Configure

In `Your org → Settings`:

- **Member privileges → Base permissions:** Read (members can read all repos; write access granted per team/repo)
- **Member privileges → Repository creation:** Disabled for org members (repos created via templates by team leads or admins)
- **Actions → General:** Allow all actions and reusable workflows from GitHub and our trusted set
- **Security → Secret scanning:** Enabled; push protection on
- **Code security and analysis → Dependabot:** Enabled for all repos
- **Copilot:** Configured at enterprise level (see strategy doc §9)

### 10.5 Quarterly Access Review

Every quarter, a team lead and an IAM team member review:

- Who holds org owner (`ets-admins` membership)
- Whether team membership reflects current staffing
- Whether repo access levels still match team responsibilities
- Whether any individual access (outside team grants) exists and is still justified

Record findings and any changes in an ADR. This is the audit trail that satisfies SoD requirements.

-----

## 11. Settings and Repo Scaffolding

### 11.1 User `settings.json`

`Ctrl+Shift+P` → “Preferences: Open User Settings (JSON)”:

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
  "github.copilot.chat.agent.thinkingTool": true,
  "chat.mcp.enabled": true,
  "redhat.telemetry.enabled": false,
  "yaml.schemas": {
    "https://json.schemastore.org/github-workflow.json":
      ".github/workflows/*.yml",
    "https://json.schemastore.org/ansible-playbook.json":
      "**/playbooks/*.yml"
  }
}
```

### 11.2 Per-Repo `.gitattributes`

```gitattributes
# Normalize all text files to LF on commit
* text=auto eol=lf

# Windows tools that need CRLF
*.ps1 text eol=crlf
*.psm1 text eol=crlf
*.psd1 text eol=crlf
*.sql text eol=crlf
*.bat text eol=crlf
*.cmd text eol=crlf

# Binary — never touch
*.png binary
*.jpg binary
*.zip binary
*.7z binary
*.dacpac binary
*.bacpac binary
*.pfx binary
```

After adding `.gitattributes`:

```bash
git add --renormalize .
git commit -m "Normalize line endings"
```

### 11.3 Per-Repo `.editorconfig`

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

[*.{ps1,psm1,psd1}]
indent_size = 4
end_of_line = crlf

[*.sql]
indent_size = 4
end_of_line = crlf

[*.{tf,tfvars}]
indent_size = 2

[*.sls]
indent_size = 2
```

-----

## 12. Day-in-the-Life Examples

### 12.1 Cross-Team: Data Platform + IAM + Platform

```
Situation: Provisioning a new SQL Server instance on VMware,
integrating with the gMSA provisioned by IAM,
using the landing zone networking configured by Platform.

1. Open data-platform-partnership.code-workspace
   All five repos visible (SQL automation, landing zone, IAM policy,
   VMware IaC, shared PS modules)

2. VMware Aria MCP running. Platform team's Terraform state visible.
   IAM MCP running. Service account inventory accessible.

3. Ask: "@workspace — what network segment is configured for SQL
   servers in the VMware landing zone, and does IAM have a gMSA
   for the backup automation service in that segment?"

4. Copilot answers by reading:
   - VMware IaC repo (networking config)
   - IAM policy repo (gMSA definitions)
   And gives a specific answer: yes, here is the gMSA and
   the network segment it's scoped to.

5. You draft the Terraform change for the SQL VM using
   the platform's module — Copilot generates it following
   the instruction file conventions, with the correct tags.

6. PR opened. CODEOWNERS auto-requests:
   - @ets-data/dba-team (SQL config)
   - @ets-platform (VMware/Terraform)
   - @ets-iam (service account used)
```

### 12.2 On-Prem Ansible Hardening Run

```
Situation: Applying CIS RHEL 9 baseline to a new batch of VMs
provisioned by Aria Automation.

1. Open on-prem-platform workspace
   Ansible role repo and VMware IaC repo both visible.

2. Use /review-ansible prompt on the cis-baseline role:
   Chat: "/review-ansible"
   Select the role's main task file.
   Copilot reviews for idempotency, FQCN, hardcoded values.

3. Fix issues, push to feature branch, PR opened.
   ansible-lint runs in CI (WSL-based self-hosted runner).
   Checkov scans for IaC security issues.

4. Merge to main. Pipeline calls Ansible Navigator
   via the self-hosted runner, applies the role to
   the new VM group via Aria Automation webhook.
```

### 12.3 Agency Cloud Deployment

```
Situation: An agency team is deploying a new application
to their AWS account using the agency self-service process.

1. Agency team opens their repo (from tpl-terraform-aws-module)
   pre-configured with guardrail workflows.

2. They write their Terraform, extending the base module.
   Their PR triggers:
   - terraform validate + fmt
   - tflint (AWS rules)
   - Checkov security scan
   - Tag validation: all required tags present?
   - Template compliance: required security groups intact?

3. Any failure blocks the PR with specific guidance.
   Agency team fixes and re-pushes.

4. PASS: terraform plan posted as PR comment.
   Agency tech lead reviews the plan and approves.
   Our platform team is auto-requested on networking changes.

5. Merge to main → deployment pipeline runs automatically
   for dev/test environments.

6. To promote to production: agency tech lead opens a separate
   PR from test state to production. Requires:
   - 2 approvals (agency lead + platform on-call)
   - Deployment window check (only Tue/Thu 10am-2pm)
   - Evidence package generated automatically on completion.
```

-----

## 13. Troubleshooting

### 13.1 Copilot Suggestions Feel Generic

1. Does `.github/copilot-instructions.md` exist and have content?
1. Is `"github.copilot.chat.codeGeneration.useInstructionFiles": true` in settings?
1. If asking about multiple repos, are they all in a workspace?
1. Verify: In Chat, ask “What instructions do you have for this repo?”

### 13.2 `@workspace` Gives Weak Answers

- Confirm all repos are in the Folders list (left panel).
- First-open indexing takes a few minutes on large repos. Wait and retry.
- Check `files.exclude` — don’t exclude source files by accident.

### 13.3 Ansible Extension Won’t Run lint

- Verify `ansible-lint` is installed in WSL: `ansible-lint --version`
- Confirm the extension setting `ansible.validation.lint.path` points to the correct path.
- The extension runs in WSL context for WSL-opened repos. Open the folder via Remote - WSL (`code .` from inside the Ubuntu terminal).

### 13.4 Terraform Plan Fails on VMware Provider

- VMware vSphere provider requires GOVC-compatible network access.
- For on-prem: use self-hosted runner (labeled `[self-hosted, vmware]`).
- GitHub-hosted runners cannot reach your internal vCenter.

### 13.5 MCP Server Won’t Start

- View → Output → MCP. Read the error message.
- `npx` not found: Node.js not installed or not in PATH.
- Most servers need Node 18+ LTS. In WSL: `nvm install --lts && nvm use --lts`.

### 13.6 Git Uses Wrong Account

```powershell
# Check current credential config
git config --global --list | Select-String credential

# Clear and re-authenticate
git credential-manager erase
# Type: https://github.com
# Press Enter twice

# Re-clone — it will prompt for the EMU account
```

### 13.7 Azure Data Studio Opens from Old Shortcut

Azure Data Studio has been retired. Uninstall it (`winget uninstall --id Microsoft.AzureDataStudio`). Use:

- VS Code + mssql extension for query development
- SSMS for advanced DBA administration
- DBeaver Community for multi-database browsing

### 13.8 Line Endings Change on Every Commit

Add `.gitattributes` (§11.2). Then re-normalize:

```bash
git add --renormalize .
git commit -m "Normalize line endings per .gitattributes"
```

### 13.9 Security Scan Says Zero Findings — But the Repo Has PowerShell or SQL

Zero findings almost always means the wrong scanner ran, not that the code is clean.

**Check which scanner ran:** Open the Actions run, expand the security scan job, look at which tool executed and which files it reported scanning.

**If CodeQL ran on a PowerShell or SQL repo:** CodeQL skips those file types silently and reports success. This is not the correct scanner. The repo should use PSScriptAnalyzer and Semgrep instead. Check whether the workflow was copied from an Enterprise Development template rather than from the correct PowerShell or data platform template.

**If no scanner ran at all:** Check that the security scan workflow exists in `.github/workflows/` and that it is triggered on `pull_request`. New repos created from our templates have this pre-configured. Repos that predate the templates need the workflow added manually.

**If Gitleaks ran but found nothing unusual:** That is expected when the repo is clean. Gitleaks only flags known credential patterns — it will not flag every string. To test it works, check the Actions log for “X commits scanned” output. If you see that line, the scan ran correctly.

### 13.10 PSScriptAnalyzer Passes in the Pipeline but Flags Issues Locally (or Vice Versa)

The pipeline and your local VS Code may be using different rule sets or PSScriptAnalyzer versions.

```powershell
# Check local version
Get-Module PSScriptAnalyzer -ListAvailable | Select Version

# Update to latest
Update-Module PSScriptAnalyzer
```

The pipeline uses the version specified in the workflow. If they diverge, pin the pipeline to the same version you are using locally, or always update to the latest on both sides.

### 13.11 Push Protection Blocked My Commit — But It’s Not a Real Secret

Push protection uses pattern matching. It occasionally flags things that look like credentials but are not (a placeholder like `Password=example123` in a test fixture, for example).

**If it is a false positive:**

1. GitHub shows you the exact string it matched and which pattern triggered it
1. You can request a bypass through the GitHub UI (this creates an audit log entry — it is not invisible)
1. The bypass is logged and will appear in the compliance dashboard

**If it is a real secret that got into the code by mistake:**

1. Do not bypass — fix the code first
1. Remove the credential from the code
1. Rotate the credential immediately (assume it is compromised)
1. Commit the fix, then push

If the secret was already pushed to any branch at any point, rotating it is required regardless of whether the branch was merged. Git history preserves it.

-----

## 14. Glossary

**ADR (Architecture Decision Record).** A document recording one decision. Numbered, immutable once accepted.

**Agentless.** A configuration management approach (like Ansible) that connects to target nodes over SSH/WinRM — no agent software needs to be installed on managed nodes.

**Ansible.** Agentless configuration management and deployment tool. Uses YAML playbooks. Runs via SSH or WinRM.

**Aria Automation.** VMware Broadcom’s self-service infrastructure portal (formerly vRealize Automation / vRA). Manages on-premises blueprints and post-provisioning configuration orchestration.

**Azure Data Studio.** A Microsoft database tool — RETIRED in 2025. Do not install. Use VS Code + mssql extension instead.

**BCP (Bulk Copy Program).** SQL Server command-line utility for fast bulk data import/export.

**Bicep.** Microsoft’s language for Azure infrastructure as code.

**CJIS.** Criminal Justice Information Services. FBI security policy for systems touching criminal justice data.

**cfn-guard.** AWS policy-as-code tool for CloudFormation template validation at deployment time.

**cfn-lint.** AWS CloudFormation template linter. Catches syntax and structural errors.

**Checkov.** IaC security scanner supporting Terraform, Bicep, CloudFormation, Ansible, and more.

**CI/CD.** Continuous Integration / Continuous Delivery. Automated build, test, and deployment pipelines.

**CloudFormation.** AWS’s native IaC service — defines resources in JSON or YAML.

**CodeQL.** GitHub’s semantic code analysis engine. Finds security vulnerabilities in C#, Java, JavaScript, TypeScript, Python, Go, Ruby, and Swift by analyzing how data flows through a program. Does NOT support PowerShell, T-SQL, Terraform, Bicep, Ansible, or Salt — the primary languages in this environment. On a repo full of `.ps1` and `.sql` files, CodeQL scans nothing and returns zero findings. Use only on Enterprise Development repos where supported languages are present. Secret detection in PowerShell and SQL is handled by GitHub Secret Scanning, Push Protection, Gitleaks, PSScriptAnalyzer, and Semgrep instead.

**CODEOWNERS.** File listing who reviews changes to specific paths. Auto-requests reviewers on PRs.

**COE.** Center of Excellence. Cross-org artifact-producing body.

**Copilot Enterprise.** GitHub’s highest-tier AI coding assistant.

**DACPAC.** Data-tier Application Package. File format for SQL Server schema deployment.

**DBA.** Database Administrator.

**DBeaver.** Free, open-source, cross-platform database client. Supports SQL Server, PostgreSQL, MySQL, Oracle, and many more. Recommended replacement for Azure Data Studio.

**Dev container.** A Docker container defining a full development environment. Open the repo “in the container” for a consistent environment.

**DevSecOps.** Development + Security + Operations as one continuous practice.

**Drift.** When deployed infrastructure differs from the IaC definition.

**EMU.** Enterprise Managed User. GitHub accounts provisioned by the enterprise identity provider.

**gMSA.** Group Managed Service Account. Windows Active Directory account for services with auto-rotating passwords.

**GCM.** Git Credential Manager. Secure storage for Git authentication tokens.

**GHAS (GitHub Advanced Security).** GitHub’s enterprise security feature set. Includes Secret Scanning, Push Protection, Code Scanning (SARIF-based), and Dependabot. Secret Scanning and Push Protection are the most impactful features for this environment — they protect PowerShell and SQL files that CodeQL cannot scan.

**Gitleaks.** Open-source secret scanner. Scans any file type for credential patterns using built-in and custom regex rules. Particularly useful for scanning Git history to find secrets committed before push protection was enabled. Outputs SARIF for GitHub Security tab integration.

**PSScriptAnalyzer.** Microsoft’s PowerShell static analysis tool. Finds security anti-patterns in `.ps1`, `.psm1`, and `.psd1` files — hardcoded credentials, `Invoke-Expression` usage, missing error handling, deprecated cmdlets. The correct security scanner for PowerShell repos. Not CodeQL. Run it locally before pushing: `Invoke-ScriptAnalyzer -Path . -Recurse -Severity Warning,Error`.

**Push Protection.** A GitHub GHAS feature that intercepts a commit before it reaches any branch if a recognized secret pattern is found. The most effective preventive control for credential exposure. Must be enabled at the enterprise level. When it blocks a commit, the developer sees exactly which pattern was matched and why.

**SARIF (Static Analysis Results Interchange Format).** A standard JSON schema for security tool findings. Any tool that outputs SARIF can upload results to GitHub’s Security tab using `github/codeql-action/upload-sarif`. This is how Semgrep, Gitleaks, PSScriptAnalyzer, Checkov, and Trivy all appear in one unified security view alongside GitHub’s native findings.

**Semgrep.** Open-source and commercial rule-based code scanner. Community rule packs cover PowerShell, SQL, Terraform, Bicep, Ansible, YAML, Python, JavaScript, Go, and more. Allows custom rules for organization-specific patterns — e.g., flagging connection strings that match your internal server naming convention. Outputs SARIF. Runs in GitHub Actions with `semgrep/semgrep-action`.

**TruffleHog.** Open-source secret scanner focused on Git history — finds credentials committed and then deleted (which still exist in the repository’s history). Important for auditing repos that predate push protection being enabled.

**GitHub Actions.** GitHub’s built-in CI/CD system. Workflows in YAML files triggered by events.

**IAM.** Identity and Access Management.

**IaC.** Infrastructure as Code. Defining infrastructure in configuration files.

**KICS.** Keeping Infrastructure as Code Secure. Open-source multi-IaC security scanner by Checkmarx.

**MCP.** Model Context Protocol. Standard for connecting AI tools to external systems.

**mssql extension.** VS Code extension for SQL Server — the actively maintained successor to Azure Data Studio’s SQL functionality.

**Multi-root workspace.** A `.code-workspace` file grouping multiple repos in one VS Code window.

**nvm.** Node Version Manager. Manages multiple Node.js versions.

**OIDC.** OpenID Connect. Protocol enabling short-lived cloud credentials from GitHub Actions.

**OPA.** Open Policy Agent. Policy-as-code engine.

**Pester.** PowerShell testing framework.

**Podman Desktop.** Free, open-source container runtime — Docker Desktop alternative with no enterprise licensing fee.

**PR.** Pull Request. Code change proposal reviewed before merging.

**Profile (VS Code).** Bundle of extensions and settings. Switch profiles to change working contexts instantly.

**PSScriptAnalyzer.** PowerShell static analysis tool. Catches style and security issues.

**RACI.** Responsible, Accountable, Consulted, Informed.

**Reusable workflow.** GitHub Actions workflow defined once, called by many repos.

**Ruleset.** GitHub policy object applying rules to repos matching criteria.

**Salt / SaltStack.** Agent-based configuration management. Manages large fleets via Salt minions. Strong VMware on-prem integration.

**SBOM.** Software Bill of Materials. Complete inventory of build components.

**SCIM.** System for Cross-domain Identity Management. Automates user provisioning.

**SIEM.** Security Information and Event Management.

**SLSA.** Supply-chain Levels for Software Artifacts. Build provenance framework.

**SoD.** Separation of Duties. Builder ≠ auditor.

**SSO.** Single Sign-On.

**SSMS.** SQL Server Management Studio. Microsoft’s primary DBA GUI. Actively maintained.

**T-SQL.** Transact-SQL. Microsoft’s SQL Server query dialect.

**Terraform.** Open-source multi-cloud IaC tool by HashiCorp.

**tflint.** Terraform linter for provider-specific rules and best practices.

**Trivy.** Open-source scanner for containers, file systems, and IaC. Now covers Terraform scanning.

**VMware / Broadcom.** VMware (acquired by Broadcom) produces the enterprise hypervisor stack (vSphere, vCenter, NSX) used for on-premises infrastructure.

**WSL.** Windows Subsystem for Linux. Run Ubuntu inside Windows.

**Zero Trust.** Never assume trust by location. Verify every identity and request.

-----

## 15. Where to Ask Questions

|Question type                            |Channel                                  |
|-----------------------------------------|-----------------------------------------|
|VS Code, Copilot, MCP setup              |Team channel                             |
|Cross-team workspace questions           |Either team’s channel, tag the other team|
|GitHub access and permissions            |Team channel or @ets-admins              |
|Security policy questions                |@ets-secops channel                      |
|COE / standards questions                |DevSecOps COE channel                    |
|GitHub admin issues                      |@ets-admins rotation                     |
|Tool procurement (Podman vs Docker, etc.)|Team channel + manager                   |

**Do not self-serve on anything touching production data, credentials, or shared tooling.** Ask first. This includes MCP connections to anything above Tier 1 in the registry.

-----

## 16. What to Do Next

```
PROGRESSION PATH

Week 1
  ☐ Complete §2 — full install stack for your role
  ☐ Complete §3 — sign in and verify Copilot
  ☐ Create 2-3 profiles from §4 matching your role
  ☐ Open your team's stable workspace

Week 2-3
  ☐ Write a copilot-instructions.md for a repo you own
     Note how suggestions change immediately
  ☐ Add one Tier 1 MCP server (Filesystem or Fetch — no credentials)
  ☐ Set up the team CODEOWNERS file per §10.3

Week 4+
  ☐ Open or request a cross-delivery workspace with adjacent team
  ☐ Contribute a prompt file to the shared prompts repo
  ☐ Add the SQL or Azure MCP (dev/sandbox, read-only)
  ☐ Read the Enterprise Multi-Org Strategy document

When ready
  ☐ Participate in the DevSecOps COE
  ☐ Help onboard the next new team member using this guide
  ☐ Submit corrections to this guide via PR
```

-----

*Maintained by the ETS team. Corrections and improvements via PR to the platform repo. Updates tracked in Git. If something was confusing, flag it — the next person will hit the same thing.*