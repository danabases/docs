# Enterprise Multi-Org DevSecOps Strategy

**Purpose.** A living strategy document for the enterprise-wide rollout of GitHub, Copilot Enterprise, MCP, and VS Code practices across Commonwealth of Pennsylvania orgs — Enterprise Managed Services & Delivery, the Enterprise Information Security Office (EISO), Enterprise Development, CODE PA, and participating agency orgs. Written to be readable by people new to this space and to give the DevSecOps COE a stable reference to work from, one step at a time.

**Status.** Draft v1.1. Expected to evolve through COE decisions recorded as ADRs.

**Audience.** Org owners, COE members, architects, security partners, team leads across participating orgs. No deep GitHub-admin expertise assumed.

**Companion document.** For the developer-level setup (VS Code, profiles, workspaces, MCP wiring, instruction files), see the *Team Setup Guide*. This strategy document is the governance and rollout layer; the Team Setup Guide is the hands-on day-one onboarding layer.

-----

## 0. How to Read This Document

This is a long document because the topic is large. You don’t need to read it end-to-end. Sections are independent enough to stand alone. Suggested reading order by role:

- **Brand new to this work:** sections 1, 2, 3, 14, 15.
- **COE members:** sections 1, 4, 5, 12, 13, 14, 15.
- **Security partners (EISO / agency ISOs):** sections 1, 4, 6, 7, 8, 11.
- **Developers and team leads:** sections 2, 3, 9, 10, 14. Then read the Team Setup Guide.
- **GitHub tenancy admins:** sections 4, 6, 7, 8, 11, 13.

Every section answers three questions: **what** (is this thing), **why** (does it matter here), **how** (do we start, in small steps).

-----

## 1. Guiding Principles

These are the decisions we’ve made about *how* we work, before any specific tool choice.

**We are running a platform, not a team.** Our org provides shared services. Other orgs and agencies are our customers. Every pattern we adopt needs a consumable version — a template, a reusable workflow, a shared module, a documented standard — not just a working implementation inside our own repos.

**Small steps, staged rollout.** This is new territory for most participants, and the Enterprise GitHub EMU tenancy is still young. We deliberately avoid big-bang changes. The COE agenda moves one or two artifacts forward per sprint. Patterns are piloted in low-risk repos before being required anywhere.

**Make the right thing the easy thing.** Orgs that resist standards aren’t the problem. Standards that are harder to follow than to ignore are the problem. Templates, reusable workflows, instruction files, Copilot Spaces, and MCP servers exist to reduce friction, not to enforce compliance through pain.

**Transparency and visibility over gatekeeping.** Zero Trust means continuous verification, not continuous blocking. We prefer measurable posture (dashboards, attestations, evidence) over ad-hoc approval gates.

**Grounded in recognized standards.** Security architecture choices reference NIST (Cybersecurity Framework 2.0, SP 800-53 Rev. 5, SP 800-218 SSDF, SP 800-218A for AI/LLM-related development, SP 800-207 Zero Trust), CJIS Security Policy where applicable, CIS Benchmarks, and OWASP SAMM/ASVS. Commonwealth-specific requirements flow from Executive Order 2016-06 and the OA/OIT Information Technology Policy (ITP) catalog. When we deviate from any of these, we record why in an ADR.

**Flexibility and learning.** This document will be wrong in places. It will get updated. Nobody is expected to know all of it. Asking questions in the COE is the point of the COE.

**Crossover is temporary and intentional.** The current topology has our Enterprise Managed Services org holding responsibilities that logically belong to Enterprise Security (EISO), including GitHub tenancy admin. This is acknowledged. The long-term goal is proper separation of duties. Near-term, we act as stewards: we build the patterns, document the handoffs, and move responsibilities when the receiving org is ready.

-----

## 2. The Problem We’re Solving

The Commonwealth has many orgs and agencies doing technology work. Some silos are healthy (focused ownership); many are not (duplicated effort, inconsistent security posture, no shared language for architecture, tooling adopted without cross-functional input).

Specific pain today:

- Enterprise GitHub EMU is relatively new — there is active growth without consistent guardrails in place yet. Wild-West patterns are forming organically and will be harder to change later than now.
- New tools get deployed without security, data, or architecture review.
- Standards exist as slides and wiki pages. Very little is enforceable or measurable.
- Each team reinvents CI/CD, secret scanning, and dependency management.
- AI tooling (Copilot, MCP, Copilot Spaces, coding agent) is arriving fast — faster than policy can keep up.
- Agency orgs have no clear “golden path” — they ask us, they ask EISO, they get different answers, or no answer.
- Evidence for audits is gathered manually every time.

The goal: a federated technology practice where **required** security is enforced consistently, **recommended** patterns are easy to adopt, and **optional** customization is possible without breaking the baseline. Zero Trust applied to code: verify continuously, attribute everything, trust the process more than the individual.

-----

## 3. Roles and Orgs — Who Does What

Written plainly so people joining the COE understand the players. Names are descriptive, not formal.

**Enterprise Managed Services & Delivery (our org).** Builds and runs shared platforms: data platform, backup, infrastructure, firewall, change and ordering systems, on-prem and multi-cloud. Currently also holds GitHub tenancy admin. Leads the DevSecOps COE and publishes most shared templates, workflows, and modules.

**Enterprise Information Security Office (EISO).** Policy, governance, scanning, logging, incident response. Owns required security baselines. Led by the Commonwealth CISO. Partners with us on architecture. Long-term owner of enterprise-level GitHub security configuration.

**Enterprise Development.** Common models, shared application frameworks, custom development. Publishes libraries and reference implementations that other orgs consume.

**CODE PA (Commonwealth Office of Digital Experience).** Established 2023. Focused on citizen-facing digital services and consolidation of the public-facing PA.gov footprint. Natural consumer and contributor to the shared CI/CD and templating work described here.

**OA/OIT (Office of Administration, Office of Information Technology).** Publishes the Commonwealth’s IT policy catalog (ITPs) under Executive Order 2016-06. Sets the baseline that all agencies follow.

**Agency orgs (varied).** Some run fully self-sufficient technology. Some consume our managed services entirely. Most are hybrid. They are not obligated to adopt every pattern we publish, but we want our patterns to be the easiest choice.

**The DevSecOps COE.** A standing cross-org body that produces decisions, standards, reference implementations, and reusable workflows. Not a meeting body — an artifact-producing body. Membership includes representatives from each participating org. Currently running as an informal community effort; formalization is itself a COE agenda item.

A one-page responsibility matrix (who sets, who enforces, who audits each class of policy) lives in the COE repo and gets updated whenever a handoff happens. The matrix explicitly calls out the OA/OIT ITP-catalog relationship so agency teams can trace “why is this required” back to a published Commonwealth policy.

-----

## 4. GitHub Enterprise Topology

### 4.1 The Layers

GitHub EMU has three levels of configuration: **enterprise**, **org**, and **repo**. Each has things it does well. Placing a setting at the wrong level creates either sprawl or inflexibility. Rough rules:

|Setting type                                                                     |Level                       |Owner                        |
|---------------------------------------------------------------------------------|----------------------------|-----------------------------|
|SSO, IP allow lists, audit log streaming                                         |Enterprise                  |EISO (eventual), us (current)|
|Copilot policies, content exclusions, Spaces enablement, model availability, BYOK|Enterprise                  |EISO                         |
|Secret scanning, push protection defaults                                        |Enterprise                  |EISO                         |
|Enterprise-wide custom properties                                                |Enterprise                  |Joint                        |
|Enterprise-level required rulesets (signed commits, required workflows)          |Enterprise                  |EISO                         |
|Organization-level custom properties (enterprise GA as of 2025)                  |Enterprise                  |Joint                        |
|Org-level Actions permissions                                                    |Org                         |Each org                     |
|Recommended reusable workflows                                                   |Org (ours)                  |Us                           |
|Repo templates per archetype                                                     |Org (ours or Enterprise Dev)|Publishing org               |
|Branch protection beyond the baseline                                            |Repo                        |Repo owner                   |
|CODEOWNERS                                                                       |Repo                        |Repo owner                   |

### 4.2 Suggested Org Structure

Over time, we move toward:

- **EISO org** — required rulesets, CodeQL custom queries, required reusable workflows (security scans, SBOM, attestations), policy-as-code bundles, Enterprise Security Copilot Space.
- **Our org (Enterprise Managed Services)** — our delivery repos, the DevSecOps COE artifacts, recommended reusable workflows for our domains, shared modules, MCP configuration references, Copilot Spaces for data platform and cloud governance.
- **Enterprise Development org** — shared libraries, reference applications, common frameworks.
- **CODE PA org** — citizen-facing service code and shared components for the PA.gov platform.
- **Agency orgs** — their own code, consuming from the above.

Interim state (today): we hold some of what belongs in EISO. We document the handoff plan in the COE and move responsibilities piece by piece when the receiving team is staffed and ready.

### 4.3 Custom Properties — Tag Everything

Custom properties are labels applied to repos or orgs at the enterprise or org level. They let rulesets and automation target groups of repos without naming each one. Both **enterprise custom properties** and **organization custom properties** are now generally available, so you can tag once at the enterprise level and have the property visible everywhere.

> **Recent capability.** GitHub now lets enterprise/org admins require repo creators to *explicitly* select a value for a custom property instead of relying on the default. Use this for `data-classification`, `compliance-scope`, and `owning-team` so repos can’t be created without classification metadata.

Starting property set:

- `data-classification` — public, internal, confidential, restricted
- `environment` — sandbox, dev, test, prod
- `service-tier` — tier-1, tier-2, tier-3, none
- `owning-team` — short team code
- `owning-agency` — the owning Commonwealth agency or “enterprise”
- `compliance-scope` — none, pci, hipaa, sox, cjis, irs-1075, ferpa, other
- `lifecycle` — active, maintenance, deprecated, archived
- `ai-usage` — none, copilot-code-suggestions, copilot-agent, copilot-coding-agent (helps EISO track where AI tooling is involved)

Tagging existing repos is tedious but pays off forever. Start with the top 50 by activity. Everything else gets defaults. Treat this as a week-one COE deliverable, not a “someday” item.

### 4.4 Rulesets — Policy That Scales

Rulesets replace manual branch-protection configuration. Write a ruleset once, target by custom property, apply across many repos. **Enterprise-level rulesets are now GA**, so EISO can own baselines at the enterprise tier; we layer on top for our repos.

Examples we expect to land:

- All repos with `compliance-scope = cjis` require signed commits, two reviewers, passing CodeQL, and the security reusable workflow.
- All repos with `environment = prod` require CODEOWNERS review on `/infra/**` and `/.github/workflows/**`.
- All repos with `data-classification = restricted` block force-push to default branches and require linear history.
- All repos tagged `owning-agency` require a populated `CODEOWNERS` before the first PR can merge.

Rulesets are staged in **Evaluate mode** first (report-only) before becoming enforcing. Use Rule Insights to track which rules would have blocked work and prioritize developer enablement before flipping to enforcing. Nothing surprises a team on a Monday.

Bypass roles should be scoped to org/repo admin roles, not named individuals. Named-actor bypass exists but is a last resort and should carry an ADR.

-----

## 5. Inner Source — How Patterns Actually Spread

In a multi-org enterprise, mandates don’t spread patterns; adoption does. Inner source treats internal repos like open source across org boundaries.

**What we publish from our org:**

- Repository templates per archetype (SQL Database Project, PowerShell Module, Bicep Module, Terraform Module, Python Service, Static Docs Site).
- Reusable workflows for each archetype (build, test, scan, deploy).
- Starter workflows so the GitHub Actions “new workflow” button shows enterprise-blessed templates first.
- Shared PowerShell modules, Bicep modules, Terraform modules.
- The Approved MCP Server Registry.
- Copilot instruction file libraries per archetype.
- Seed content and instructions for shared Copilot Spaces.

**How we publish:**

- Internal visibility (the whole enterprise can read).
- Semantic versioning and tagged releases. Consumers pin to versions, not `@main`.
- `CONTRIBUTING.md`, `CODEOWNERS`, clear issue templates.
- A triage rotation on our side — someone on our team owns external issues each sprint. Without this, inner source decays.

**Discovery:** an internal docs site (GitHub Pages is enough to start) indexes templates and workflows, shows “when to use this,” and links to the source. Templates nobody can find don’t get used. The same index powers a Copilot Space so developers can also *ask* “which template fits my project?” in Chat.

-----

## 6. Security Strategy — Grounded and Staged

This section is where we as point lead contribute most directly, in partnership with EISO.

### 6.1 Anchor Standards

Our security architecture references established frameworks rather than inventing new ones. Named explicitly so teams know what to read:

- **NIST Cybersecurity Framework 2.0 (Govern, Identify, Protect, Detect, Respond, Recover).** CSF 2.0 added the **Govern** function as the sixth core function — explicitly covering cybersecurity strategy, policy, oversight, and supply-chain risk management. Our COE, responsibility matrix, and ADR process map naturally to the Govern function; when speaking to auditors or other agencies, we frame what the COE produces as Govern-function outputs.
- **NIST SP 800-218 (SSDF) v1.1.** Secure software development framework — the baseline for how we structure CI/CD security controls. Note: NIST published an SP 800-218 Rev. 1 (v1.2) public draft in December 2025; we’ll track the final and adapt.
- **NIST SP 800-218A.** SSDF Community Profile for generative AI and dual-use foundation models. Directly relevant to how we govern Copilot and MCP in the development flow. Applies in addition to SP 800-218, not instead of it.
- **NIST SP 800-53 Rev. 5.** Control catalog. Used when agency orgs ask “what control does this map to?”
- **NIST SP 800-207 (Zero Trust).** Architectural principles applied to identity, device, workload, and data.
- **CJIS Security Policy.** Required for any repo with `compliance-scope = cjis`. Drives specific controls around logging, access, and data handling. Relevant wherever Commonwealth criminal justice data is in scope.
- **CIS Benchmarks.** For baseline hardening of underlying platforms.
- **OWASP SAMM / ASVS.** Maturity and verification models for application security.
- **Commonwealth ITP catalog and Executive Order 2016-06.** The Pennsylvania-specific policy layer that sits on top of the above frameworks. Every standard we publish should cite the relevant ITP where one exists and flag when one is missing so EISO can consider publishing one.

None of these are complete on their own. Together they give us a vocabulary that EISO, auditors, agency partners, and vendors all recognize.

### 6.2 Zero Trust Applied to Code

Zero Trust in the development context means:

- **Identity is verified per action, not per session.** SSO + SCIM for GitHub EMU. Short-lived tokens. No shared service accounts.
- **Machines authenticate like humans.** OIDC federation from GitHub Actions to cloud providers. No long-lived cloud credentials stored in GitHub Secrets.
- **Every artifact is attributable.** Build provenance attestations (SLSA-style) generated by reusable workflows. SBOMs produced per build.
- **Verify continuously.** Secret scanning with push protection, dependency scanning, CodeQL, IaC scanning (Checkov, tfsec, Trivy) — all running on every PR, not just pre-release.
- **Least privilege everywhere.** Workflow tokens scoped to the minimum. CODEOWNERS on sensitive paths. Environment protection rules on deploy jobs.
- **AI tool use is traceable.** Copilot audit logs streamed to SIEM alongside other GitHub events. Spaces, prompt files, and MCP usage governed through org-level policy.

### 6.3 Staged Rollout

We don’t turn everything on everywhere. A rough sequence:

**Stage 1 — Foundations (first quarter of work).** SSO, audit log streaming (including Copilot events) to SIEM, enterprise-level secret scanning with push protection, Dependabot on by default. Low-risk, high-leverage, mostly invisible to developers. Rulesets drafted in Evaluate mode.

**Stage 2 — Baseline CI.** Required reusable workflow for security scans on a pilot set of repos. CodeQL on all repos where the language is supported. SBOM generation. Measured by coverage percentage. First enforcing rulesets for `compliance-scope != none` repos.

**Stage 3 — Provenance and Attestation.** OIDC deployments. Signed commits on `compliance-scope != none` repos. Build provenance attestations on release artifacts.

**Stage 4 — Differentiated Policy.** Rulesets targeting additional custom properties. Stricter controls on `restricted` and `cjis` repos. Environment protection rules on prod deployments. AI-usage policy tightened for repos where AI-generated code is in scope.

**Stage 5 — Continuous Evidence.** Dashboards aggregating posture per repo. Automated audit evidence bundles. Compliance scope maps to a defined control set automatically. Posture visible to EISO without a ticket.

Each stage is a COE agenda item. Each stage produces artifacts. We don’t start a stage until the prior one is stable.

### 6.4 Interface with Enterprise Security (EISO)

Two-way, explicit, no email-only requests:

- **They publish:** required policies, required workflows, content exclusion lists, custom CodeQL queries, custom secret-scan patterns, approved AI-model list, Copilot policy settings.
- **We publish:** compliance dashboards, evidence bundles, exception requests with reasoning, proposed standards for their review.
- **Joint:** the responsibility matrix, ADRs for shared decisions, the DevSecOps COE agenda, the AI-usage monitoring plan.

-----

## 7. Copilot Enterprise Strategy

### 7.1 Enterprise-Level Settings

Configured once, at the top:

- **Content exclusions** for credential files, regulated-data paths, and specific sensitive repos. Owned by EISO. Set at enterprise level so no org can relax them.
- **Duplicate suggestion filter** on.
- **Public code filter** per policy decision (recommend on for now; revisit).
- **Copilot Chat model access** — which models are available to which orgs. Reasoning-heavy models (Claude Opus-class, GPT-5.x-class) available by default; specific orgs can request additions. Expect the model roster to keep evolving.
- **Bring Your Own Key (BYOK)** — available in preview for Copilot Enterprise. Lets the Commonwealth supply API keys from Anthropic, OpenAI, Azure AI Foundry, etc., so AI model usage bills and data flows can be brought inside existing Commonwealth vendor agreements. Worth a dedicated ADR before enabling.
- **Data residency and FedRAMP.** Copilot data residency (US and EU options) and FedRAMP-compliant deployments are available to enterprise customers; EISO should own the selection.
- **Audit log streaming** for Copilot events alongside other GitHub events. Non-negotiable.
- **Premium request budgets.** Copilot’s premium-request meter governs agent-mode, coding-agent, code-review, and advanced model usage. Plan allocations per org; publish guidance on model-per-task to keep routine work on lower-cost models.

### 7.2 Copilot Spaces — The Highest-Leverage Feature

**Important update.** Copilot **Spaces** replaced **Knowledge Bases** on November 1, 2025. Older documentation still refers to “knowledge bases”; everywhere in this strategy, the modern equivalent is a Space. The migration path from existing knowledge bases to Spaces was released in late 2025 and should already have converted anything legacy the enterprise had.

A **Space** is a curated container that Copilot grounds answers in. Compared to the old knowledge bases, Spaces:

- Can include code, Markdown, JSON, images, file uploads, issues, and pull requests — not Markdown only.
- Can be created by **any** Copilot user, not just organization admins. Admins control sharing scope.
- Support per-Space custom instructions that are layered onto Chat turns using that Space.
- Can be shared privately, with specific users, with a team, with the org, or publicly within the enterprise.
- Auto-sync as the underlying GitHub sources change.

Planned enterprise Spaces:

- **EISO** publishes **Enterprise Security Standards** over their policy repos.
- **Our org** publishes **Data Platform Standards**, **DevSecOps COE Patterns**, **Cloud Landing Zone Reference**.
- **Enterprise Development** publishes **Common Application Frameworks**.
- **CODE PA** publishes **Digital Experience Standards** (citizen-facing service patterns, PA.gov components).
- **OA/OIT** seed: **Commonwealth ITP Reference** so developers can ask questions grounded in current policy.

Anyone in Chat can reference a Space. A developer in an agency org drafting a SQL integration gets answers consistent with our standards. This replaces a lot of tribal knowledge. It also meaningfully reduces the “ask three people, get three answers” problem for agency consumers.

### 7.3 Custom Instructions — Shape Every Suggestion

`.github/copilot-instructions.md` in a repo is auto-loaded into every Copilot Chat turn for that repo and influences code generation. Path-scoped `.instructions.md` files with `applyTo` frontmatter apply to specific globs (e.g., rules for `**/*.ps1` distinct from `**/*.bicep`).

Good instruction files are short, specific, and honest about constraints. We publish a catalog of them per archetype so teams can copy-paste and adapt.

### 7.4 Prompt Files

`.github/prompts/*.prompt.md` define reusable prompts invokable from Chat. Examples worth publishing:

- `/review-tsql` — checklist against a SQL file.
- `/harden-powershell` — StrictMode, error handling, logging.
- `/generate-adr` — scaffold an ADR in our format.
- `/threat-model` — STRIDE-style first pass.
- `/security-review` — walks a PR against the relevant baseline (800-218 practices, OWASP top 10).
- `/map-to-itp` — given a proposed standard, suggest which Commonwealth ITP(s) it supports or references.

### 7.5 Code Review, Coding Agent, and Mission Control

- **Copilot code review.** Enable org-wide, set as required on pilot repos first (low-risk: docs, ADRs, runbooks). Expand based on measured signal-to-noise.
- **Coding Agent.** Assign a GitHub issue to Copilot; it works in the background and opens a PR. Start with doc repos and low-risk refactors. Establish trust before letting it touch production IaC.
- **Mission Control.** Dashboard for steering multiple coding-agent tasks at once. Useful once a team routinely delegates several issues in parallel. Publish “how we use Mission Control” guidance when we have enough experience.
- **Custom agents and agent skills.** Emerging extensibility surface that lets orgs define agents tuned to a specific workflow. Track what EISO is willing to approve for use with regulated data before our teams build on this.

### 7.6 Model Selection

Strongest reasoning model for architecture, threat modeling, debugging. Faster models for boilerplate. Not set-and-forget — switch per task. The COE publishes guidance on when to use which, and updates it as the model roster changes (it has been changing roughly quarterly).

-----

## 8. MCP — Model Context Protocol at Enterprise Scale

MCP lets Copilot (and other AI clients) connect to real systems — GitHub, SQL, Azure, AWS, CMDB, ticketing, observability. Done well, it transforms Chat from a generic assistant into one that knows our environment. Done poorly, it sprays credentials and creates shadow integrations.

### 8.1 Central Hosting Over Local Configuration

We don’t want every developer running their own SQL MCP against dev instances with personal credentials.

The pattern:

- **Remote MCP servers** (HTTP / Streamable HTTP transport) for high-value targets, authenticated via SSO/OIDC. Developers configure a URL; the server authorizes based on their identity.
- **Local MCP servers** only for lightweight, non-sensitive things (filesystem scoped to a repo, public-documentation fetch).
- **Prohibited local servers** for anything touching production or regulated data.

### 8.2 Approved MCP Registry

A repo in our org publishes the list of approved MCP servers: endpoint, scopes, data classification, tier (freely usable / requires request / prohibited), owner, security review status, relevant ITP reference where applicable.

Developers discover MCP through this registry, not through random npm packages. This is a DevSecOps COE deliverable — probably one of the first, because it fills a gap nobody else has filled. It also short-circuits a realistic supply-chain risk: a malicious npm MCP package with a plausible name could otherwise leak credentials before anyone noticed.

### 8.3 Starter Set of MCP Servers

Prioritized for our roles and the broader enterprise:

|Server                                                        |Priority     |Why                                                                                                                                                               |
|--------------------------------------------------------------|-------------|------------------------------------------------------------------------------------------------------------------------------------------------------------------|
|**GitHub MCP (official, remote)**                             |High         |Issues, PRs, Actions, code search across the enterprise. Hosted at `https://api.githubcopilot.com/mcp/`. Supports read-only mode and dynamic toolsets via headers.|
|**Azure MCP (`@azure/mcp`)**                                  |High         |Resource Graph, policy, cost. Official Microsoft package.                                                                                                         |
|**Microsoft SQL MCP** (approved package TBD)                  |High         |Schema, read-only queries, index/plan introspection. Several community implementations; the COE picks one and pins it.                                            |
|**AWS MCP**                                                   |High         |Equivalent for AWS workloads.                                                                                                                                     |
|**Filesystem MCP (`@modelcontextprotocol/server-filesystem`)**|Medium       |Scoped local doc access.                                                                                                                                          |
|**Fetch / Web MCP**                                           |Medium       |Pull vendor docs during Chat.                                                                                                                                     |
|**Atlassian / Confluence MCP**                                |Medium       |Jira, ADR storage (where Confluence is used).                                                                                                                     |
|**Kusto / ADX MCP**                                           |Medium       |Telemetry queries.                                                                                                                                                |
|**Terraform MCP**                                             |Medium       |Registry lookups, module introspection.                                                                                                                           |
|**CMDB / ServiceNow MCP**                                     |High (future)|Environment context; critical once we wire in Commonwealth asset records.                                                                                         |

Start read-only. Elevate only when needed. Non-production targets first. Promote to higher environments slowly.

> **Note on the SQL MCP.** At the time of writing there is no single canonical Microsoft-published npm package for a SQL Server MCP server. Multiple community implementations exist with overlapping feature sets. The COE picks one, records the rationale in an ADR, and pins the version in the Approved MCP Registry. Do not install SQL MCP servers from search results without consulting the registry.

### 8.4 Security Review for MCP

Each approved MCP server has a lightweight security review recorded: what it can read, what it can write, how it authenticates, where logs go, who owns it, which ITPs and NIST control families it touches. EISO signs off on Tier 1 (freely usable) classifications.

-----

## 9. VS Code and Developer Experience

Covered in full in the Team Setup Guide. Summary here so COE members have the vocabulary:

- **Profiles** — bundles of extensions, settings, and keybindings per domain. Keep each profile lean.
- **Multi-root workspaces** — `.code-workspace` files grouping repos by coherent problem space, checked into a shared repo so everyone gets the same view.
- **Per-domain workspaces** (stable): data platform, cloud governance, DevSecOps COE, enterprise architecture.
- **Cross-org workspaces** (short-lived): security integration, agency onboarding, COE workstreams.
- **WSL2** selectively — for MCP servers, container scans, IaC scanners that run better on Linux.
- **Core extensions** — Copilot, Copilot Chat, GitLens, Error Lens, EditorConfig, plus domain-specific extensions per profile.
- **`.vscode/mcp.json` root key is `"servers"`** (not `"mcpServers"` — that’s the Claude Desktop / Cursor convention). This is the most common setup bug; calling it out here saves support time.

-----

## 10. Templates and Golden Paths

Templates are the main vehicle for pattern adoption.

Planned repository templates:

- `tpl-sql-database-project` — SQL Database Project with build, deploy, lint workflows.
- `tpl-powershell-module` — PowerShell module with Pester, PSScriptAnalyzer, publish workflow.
- `tpl-bicep-landing-zone-module` — Bicep module with what-if, lint, policy test.
- `tpl-terraform-aws-module` — Terraform module with tflint, tfsec, plan preview.
- `tpl-python-data-service` — Python service with pytest, ruff, Trivy.
- `tpl-static-docs-site` — MkDocs or similar, Pages-deployed.
- `tpl-adr-repo` — ADR collection with numbered decision files.

Each template includes:

- Wired reusable workflows.
- `.github/copilot-instructions.md` tuned to the archetype.
- `.vscode/extensions.json` and `.vscode/settings.json` so devs get the right experience on open.
- `.vscode/mcp.json` seeded with safe-to-share servers for the archetype.
- `CODEOWNERS` placeholder with guidance.
- `README.md` with “when to use this template.”
- `SECURITY.md` linking to the relevant ITP and NIST control mappings.

A small CLI (or scripted scaffold) called something like `pa-devsec new <archetype> <name>` can create repos from templates, apply custom properties, and register in the docs site. Not required for Stage 1 — templates alone work. The CLI is a Stage 3 or Stage 4 addition.

-----

## 11. Telemetry, Evidence, and Audit

EISO needs evidence; auditors need evidence; we need visibility. The pattern that replaces manual requests:

- **Audit log streaming** to the enterprise SIEM (one-time enterprise-level setting). Include Copilot and Copilot CLI events.
- **OIDC-based cloud auth** from GitHub Actions — deployments attributable to specific workflow runs.
- **SBOMs and build provenance attestations** generated by default from reusable workflows.
- **Compliance dashboard** — a scheduled workflow that queries the GitHub API and writes to an internal site. Shows per-repo posture against baselines. EISO consumes this instead of pinging us.
- **AI-usage visibility** — Copilot usage metrics (now including CLI activity) feed into the compliance dashboard so EISO can see where AI tooling is active.

This is an investment that pays compounding returns. Every audit cycle after it’s built is dramatically shorter.

-----

## 12. The DevSecOps COE — How It Runs

The COE fails when it produces slides without code. The COE succeeds when every meeting moves at least one artifact forward.

### 12.1 Outputs (in priority order)

1. **ADRs** in a shared repo. One decision per ADR, numbered, immutable once accepted, superseded only by another ADR.
1. **Standards documents** referencing the ADRs. Checklists, not essays.
1. **Reference implementations** — real repos showing standards in action.
1. **Reusable workflows and templates** making the standards the path of least resistance.
1. **Copilot Spaces** mirroring the published standards so developers can ask questions against them.
1. **Adoption measurements** — percentage of repos using required workflows, percentage with instruction files, time-to-onboard a new repo, percentage with correct `compliance-scope` tagging.

### 12.2 Meeting Cadence

- **Weekly working session** — the people actually producing artifacts. 45 minutes. Agenda is the artifact backlog.
- **Biweekly review** — representatives from each org. What shipped, what’s next, what needs a decision.
- **Monthly showcase** — open to anyone in the enterprise. Demonstrates new templates, workflows, or patterns. Builds adoption by making the work visible.

### 12.3 Artifacts Before Opinions

A proposal in the COE must include either a draft ADR or a prototype repo. Opinions without artifacts get parked until they have one. This keeps discussion grounded.

### 12.4 Formalization Path

The COE currently runs as an informal community effort. Formalizing it — charter, named roles, budgeted time allocation from each participating org — is itself a COE agenda item (and a candidate ADR-0002 after adopting this strategy as ADR-0001). Until formalized, we keep momentum by producing artifacts; once formalized, we can commit to larger work like cross-org required rulesets.

-----

## 13. Handoff Plan for Current Crossover

Acknowledging: our org currently holds GitHub tenancy admin and parts of the security policy role. Long-term, pieces of this belong with EISO. Short-term, we’re stewards.

**What we do now:**

- Act as admin transparently. Every enterprise-level change logged in a decision log visible to EISO.
- Pair on changes. EISO is included in major configuration decisions even when they don’t have the click rights.
- Document handoffs proactively. When a responsibility logically belongs with EISO, we write the runbook now so the handoff is a paperwork exercise, not an archaeology exercise.

**What we move first (once EISO is staffed for it):**

- Content exclusion list ownership.
- Required ruleset ownership.
- Custom CodeQL queries and secret-scan patterns.
- Copilot policy settings (including BYOK, data residency, model availability).

**What we keep:**

- Org-level configuration for our own org.
- Recommended (non-required) workflows and templates.
- The DevSecOps COE secretariat.

**What stays shared long-term:**

- The responsibility matrix.
- The ADR process.
- The compliance dashboard.
- Joint ownership of the Approved MCP Registry.

-----

## 14. Glossary — Plain Language

Written for people who are new to this area. Not every concept, just the ones that come up most.

**ADR (Architecture Decision Record).** A short document recording one decision, the context, and the consequences. Numbered. Immutable once accepted. Superseded by another ADR if needed.

**Attestation.** A signed statement about how an artifact was built. Lets consumers verify provenance.

**BYOK (Bring Your Own Key).** Enterprise Copilot setting that lets an organization supply its own API keys to the underlying AI model providers, bringing AI billing and data flows inside existing vendor agreements.

**CODE PA.** Commonwealth Office of Digital Experience, established 2023. Pennsylvania’s central team for citizen-facing digital services and the PA.gov platform.

**CODEOWNERS.** A file listing who must review changes to specific paths in a repo. Automatically requests their review on PRs.

**Coding Agent.** Copilot feature where an issue is assigned to Copilot and it works autonomously in the background, opening a PR when done.

**Copilot Space.** A curated container (code, docs, issues, custom instructions, etc.) that Copilot grounds answers in. Replaced the older “knowledge bases” feature on November 1, 2025.

**CSF (NIST Cybersecurity Framework) 2.0.** The February 2024 revision of the NIST Cybersecurity Framework, which added the **Govern** function as a sixth core function alongside Identify, Protect, Detect, Respond, and Recover.

**Custom properties.** Labels on repos or organizations that let rulesets and automation target groups without naming each one. Now generally available at both enterprise and organization levels.

**DevSecOps.** Development + Security + Operations as one continuous practice. Security is built in, not bolted on.

**EISO.** Enterprise Information Security Office. The Commonwealth’s central security governance body; home of the Commonwealth CISO.

**EMU.** Enterprise Managed User. Our enterprise GitHub accounts are EMU accounts, not personal GitHub.

**Executive Order 2016-06.** The Pennsylvania executive order that centralizes IT governance under OA/OIT and authorizes the ITP catalog.

**Inner source.** Using open source practices (public visibility, PRs, CONTRIBUTING.md) *inside* an enterprise.

**ITP.** Information Technology Policy. Numbered policies published by OA/OIT that agencies under the Governor’s jurisdiction must follow.

**MCP (Model Context Protocol).** A standard that lets AI clients connect to external systems (GitHub, SQL, Azure). Think of it as “USB for AI tools.”

**Mission Control.** The GitHub dashboard for steering multiple coding-agent tasks at once.

**OA/OIT.** Office of Administration, Office of Information Technology. Publishes the ITP catalog and coordinates enterprise IT governance across agencies under the Governor’s jurisdiction.

**OIDC (OpenID Connect).** An identity protocol. Relevant here because GitHub Actions can use OIDC to get short-lived cloud credentials instead of storing long-lived secrets.

**Premium request.** Copilot’s unit of paid usage for Chat, agent mode, code review, and advanced model selection. Plan allocations govern consumption.

**Reusable workflow.** A GitHub Actions workflow defined once in one repo, called by many other repos.

**Ruleset.** A GitHub policy object that applies rules (branch protection, required checks, signed commits) to repos matching criteria. Replaces manual branch protection configuration. Generally available at enterprise and organization levels.

**SBOM (Software Bill of Materials).** A list of all components in a build. Required for supply-chain security.

**SLSA.** A framework for supply-chain security levels. Higher levels require stronger provenance guarantees.

**SSDF.** NIST Secure Software Development Framework (SP 800-218). SP 800-218A is the companion profile for AI-related development.

**Starter workflow.** A workflow template that appears in the Actions “new workflow” UI. Makes the right choice the default choice.

**Template repo.** A repo marked as a template, so new repos can be created from it. Carries workflows, instruction files, and scaffolding forward.

**Workspace (VS Code).** A `.code-workspace` file grouping folders (often repos) together. Multi-root workspaces let you work across several repos in one window.

**Zero Trust.** An architectural principle: don’t grant trust by location or session. Verify identity, device, and authorization on every request.

-----

## 15. Getting Started — Small Steps for the COE

A suggested sequence for the first several months. Each step is a meeting agenda item with an artifact.

1. **Publish this strategy document** in the COE repo. Open it for comment. Record the first ADR: “ADR-0001: Adopt the Enterprise Multi-Org DevSecOps Strategy.”
1. **Draft the responsibility matrix.** One page. Who sets, who enforces, who audits each class of policy. Signed off by EISO and us. Reference the ITP catalog where applicable.
1. **Tag the top 50 repos with custom properties.** No rulesets yet. Just labeling. Require explicit-value selection for `data-classification` and `compliance-scope` on new repos.
1. **Stand up one Copilot Space** (pick one: “Data Platform Standards” is a good first). Announce it. Convert any legacy knowledge-base content if present.
1. **Publish the first repo template** (pick one: `tpl-powershell-module` or `tpl-sql-database-project`). Include instruction files, one reusable workflow, and a seeded `.vscode/mcp.json`.
1. **Publish the Approved MCP Registry repo** with the first three entries (remote GitHub MCP, Filesystem MCP, Fetch MCP). Document the review process.
1. **Turn on audit log streaming** to the enterprise SIEM. Include Copilot events. One-time change, high visibility value.
1. **Pilot one required reusable workflow** (probably security scans) on five volunteer repos. Start in Evaluate mode. Measure, fix, expand, then flip to enforcing.
1. **Record ADRs for every significant decision above.** The COE’s credibility comes from a trail of recorded reasoning.
1. **Review at the monthly showcase.** Show what shipped. Invite adoption across agency orgs.

After these ten steps, we have foundations. Then Stage 2 and beyond (section 6.3) become tractable.

-----

## 16. What This Document Isn’t

It’s not a final plan. It will be wrong in places. It doesn’t cover every tool, every archetype, or every scenario. Agency orgs will have needs we haven’t anticipated. EISO will have priorities that reorder ours. The Copilot and MCP ecosystems continue to change on a roughly quarterly cadence, and some specific tool details here will age out before the principles do.

The goal is to have enough shared vocabulary and enough published artifacts that every conversation starts from the same baseline — and enough humility to keep updating the baseline as we learn.

-----

*This document is maintained by the DevSecOps COE. Contributions via PR welcome. Major changes require an ADR.*