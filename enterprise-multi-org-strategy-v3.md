# Enterprise Multi-Org DevSecOps Strategy
## Draft v2 — Living Document

**Purpose.** A living strategy document for the enterprise-wide rollout of GitHub, Copilot Enterprise, MCP (Model Context Protocol), and VS Code practices across multiple orgs. Covers all teams within Enterprise Technology Services, Enterprise Security, Enterprise Development, and customer/agency orgs. Written to be readable by people completely new to this space while giving the DevSecOps COE (Center of Excellence) a stable reference to work from, one step at a time.

**Status.** Draft v2. Evolves through COE decisions recorded as ADRs (Architecture Decision Records). See §13.

**Audience.** Org owners, COE members, architects, security partners, team leads across all participating orgs. No deep GitHub administration expertise assumed.

**Change log from v1:**
- Corrected branch strategy language (§5.3, §6)
- Removed Azure Data Studio (retired)
- Added Teams and Permissions Design Reference (§5)
- Added On-Premises IaC — VMware, Salt, Ansible (§10)
- Added Cloud IaC Standardization by Platform (§11)
- Added Customer/Agency Deployment Process (§12)
- Added Template Guardrails and Policy Enforcement (§13)
- Added State Government Tagging Standards (§14)
- Added State Reference Models (§15)
- Expanded cross-team workspace framework
- Expanded inner-source triage model

---

## 0. How to Read This Document

This is a long document because the topic is large. Sections are written to stand on their own. You do not need to read it end-to-end.

**Suggested reading by role:**

| Your role | Start here |
|---|---|
| Brand new to this work | §1, §2, §3, §18, §19 |
| COE members | §1, §4, §5, §6, §13, §16, §18, §19 |
| Security partners | §1, §4, §5, §7, §8, §9, §13, §14 |
| Developers and team leads | §2, §3, §6, §11, §18 — then the Team Setup Guide |
| GitHub tenancy admins | §4, §5, §7, §8, §9 |
| DBA and data platform teams | §2, §3, §6, §10, §11, §18 — then the Team Setup Guide |
| Platform / IAM / Networking / Security teams | §2, §3, §5, §6, §10, §11, §12 |
| Customer/agency teams | §12, §13, §14, §15, §18 |

Every section answers three questions: **what** (is this thing), **why** (does it matter here), **how** (do we start, in small steps).

**Key terms used throughout:**
- **Org** — a GitHub organization. A container that holds repositories and sets policies for them.
- **Repo** — a repository. A project's code, files, and history stored in GitHub.
- **CI/CD** — Continuous Integration / Continuous Delivery. Automated pipelines that build, test, and deploy code.
- **PR** — Pull Request. A proposal to merge code changes, reviewed before merging.
- **ADR** — Architecture Decision Record. A document capturing a single decision and its reasoning. See §18 for full glossary.
- **IaC** — Infrastructure as Code. Defining servers, networks, and cloud resources in configuration files rather than manual clicks.

---

## 1. Guiding Principles

These decisions about *how* we work apply before any specific tool choice.

**We are running a platform, not a team.** Our org provides shared services. Other orgs are our customers. Every pattern needs a consumable version — a template, a reusable workflow, a shared module, a documented standard — not just a working implementation inside our own repos.

**Small steps, staged rollout.** This is new territory for most participants. The COE agenda moves one or two artifacts (concrete deliverables) forward per sprint. Patterns are piloted in low-risk repos before being required anywhere.

**Make the right thing the easy thing.** Orgs that resist standards are not the problem. Standards harder to follow than to ignore are the problem. Templates, workflows, and tools exist to reduce friction.

**Transparency and visibility over gatekeeping.** Zero Trust (an architectural principle: never assume trust based on location — always verify) means continuous verification, not continuous blocking. Measurable posture over ad-hoc approval gates. No centralized controls that prevent visibility, accountability, transparency, or autonomy.

**No rogue operators, no invisible power.** Every administrative action is logged. Every policy decision is recorded in an ADR. GitHub tenancy administrators operate transparently with Enterprise Security as co-observers. Separation of duties (SoD — builder ≠ auditor) is the target state; transparency is the interim compensating control.

**Grounded in recognized standards.** Architecture choices reference NIST (SP 800-53, SP 800-218 SSDF, SP 800-207 Zero Trust), CJIS Security Policy, CIS Benchmarks, OWASP SAMM/ASVS, and applicable state government standards. Deviations are recorded in ADRs.

**Learn from the field.** States and agencies that have already modernized (see §15) provide concrete evidence of what works and what fails. We learn from them and aim to go further.

**Flexibility and learning.** Nobody is expected to know all of this. Asking in the COE is the point of the COE. This document will be updated.

**Crossover is temporary and intentional.** Our org currently holds GitHub tenancy admin. The goal is proper SoD. Near-term, we act as stewards: we build the patterns, document the handoffs, and move responsibilities when the receiving org is ready.

---

## 2. The Problem We Are Solving

The enterprise has many orgs doing technology work. Some silos are healthy. Many are not — producing duplicated effort, inconsistent security posture, and tools deployed without cross-functional review.

**Specific pain today:**

- New tools get deployed without security, data, or architecture review.
- Standards exist as slides and wiki pages. Very little is enforceable or measurable.
- Each team reinvents CI/CD, secret scanning, and dependency management.
- On-premises IaC tooling (VMware/Broadcom, Ansible, Salt) is inconsistently used and not integrated into the same DevSecOps pipeline as cloud.
- AI tooling arrives faster than policy.
- Customer/agency orgs have no clear, compliant, autonomous path to build and deploy cloud resources.
- Evidence for audits is gathered manually every time.
- GitHub teams and permissions are inconsistently configured, creating both gaps and unnecessary restrictions.

**The goal:** a federated practice where:

```
REQUIRED    → enforced consistently (Enterprise Security + policy)
RECOMMENDED → easy to adopt (published by our org / COE)
OPTIONAL    → customization within guardrails (team-specific)
```

---

## 3. Roles and Orgs — Who Does What

### 3.1 Enterprise Technology Services — Our Org

We build and run shared platforms. Our org has many teams that collaborate closely and share the same GitHub organization. Key teams:

**Data Platform / DBA Team**
SQL Server administration, optimization, migration, managed database services. Leading adoption of new tools and processes across other database teams. Close collaboration with Platform, IAM, Security, and Networking.

**Platform Engineering**
Shared infrastructure services, cloud landing zones (pre-configured cloud environments with governance built in), compute, storage. On-premises VMware/Broadcom environment management.

**IAM (Identity and Access Management)**
Enterprise identity, SSO (Single Sign-On), directory services, access policy, privileged access management. Cross-cutting concern touching every other team.

**Security Operations (SecOps)**
Threat monitoring, incident response, vulnerability management, security tooling. Works closely with Enterprise Security on policy; handles operational security within our org.

**Networking / Firewall**
Enterprise network design, firewall rule management, change management for network policy, DNS, connectivity between on-prem and cloud.

**Backup and Disaster Recovery**
Enterprise backup systems, recovery testing, data protection standards, RPO/RTO (Recovery Point/Time Objectives) governance.

**Build, Ordering, and Automation**
Change management, service request fulfillment, ordering systems, integration workflows between enterprise technology services and customer orgs.

**DevSecOps COE Secretariat**
We run the COE (Center of Excellence), produce its artifacts, and lead adoption. The COE draws participants from all teams above and from Enterprise Security and Enterprise Development.

### 3.2 Enterprise Security

Policy, governance, security scanning, logging, incident response. Owns required security baselines. Long-term owner of enterprise-level GitHub configuration. Primary architecture partner.

### 3.3 Enterprise Development

Common application models, shared libraries, frameworks, custom development. Publishes reference implementations that other orgs consume.

### 3.4 Customer / Agency Orgs — Three Types

```
Customer/Agency Org Types
├── Self-Sufficient
│   ├── Own technology teams
│   ├── Own GitHub orgs (within the enterprise)
│   ├── May adopt our patterns voluntarily
│   └── Need: compliant autonomous cloud deployment process
│
├── Hybrid
│   ├── Some in-house capability
│   ├── Consume some managed services
│   ├── Mix of their own and our standards
│   └── Need: clear handoff points between self-managed and managed
│
└── Fully Managed
    ├── Limited or no dedicated tech staff
    ├── Consume our services end-to-end
    └── Need: turn-key compliant environments with visibility
```

### 3.5 The DevSecOps COE

A standing cross-org body that produces decisions, standards, reference implementations, and workflows. Not a meeting body — an artifact-producing body.

### 3.6 Org Constellation

```
                    ┌─────────────────────────────────────┐
                    │       GitHub Enterprise EMU         │
                    │  SSO · SCIM · Audit Logs · Policies │
                    └──────────────┬──────────────────────┘
                                   │ contains
          ┌────────────────────────┼────────────────────────┐
          │                        │                        │
          ▼                        ▼                        ▼
┌─────────────────┐   ┌──────────────────────┐   ┌─────────────────┐
│ Enterprise      │   │  Enterprise Tech      │   │  Enterprise     │
│ Security Org    │   │  Services Org (ours)  │   │  Development    │
│                 │   │                       │   │  Org            │
│ Required policy │   │  Teams (see §5):      │   │                 │
│ Scan rules      │◄──│  Data/DBA · Platform  │──►│ Libraries       │
│ Scan patterns   │   │  IAM · SecOps         │   │ Frameworks      │
│ Audit baselines │   │  Networking · Backup  │   │ Reference apps  │
│                 │   │  Automation · COE     │   │                 │
│                 │   │  [interim: GH admin]  │   │                 │
└────────┬────────┘   └──────────┬────────────┘   └────────┬────────┘
         │                       │ patterns/templates       │
         └───────────────────────┼──────────────────────────┘
                                 ▼
         ┌───────────────────────────────────────────────┐
         │           Customer / Agency Orgs              │
         │  Self-Sufficient │   Hybrid   │   Managed     │
         └───────────────────────────────────────────────┘
```

---

## 4. GitHub Enterprise Topology

### 4.1 Three Configuration Levels

```
GITHUB ENTERPRISE CONFIGURATION HIERARCHY

┌─────────────────────────────────────────────────────────────────┐
│  ENTERPRISE LEVEL                                               │
│  Owner: Enterprise Security (target) / Us (current interim)   │
│                                                                 │
│  SSO · SCIM · IP allow lists · Audit log streaming             │
│  Copilot content exclusions · Secret scanning defaults         │
│  Push protection · Required rulesets · Custom properties       │
└───────────────────────────┬─────────────────────────────────────┘
                            │
          ┌─────────────────┼──────────────────────┐
          ▼                 ▼                      ▼
┌──────────────┐   ┌──────────────┐      ┌──────────────┐
│  ENTERPRISE  │   │  MANAGED     │      │  CUSTOMER    │
│  SECURITY    │   │  SERVICES    │      │  AGENCY ORG  │
│  ORG         │   │  ORG (ours)  │      │              │
│              │   │              │      │              │
│ Required     │   │ Recommended  │      │ Their code   │
│ workflows    │   │ workflows    │      │ Their teams  │
│ Scan rules   │   │ Templates    │      │ (+ baseline) │
└──────────────┘   └──────────────┘      └──────────────┘
                            │
                   ┌────────▼────────┐
                   │  REPO LEVEL     │
                   │  CODEOWNERS     │
                   │  Branch rules   │
                   │  Instruction    │
                   │  files, mcp.json│
                   └─────────────────┘
```

### 4.2 Custom Properties — Tagging Every Repo

Custom properties are labels on repos that let rulesets and automation target groups by characteristic. See §14 for full state government tagging standards. Core set:

| Property | Values | Purpose |
|---|---|---|
| `data-classification` | public, internal, confidential, restricted | Data protection level |
| `environment` | sandbox, dev, test, prod | Where output runs |
| `service-tier` | tier-1, tier-2, tier-3, none | Business criticality |
| `owning-team` | team code | Accountability |
| `compliance-scope` | none, pci, hipaa, sox, cjis, fisma, other | Regulatory requirements |
| `lifecycle` | active, maintenance, deprecated, archived | Repo lifespan state |
| `agency-code` | agency identifier | Customer attribution for billing/reporting |
| `iac-platform` | vmware, azure, aws, gcp, hybrid, none | Infrastructure deployment target |
| `deployment-model` | managed, self-service, hybrid | How this repo's output gets deployed |

### 4.3 Rulesets — Policy That Scales

Write a ruleset once, target by custom property, apply across many repos. Always stage in **evaluate mode** (report-only) before enforcing.

```
Example rulesets:

Ruleset: CJIS-Compliance-Baseline
  Targets: compliance-scope = cjis
  Rules:
    ✓ Require signed commits
    ✓ Require 2 reviewers (including security team member)
    ✓ Require security scan workflow to pass (Semgrep + Gitleaks + PSScriptAnalyzer)
    ✓ Require secret scanning to pass (push protection always on)
    ✓ Require linear history (no merge commits)

Ruleset: Production-Environments
  Targets: environment = prod
  Rules:
    ✓ Require CODEOWNERS review on /infra/**
    ✓ Require CODEOWNERS review on /.github/workflows/**
    ✓ Prohibit force-push
    ✓ Require deployment workflow approval

Ruleset: Agency-Self-Service-Guardrail
  Targets: deployment-model = self-service
  Rules:
    ✓ Require template compliance workflow to pass
    ✓ Require tag-validation workflow to pass
    ✓ Prohibit deletion of required policy files
    ✓ Require security scan workflow
```

---

## 5. Teams and Permissions Design Reference

This section is a **model and reference** for our org and for any other org learning to set up GitHub teams. The patterns here should be the first thing a new org reads when setting up their own team structure.

### 5.1 Core Concepts

**GitHub access levels (lowest to highest):**

```
READ     → Clone, view code, open issues
TRIAGE   → Manage issues and PRs, label, close/reopen (no code push)
WRITE    → Push branches, open and merge PRs (within protection rules)
MAINTAIN → Manage repo settings (not destructive admin actions)
ADMIN    → Full control including destructive actions (delete, archive)
```

**Rule:** Assign the minimum level that lets the team do their job. Start at Read. Elevate when needed. Document why.

**Org owners** (the GitHub equivalent of system admin) should be ≤3 people, named individuals, audited regularly. This is not a role for teams or shared accounts. Org owner access should require MFA (Multi-Factor Authentication) and short-lived sessions.

### 5.2 Enterprise Technology Services — Team Hierarchy

Teams in GitHub can be **nested** (parent/child). Child teams inherit parent permissions but can be granted additional access. This allows "department-level" and "squad-level" teams without duplicating permission assignments.

```
GitHub Org: Enterprise Technology Services
│
├── @ets-admins                          ← Org owners. ≤3 named people.
│   Purpose: Emergency access, org config, billing
│   Repo access: Admin on all repos (via org ownership)
│   Policy: Every admin action must be logged with a reason
│
├── @ets-platform                        ← Platform Engineering team
│   Purpose: Shared infra, landing zones, cloud platforms
│   Repo access:
│     Maintain → platform repos
│     Write    → shared module repos, IaC repos
│     Read     → all org repos
│
├── @ets-data                            ← Data Platform / DBA team
│   Purpose: SQL Server, data platform, backup pipelines, BCP
│   Repo access:
│     Maintain → data platform repos
│     Write    → shared PS modules repo, data ADR repo
│     Read     → platform repos (for integration awareness)
│   Sub-teams:
│     @ets-data-dba    ← SQL Server DBAs specifically
│     @ets-data-eng    ← Data engineers / pipeline builders
│
├── @ets-iam                             ← Identity and Access Management
│   Purpose: SSO policy, access provisioning, privileged access
│   Repo access:
│     Maintain → IAM policy repos
│     Write    → identity configuration repos
│     Read     → ALL repos (IAM is a cross-cutting concern)
│   Note: Read-all is justified — IAM must understand access
│         patterns across everything they govern
│
├── @ets-secops                          ← Security Operations
│   Purpose: Threat monitoring, incident response, vuln management
│   Repo access:
│     Maintain → security tooling repos, SIEM config repos
│     Triage   → ALL repos (can manage security issues/alerts anywhere)
│     Read     → ALL repos (audit and monitoring function)
│   Note: Triage-all is intentional. SecOps must be able to
│         manage security alerts in any repo without a PR.
│
├── @ets-networking                      ← Networking / Firewall
│   Purpose: Network design, firewall policy, DNS, connectivity
│   Repo access:
│     Maintain → networking and firewall repos
│     Write    → network IaC repos (Ansible, VMware NSX)
│     Read     → platform repos, cloud landing zone repos
│
├── @ets-backup                          ← Backup and DR
│   Purpose: Enterprise backup, recovery testing, DR procedures
│   Repo access:
│     Maintain → backup automation repos
│     Write    → shared backup module repo
│     Read     → data platform repos, platform repos
│
├── @ets-automation                      ← Build, Ordering, Automation
│   Purpose: Change management, service ordering, integration workflows
│   Repo access:
│     Maintain → automation and ordering repos
│     Write    → integration workflow repos
│     Read     → platform repos
│
├── @ets-devsecops-coe                   ← COE participants (cross-team)
│   Purpose: COE artifact production, standards, templates
│   Membership: One or more people from each above team
│   Repo access:
│     Maintain → COE repo, template repos, reusable workflow repos
│     Write    → docs site repo
│     Read     → all org repos
│
└── @ets-contributors                    ← General contributors (external to our teams)
    Purpose: Partner-team members who contribute to our repos
    Repo access: Granted per-repo by repo maintainer
    Default: Read on repos they need. Write on specific paths via PR.
```

### 5.3 Cross-Team Collaboration Pattern

Many repos serve multiple teams. Rather than adding individuals, use team-based access:

```
Repo: sql-backup-automation
  @ets-data       → Maintain  (primary owners)
  @ets-platform   → Write     (platform integration points)
  @ets-backup     → Write     (backup team contributes procedures)
  @ets-secops     → Triage    (can manage security alerts)
  All others      → Read      (via org-wide read default)

CODEOWNERS in this repo:
  /src/              @ets-data/dba-team
  /infra/            @ets-platform
  /backup-procs/     @ets-backup @ets-data/dba-team
  /.github/          @ets-devsecops-coe @ets-secops
```

**The CODEOWNERS file provides finer control than team access level.** A team might have Write access to a repo but CODEOWNERS ensures the right sub-group reviews the right paths.

### 5.4 Permissions Guardrails — Preventing Rogue Operators

The following controls prevent unchecked access without creating unnecessary friction:

```
PREVENTING ROGUE OPERATIONS

Control                         How it's implemented
──────────────────────────────  ──────────────────────────────────────────
No direct push to main          Branch protection: require PR on main
                                No one bypasses this — including admins

No self-approval                Branch protection: author cannot approve
                                their own PR

Required reviews                Minimum 1 reviewer for standard repos
                                Minimum 2 for compliance-scoped repos

Workflow file protection        CODEOWNERS: /.github/workflows/ requires
                                @ets-secops or @ets-devsecops-coe review

Admin actions logged            GitHub audit log streams to SIEM
                                All org-owner actions generate alerts

Org owner audit                 Quarterly review of who holds org owner
                                Any addition requires ADR

Secret management               No secrets in code (push protection on)
                                Secrets via GitHub Environments with
                                required reviewer approval for prod

Visibility: not hidden power    Security and IAM teams have read-all
                                so no configuration is invisible to them
```

### 5.5 This Org as a Model

Our org's team structure should be documented, published (as internal visibility), and referenced by customer/agency orgs when they set up their own GitHub organizations. The pattern:

1. Start with the minimum team hierarchy needed
2. Use nested teams for sub-specialties
3. Assign repo access at team level, not individual level (individuals move; teams are stable)
4. Use CODEOWNERS for path-level control
5. Document every non-obvious access decision in the repo's README or in an ADR

---

## 6. Branch Strategy — Clarified

This section replaces and corrects language in v1 that caused confusion about when to use `main` versus version tags.

### 6.1 `main` Is the Version of Truth

In every repo we own and every repo our team develops on, **`main` is the production-ready branch.** It is always deployable. It is always reviewed. It is the thing others should clone, fork from, or reference.

```
STANDARD BRANCH MODEL (GitHub Flow)

  feature/add-retention-policy  ──────────────────────► PR ──► main
  fix/bcp-quote-handling         ──────────────────────► PR ──► main
  hotfix/cjis-audit-gap          ──────────────────────► PR ──► main

main is always:
  ✓ Passing all required checks (security scan, secret scan, build, lint)
  ✓ Reviewed and approved (via PR + CODEOWNERS)
  ✓ Deployable to production
  ✓ The version template processes and pipelines reference
  ✓ The thing you clone to start work
```

Feature branches are in-progress work. They are merged to `main` via PR only after review and passing checks. No one develops directly on `main`. Releases are tagged on `main`.

### 6.2 When Version Tags Apply (Consumer vs Producer)

The distinction is between **your own repos** (where `main` is truth) and **consuming shared repos from someone else** (where pinning to a version tag protects you from their changes).

```
YOU ARE THE PRODUCER (your own repo):
  main = production truth
  Feature branches → PR → main
  Tag releases from main: v1.0, v2.1, v3.0

YOU ARE THE CONSUMER (calling another org's shared workflow):
  # Fragile — their next commit to main could break you:
  uses: enterprise-security/security/.github/workflows/security-scan.yml@main

  # Stable — you update on your schedule, after testing:
  uses: enterprise-security/security/.github/workflows/security-scan.yml@v3
```

**Why pin versions when consuming?** If Enterprise Security pushes a breaking change to their reusable workflow's `main`, every repo referencing `@main` breaks simultaneously — no warning, no testing, no choice. Pinning to `@v3` means their `main` changes don't affect you until you explicitly update your reference.

This is identical to how package managers work: your project's `main` branch is your production code; your `package.json` pins `"lodash": "4.17.21"` — not `"latest"` — because you don't want lodash's next release to unexpectedly break your app.

**Summary table:**

| Role | Reference | Rule |
|---|---|---|
| Producer (own repo) | Your own `main` | `main` = truth. Protect it. Require PRs. Tag releases. |
| Consumer (calling shared workflows) | Another repo's workflow | Pin to their version tag, not their `@main` |
| Consumer (using shared modules) | Another repo's published module | Pin to a released version |

### 6.3 Branch Protection on `main`

Every repo should protect `main` with at minimum:

```
Branch protection rules for main:
  ✓ Require pull request before merging
  ✓ Require 1+ approvals (2 for compliance-scoped repos)
  ✓ Dismiss stale approvals when new commits are pushed
  ✓ Require status checks to pass (security-scan, secret-scan, build, lint)
  ✓ Require branches to be up to date before merging
  ✓ Do not allow bypassing the above settings
      (this applies to org owners too — no exceptions)
```

---

## 7. Inner Source — How Patterns Actually Spread

### 7.1 What We Publish

```
Our Org (Enterprise Technology Services)
├── Repository templates
│   ├── tpl-sql-database-project
│   ├── tpl-powershell-module
│   ├── tpl-bicep-landing-zone-module
│   ├── tpl-terraform-aws-module
│   ├── tpl-terraform-azure-module
│   ├── tpl-cloudformation-stack          (CloudFormation = AWS native IaC)
│   ├── tpl-ansible-role                  (Ansible = config management / IaC)
│   ├── tpl-salt-formula                  (Salt/SaltStack = on-prem IaC)
│   ├── tpl-python-data-service
│   ├── tpl-static-docs-site
│   └── tpl-adr-repo
│
├── Reusable workflows (CI/CD building blocks)
│   ├── sql-project-build-deploy.yml
│   ├── powershell-module-test-publish.yml
│   ├── bicep-lint-whatif-deploy.yml
│   ├── terraform-plan-apply-opa.yml
│   ├── cloudformation-validate-deploy.yml
│   ├── ansible-lint-test.yml
│   ├── security-scan-baseline.yml        (Gitleaks + Semgrep + PSScriptAnalyzer)
│   ├── secret-scan-baseline.yml          (Gitleaks custom patterns)
│   ├── iac-security-scan.yml             (Checkov + Trivy)
│   ├── tag-validation.yml                (validates required tags)
│   └── template-compliance-check.yml     (guardrail enforcement)
│
├── Shared modules
│   ├── PowerShell shared module repo
│   ├── Bicep module library
│   ├── Terraform module registry (Azure + AWS)
│   ├── Ansible role library
│   └── Salt formula library
│
├── Approved MCP registry
├── Copilot knowledge bases
└── Platform docs site
```

### 7.2 Versioning and Consumption

**As a producer:** Tag releases from `main`. Use semantic versioning (`v1.0.0`, `v1.1.0`, `v2.0.0`).

**As a consumer:** Reference specific version tags in workflow `uses:` and module imports. Update on your schedule after testing the new version. Document the update in a commit message or PR description.

**Major version bumps** (v1 → v2) indicate breaking changes. Minor bumps (v1.0 → v1.1) are backward-compatible additions. Communicate breaking changes in the CHANGELOG and with advance notice.

### 7.3 Inner Source Triage Rotation

Someone on our team owns triage of external contributions each sprint:
- Acknowledge new issues within 3 business days
- Review external PRs on shared platform repos
- Keep the CONTRIBUTING.md current
- Track and report adoption metrics to the COE

---

## 8. Security Strategy

### 8.1 Anchor Standards

| Standard | Full name | How we apply it |
|---|---|---|
| NIST SP 800-218 | Secure Software Development Framework | Structures CI/CD security controls. Reusable workflows map to SSDF practices. |
| NIST SP 800-207 | Zero Trust Architecture | Shapes OIDC deployment auth, continuous verification, attestation. |
| NIST SP 800-53 | Security and Privacy Controls | Translation layer for control mapping. |
| NIST SP 800-171 | Protecting CUI | Applies where Controlled Unclassified Information (CUI) is processed. |
| CJIS Security Policy | Criminal Justice Information Services | Required for repos touching criminal justice data. |
| CIS Benchmarks | Center for Internet Security | Platform, container, and cloud workload hardening. |
| OWASP SAMM/ASVS | Software Assurance Maturity + Verification | Appsec maturity measurement and verification bar. |
| FedRAMP | Federal Risk and Authorization Management | Cloud service authorization baseline for federal-connected workloads. |
| StateRAMP | State risk and authorization management | State government cloud authorization — aligns with FedRAMP but state-administered. |

### 8.2 Tiered Adoption Model

```
┌──────────────────────────────────────────────────────────────┐
│  TIER 1 — REQUIRED                                           │
│  Set by Enterprise Security at enterprise level              │
│  Applied via rulesets and required workflows                 │
│  Cannot be turned off                                        │
│  Examples: push protection, SSO, audit logging, CJIS rules  │
├──────────────────────────────────────────────────────────────┤
│  TIER 2 — RECOMMENDED                                        │
│  Published by our org and the COE                            │
│  Easy to adopt; adoption is measured and reported            │
│  Examples: reusable workflows, templates, instruction files  │
├──────────────────────────────────────────────────────────────┤
│  TIER 3 — OPTIONAL                                           │
│  Team-specific customization within guardrails               │
│  Must not conflict with Tier 1 or Tier 2                     │
│  Examples: team-specific prompts, additional lint rules      │
└──────────────────────────────────────────────────────────────┘
```

### 8.3 Security Scanning — What Does What and Why

This is the most important section to understand before configuring any pipeline. There are two separate problems, two separate tool families, and one common mistake: treating CodeQL as a comprehensive security scanner for all code types.

**The two problems:**

```
PROBLEM 1: SECRETS AND CREDENTIALS IN CODE
─────────────────────────────────────────────────────────────────
Passwords, API keys, connection strings, certificates, tokens
embedded in PowerShell scripts, SQL files, IaC templates,
config files, YAML, JSON — anywhere a developer typed or
pasted something they shouldn't have.

Tools: GitHub Secret Scanning, Push Protection, Gitleaks, TruffleHog
These tools scan file CONTENT for patterns matching known
credential formats. Language doesn't matter — they scan
every file regardless of extension.

PROBLEM 2: VULNERABLE CODE PATTERNS
─────────────────────────────────────────────────────────────────
Logic flaws, injection risks, insecure function usage, unsafe
data handling patterns — vulnerabilities in how the code works,
not in what strings are embedded in it.

Tools: Semgrep, PSScriptAnalyzer, SonarQube, CodeQL (limited)
These tools analyze code STRUCTURE and PATTERNS.
Language matters — each tool covers specific languages.
```

**Why CodeQL is largely the wrong tool here:**

CodeQL is GitHub's semantic code analysis engine. It is excellent at finding vulnerabilities in C#, Java, JavaScript, Python, Go, Ruby, and Swift. It understands how data flows through a program and flags when untrusted input reaches a dangerous function.

What CodeQL does not support:

```
CodeQL supported languages:
  ✓  C / C++          ✓  Go
  ✓  C#               ✓  Java / Kotlin
  ✓  JavaScript /     ✓  Python
     TypeScript        ✓  Ruby / Swift

NOT supported by CodeQL:
  ✗  PowerShell       ← primary scripting language in this enterprise
  ✗  T-SQL / SQL      ← primary database language
  ✗  Terraform HCL    ← primary cloud IaC
  ✗  Bicep            ← Azure IaC
  ✗  Ansible YAML     ← on-prem config management
  ✗  Salt SLS         ← on-prem config management
  ✗  Bash / Shell     ← automation scripts
```

Running CodeQL on a repo full of `.ps1` and `.sql` files returns zero findings — not because the code is clean, but because the tool skips those files entirely. **CodeQL stays in the toolset only for repos from Enterprise Development that contain C#, Python, or JavaScript.** It is not a default requirement for this environment.

**The right tool for each language and concern:**

```
LANGUAGE / FILE TYPE    SECRET DETECTION        CODE ANALYSIS
──────────────────────  ──────────────────────  ──────────────────────────
PowerShell (.ps1/.psm1) GitHub Secret Scanning  PSScriptAnalyzer
                        Push Protection         Semgrep (PS rules)
                        Gitleaks

T-SQL / SQL (.sql)      GitHub Secret Scanning  Semgrep (SQL rules)
                        Push Protection         SQLFluff (quality)
                        Gitleaks

Terraform (.tf)         GitHub Secret Scanning  Checkov
                        Push Protection         Trivy
                        Gitleaks                tflint (provider rules)
                                                OPA/Sentinel policy gate

Bicep (.bicep)          GitHub Secret Scanning  Checkov
                        Push Protection         bicep lint (built-in)
                        Gitleaks

CloudFormation (.yaml)  GitHub Secret Scanning  cfn-lint
                        Push Protection         cfn-guard
                        Gitleaks

Ansible (.yml)          GitHub Secret Scanning  ansible-lint
                        Push Protection         Semgrep (YAML rules)
                        Gitleaks

Salt (.sls)             GitHub Secret Scanning  salt-lint
                        Push Protection         Semgrep (YAML/Jinja)
                        Gitleaks

C# / Python / JS        GitHub Secret Scanning  CodeQL ← appropriate here
(Enterprise Dev only)   Push Protection         Semgrep
                        Gitleaks
```

**How findings from all tools appear in one place:**

GitHub's code scanning framework accepts **SARIF** (Static Analysis Results Interchange Format — a standard output format for security tool findings) from any tool. Every scanner above outputs SARIF. All findings surface in the GitHub Security tab, the org-wide Security Overview, and stream to the SIEM via audit log. Enterprise Security gets a unified view.

```
SECURITY FINDING FLOW — ALL TOOLS → ONE DASHBOARD

  GitHub Actions pipeline runs on every PR:
  │
  ├── GitHub Secret Scanning       (always on, GitHub native)
  │   Finds: credentials, API keys, connection strings
  │   Coverage: every file type, every language
  │
  ├── Push Protection              (always on, blocks at commit)
  │   Prevents: secrets reaching the repo at all
  │
  ├── Gitleaks                     → SARIF → Security tab
  │   Finds: custom patterns + history scanning
  │   Coverage: every file type
  │
  ├── PSScriptAnalyzer             → SARIF → Security tab
  │   Finds: PS security anti-patterns, credential misuse
  │   Coverage: .ps1, .psm1, .psd1
  │
  ├── Semgrep                      → SARIF → Security tab
  │   Finds: code vulnerabilities, custom org patterns
  │   Coverage: PowerShell, SQL, Terraform, Ansible, YAML, + more
  │
  ├── Checkov                      → SARIF → Security tab
  │   Finds: IaC misconfigurations
  │   Coverage: Terraform, Bicep, CloudFormation, Ansible, Salt
  │
  ├── Trivy                        → SARIF → Security tab
  │   Finds: container vulns, IaC issues, file system scan
  │
  ├── cfn-guard                    → SARIF → Security tab
  │   Finds: CloudFormation policy violations (AWS)
  │
  └── CodeQL                       → SARIF → Security tab
      Finds: code flow vulnerabilities
      Coverage: C#, Python, JS ONLY (Enterprise Dev repos)

  All findings → GitHub Security tab (per repo)
              → GitHub Security Overview (org-wide)
              → SIEM (via audit log streaming)
              → Compliance dashboard (automated evidence)
```

### 8.4 Staged Security Rollout

```
Stage 1: Foundations (invisible to developers)
  ✓ SSO + SCIM enforced
  ✓ Audit log streaming → SIEM
  ✓ Secret scanning enabled across all repos (GitHub native)
  ✓ Push protection on (blocks secrets at commit time)
  ✓ Dependabot alerts enabled
  Exit criteria: all five active, no production breakage

Stage 2: Baseline Code Scanning
  ✓ Gitleaks in baseline security workflow (pilot 5 repos, evaluate mode)
  ✓ PSScriptAnalyzer in workflow for all repos containing .ps1/.psm1
  ✓ Semgrep in workflow for all repos (community rules + custom)
  ✓ Checkov for all repos containing IaC
  ✓ SARIF upload configured so all findings appear in Security tab
  ✓ SBOM generation per build
  ✓ Tag validation workflow
  Exit criteria: >80% pilot repos clean, false-positive noise tuned

Stage 3: Provenance and Attribution
  ✓ OIDC federation for all cloud deployments
  ✓ Signed commits on compliance-scoped repos
  ✓ Build attestations on release artifacts
  ✓ SLSA posture documented
  Exit criteria: zero long-lived cloud credentials in Secrets

Stage 4: Differentiated Policy
  ✓ Rulesets targeting custom properties
  ✓ Stricter controls on cjis/restricted repos
  ✓ Environment protection rules on prod deployments
  ✓ Template compliance checks for agency self-service
  ✓ CodeQL added to Enterprise Dev repos with supported languages
  Exit criteria: all repos tagged, rulesets enforcing

Stage 5: Continuous Evidence
  ✓ Per-repo posture dashboard live
  ✓ Automated evidence bundles
  ✓ Control-set mapping by compliance-scope
  ✓ Audit cycle in days, not weeks
```

### 8.5 Two-Way Interface with Enterprise Security

```
ENTERPRISE SECURITY publishes:      OUR ORG consumes:
  Required policies              ──►  Applied via rulesets
  Required workflows             ──►  Called from all repos
  Content exclusion lists        ──►  Enterprise-level Copilot
  Custom Semgrep rules           ──►  Security scan workflow config
  Custom Gitleaks patterns       ──►  Secret scan workflow config
  Custom secret-scan patterns    ──►  Enterprise-level scan

OUR ORG publishes:                  ENTERPRISE SECURITY consumes:
  Compliance dashboards          ──►  Posture visibility
  Evidence bundles               ──►  Audit package
  Exception requests             ──►  With reasoning attached
  Proposed standards             ──►  For their review/approval

Jointly maintained:
  Responsibility matrix ↔ ADR process ↔ Handoff runbooks
```

---

## 9. Copilot Enterprise Strategy

### 9.1 Enterprise-Level Settings

- **Content exclusions** — credential files, regulated-data paths. Owned by Enterprise Security. Cannot be relaxed by individual orgs.
- **Audit log streaming** — every Copilot interaction logged.
- **Model access** — which models are available; approved at enterprise level.

### 9.2 Knowledge Bases

| Knowledge Base | Published by | Repos included |
|---|---|---|
| Security Standards | Enterprise Security | Policy repos, required workflow repos |
| Data Platform Standards | Our org | SQL automation, backup pipelines, DBA ADRs |
| DevSecOps COE Patterns | Our org | COE decisions, reusable workflows, templates |
| Cloud Landing Zone | Our org | Azure/AWS landing zone, control mappings |
| On-Prem Platform Standards | Our org | VMware, Salt, Ansible runbooks and patterns |
| Common App Frameworks | Enterprise Dev | Libraries, reference implementations |
| State Compliance Standards | Our org + Ent. Sec. | CJIS, NIST control mapping, tagging standards |

### 9.3 Instruction Files and Prompt Files

See the Team Setup Guide for full examples. Every repo in our org should have `.github/copilot-instructions.md` tuned to its archetype. The COE publishes a library of instruction file templates per archetype (SQL, PowerShell, Bicep, Terraform, Ansible, Salt).

---

## 10. On-Premises IaC — VMware, Salt, and Ansible

On-premises infrastructure in our enterprise is managed through the VMware Broadcom stack. This section defines how on-prem IaC integrates into the same DevSecOps pipeline as cloud IaC.

### 10.1 The On-Premises Stack

**VMware Broadcom Environment:**

```
On-Prem IaC Technology Stack
│
├── VMware vSphere / vCenter
│   └── VM provisioning, compute management
│       IaC tooling: Terraform VMware Provider
│                    VMware Aria Automation (formerly vRealize Automation)
│                    PowerCLI (PowerShell-based VMware management)
│
├── VMware NSX
│   └── Software-defined networking, micro-segmentation, firewall rules
│       IaC tooling: Terraform NSX-T Provider
│                    Ansible VMware NSX collection
│
├── VMware Aria Automation (formerly vRealize Automation / vRA)
│   └── Self-service cloud portal for on-prem resources
│       Infrastructure blueprints (templates for VM provisioning)
│       Integration with Salt and Ansible for configuration management
│
├── SaltStack (Salt / Salt Project)
│   └── Configuration management, remote execution, orchestration
│       Formula library (reusable Salt state files)
│       Salt Reactor for event-driven automation
│       Integration: Aria Automation → Salt for post-provision config
│
└── Ansible
    └── Agentless configuration management and application deployment
        Role library (reusable Ansible roles)
        Ansible Navigator + Execution Environments (containerized runs)
        Integration: Aria Automation → Ansible for app configuration
```

**Key distinction:**
- **Salt** — agent-based (Salt minions installed on managed nodes), better for real-time state enforcement and large-scale fleets. Strong on-prem VMware integration.
- **Ansible** — agentless (uses SSH/WinRM), better for one-time provisioning tasks and application deployment. Better for cross-platform and cloud-adjacent workflows.
- **Use both** where they complement each other; don't pick one to replace the other.

### 10.2 IaC Pipeline for On-Prem Deployments

```
ON-PREM INFRASTRUCTURE PIPELINE

  Developer/Engineer writes:
    Terraform plan (.tf files)     → VMware vSphere + NSX resources
    Salt formula (.sls files)      → Configuration state for managed nodes
    Ansible role/playbook (.yml)   → Application and OS configuration

  Pipeline (GitHub Actions):
    ┌─────────────────────────────────────────┐
    │ Lint + validate                         │
    │   terraform validate                    │
    │   terraform fmt --check                 │
    │   tflint (Terraform linter)             │
    │   ansible-lint                          │
    │   salt-lint (Salt state linter)         │
    └──────────────┬──────────────────────────┘
                   │
    ┌──────────────▼──────────────────────────┐
    │ Security scan                           │
    │   Checkov (Terraform IaC scan)          │
    │   Trivy (file system + IaC scan)        │
    │   KICS (infrastructure security scan)   │
    └──────────────┬──────────────────────────┘
                   │
    ┌──────────────▼──────────────────────────┐
    │ Plan / dry-run                          │
    │   terraform plan → output as PR comment │
    │   ansible-playbook --check              │
    │   salt-call state.show_sls (dry run)    │
    └──────────────┬──────────────────────────┘
                   │ (PR approval + manual approval for prod)
    ┌──────────────▼──────────────────────────┐
    │ Apply (after merge to main + approval)  │
    │   terraform apply                       │
    │   Aria Automation blueprint trigger     │
    │   Ansible playbook run                  │
    │   Salt highstate                        │
    └──────────────┬──────────────────────────┘
                   │
    ┌──────────────▼──────────────────────────┐
    │ Verify + evidence                       │
    │   Terraform state validation            │
    │   Salt mine / grains verification       │
    │   Compliance evidence stored            │
    └─────────────────────────────────────────┘
```

### 10.3 VMware and GitHub Actions Integration

GitHub Actions runners for on-prem workloads run as **self-hosted runners** (GitHub-hosted runners can't reach your internal VMware network):

```
Self-Hosted Runner Setup:
  Runners deployed as VMs in vSphere
  Register with GitHub Actions (runner token, scoped to org)
  Label runners by capability: [vmware, on-prem, salt, ansible]
  Workflow references runners by label:
    runs-on: [self-hosted, vmware, on-prem]

Security for self-hosted runners:
  ✓ Runners run in dedicated VM cluster (not shared with prod workloads)
  ✓ Network segmented: can reach vCenter/NSX, cannot reach internet
  ✓ Runner identity managed via gMSA (Group Managed Service Account)
  ✓ Ephemeral runners (each job gets a fresh runner, then it's deleted)
  ✓ Runner logs streamed to SIEM
```

### 10.4 Terraform for VMware — Provider Reference

```hcl
# Terraform VMware vSphere provider
terraform {
  required_providers {
    vsphere = {
      source  = "hashicorp/vsphere"
      version = "~> 2.6"   # Pin to minor version
    }
    nsxt = {
      source  = "vmware/nsxt"
      version = "~> 3.3"
    }
  }
}

# State stored in a secure backend (not local files)
# Options: Terraform Cloud, S3+DynamoDB, Azure Blob with lock
```

---

## 11. Cloud IaC Standardization by Platform

Each cloud platform has a preferred native IaC tooling and a cross-platform alternative. We standardize so that every deployment method has a known pipeline, known guardrails, and known compliance posture.

### 11.1 Platform Matrix

| Platform | Native IaC | Cross-Platform IaC | Our Standard |
|---|---|---|---|
| Microsoft Azure | Bicep (preferred), ARM templates | Terraform | Bicep for new; Terraform for multi-cloud or existing Terraform estate |
| Amazon Web Services | CloudFormation, CDK | Terraform | Terraform for new; CloudFormation for AWS-native services where Terraform coverage is weak |
| VMware On-Premises | Aria Automation Blueprints, PowerCLI | Terraform (vsphere provider) | Terraform for vSphere+NSX; Aria Automation for self-service portal |
| Multi-Cloud / Hybrid | — | Terraform | Terraform as the standard cross-platform tool when managing resources on 2+ platforms in one pipeline |

**Why not one tool for everything?** Bicep has deeper Azure feature coverage and faster new-resource support than Terraform's azurerm provider. CloudFormation is tightly integrated with AWS IAM and native services. Forcing everything through Terraform means being behind on new platform features. We use the right tool per platform and standardize the *pipeline pattern*, not the syntax.

### 11.2 Azure Deployment Pipeline

```
Azure IaC Pipeline (Bicep or Terraform)

  Code (Bicep .bicep or Terraform .tf)
    │
    ├── Lint: bicep lint / terraform fmt + validate
    ├── Security: Checkov (bicep/terraform), tflint for Azure rules
    ├── What-if: az deployment group what-if → PR comment
    │            terraform plan → PR comment
    │
    ├── PR Approval: CODEOWNERS on /infra/ paths
    │               Required: platform team + security team review
    │
    ├── Merge to main (triggers deploy job)
    │
    ├── Environment protection: required manual approval for prod
    │
    ├── Deploy: az deployment group create (Bicep)
    │           terraform apply (Terraform)
    │           Auth: OIDC only (no stored credentials)
    │
    └── Evidence: deployment log, SBOM, attestation stored

Mandatory tags on every Azure resource (see §14):
  environment, owning-team, agency-code, data-classification,
  cost-center, compliance-scope, lifecycle
```

### 11.3 AWS Deployment Pipeline

```
AWS IaC Pipeline (Terraform or CloudFormation)

  Code (.tf or CloudFormation .yaml/.json templates)
    │
    ├── Lint: tflint (AWS rules) / cfn-lint (CloudFormation linter)
    ├── Security: Checkov (terraform+cfn), cfn-guard (AWS rule enforcement)
    │            cfn-guard is critical for guardrail enforcement (see §13)
    │
    ├── Plan/Change set: terraform plan / CloudFormation change set
    │                    Output posted to PR
    │
    ├── PR Approval: CODEOWNERS, required reviews
    │
    ├── Merge to main
    │
    ├── Environment protection: manual approval for prod
    │
    ├── Deploy: terraform apply / aws cloudformation deploy
    │           Auth: OIDC (GitHub Actions → AWS IAM Identity Center)
    │
    └── Evidence: deployment log, SBOM, stack tags verified

AWS Config rules enforced after deployment:
  Verify required tags present
  Verify required security controls active
  Alert on drift from approved state
```

### 11.4 Consistency Across Platforms

Regardless of which platform or IaC tool:
- Pipeline structure is the same (lint → security → plan → approve → deploy → verify)
- OIDC authentication (no long-lived credentials)
- Required tags validated before deployment completes
- Evidence (SBOM, attestation, deployment log) stored per deployment
- Post-deployment drift detection enabled

---

## 12. Customer / Agency Deployment Process

Customer and agency orgs need a path to build and deploy their own cloud resources that is:
- **Compliant** — meets enterprise and state government security standards automatically
- **Autonomous** — they can move without waiting for us on every change
- **Guardrailed** — they can't accidentally (or intentionally) break required controls
- **Auditable** — every deployment is traceable and evidenced

### 12.1 The Self-Service Model

```
CUSTOMER/AGENCY SELF-SERVICE DEPLOYMENT FLOW

  Agency team writes IaC (CloudFormation, Bicep, Terraform)
  in their own repo within the GitHub enterprise
        │
        ▼
  Template compliance check runs automatically
  (validates against enterprise-approved base templates)
  [see §13 for guardrail details]
        │
        ├── PASS: Proceed to security scan
        └── FAIL: PR blocked with specific violations listed
                  Agency must correct before proceeding
        │
        ▼
  Security scan (Checkov, tflint, cfn-guard)
  Tag validation (required tags present and valid)
        │
        ├── PASS: Plan/what-if runs, posted to PR
        └── FAIL: PR blocked, specific issues listed
        │
        ▼
  Plan/what-if output reviewed by agency team
  PR approval: agency team lead + (auto-requested) our platform team
               for any change touching networking or IAM resources
        │
        ▼
  Merge to main (triggers deployment pipeline)
        │
        ▼
  DEVELOPMENT / TEST ENVIRONMENTS:
    Automatic deployment after merge
    No additional approval required
    Deployment logged and evidence stored
        │
        ▼ (after testing, promote via separate PR/pipeline)
  PRODUCTION ENVIRONMENT:
    Requires manual approval (GitHub Environment protection)
    Approver: agency tech lead + our platform team on-call
    Time window enforced (deployments only during defined hours)
    Full evidence package generated and stored
        │
        ▼
  Post-deployment:
    Drift detection (AWS Config, Azure Policy)
    Alert if deployed state diverges from approved template
    Compliance evidence stored for audit
```

### 12.2 What "Autonomy" Means

Autonomy does not mean unconstrained. It means agencies can:
- Write their own application code and configuration
- Choose which approved template to build on
- Deploy to their environments on their schedule
- Modify resources within the bounds of the template
- Open issues against our platform team for new capabilities

Autonomy does **not** mean:
- Disabling required security controls
- Bypassing tag requirements
- Removing audit logging from their environments
- Deploying to production without approval
- Using unapproved base images or modules

This distinction is enforced by guardrails (§13), not by trust.

### 12.3 Environment Tiers for Agencies

```
AGENCY ENVIRONMENT TIERS

Sandbox / Exploration
  Purpose: Learning, experimentation, proof-of-concept
  Controls: Basic (tag validation, no sensitive data)
  Approval: Automatic deployment on PR merge
  Cost: Spend limit enforced at cloud account level

Development / Test
  Purpose: Active development, integration testing
  Controls: Standard (all security scans, required tags)
  Approval: Automatic after checks pass
  Cost: Budget alert at 80%, hard limit at 100%

Pre-Production / Staging
  Purpose: Final validation before production
  Controls: Full production controls applied
  Approval: Agency tech lead approval required
  Cost: Budget managed same as production

Production
  Purpose: Live citizen-facing or operational systems
  Controls: Full (all scans, CJIS if applicable, drift detection)
  Approval: Agency tech lead + our platform on-call
  Deployment window: Defined schedule (e.g., Tue/Thu 10am-2pm)
  Cost: Budget with hard stop and alert chain
```

---

## 13. Template Guardrails and Policy Enforcement

Agencies and teams can customize their deployments, but required controls cannot be removed or modified outside the bounds of the enterprise template. This is enforced in the pipeline, not just documented in policy.

### 13.1 How Guardrails Work

The enterprise publishes **base templates** — IaC starting points with required controls built in. Agencies extend the templates; they cannot delete required blocks.

```
GUARDRAIL ENFORCEMENT MODEL

  Enterprise publishes base template:
    /templates/azure-agency-landing-zone.bicep
    /templates/aws-agency-account-foundation.tf
    /templates/vmware-agency-vm-standard.tf

  Agency forks/extends the template:
    They add their application resources
    They adjust parameters (VM size, region, etc.)
    They CANNOT remove:
      - Required security groups / NSGs (Network Security Groups)
      - Audit logging configuration
      - Encryption settings
      - Required tags
      - Mandatory security controls

  Enforcement mechanism:
    Policy-as-code (cfn-guard / OPA / Azure Policy)
    runs in the PR pipeline BEFORE any deployment
    and BLOCKS the PR if required blocks are absent or modified
```

### 13.2 AWS — cfn-guard Rules Example

```
# cfn-guard rule: S3 buckets must have encryption and logging

rule s3_bucket_encryption_required {
  when AWS::S3::Bucket EXISTS {
    Properties.BucketEncryption EXISTS
    Properties.BucketEncryption.ServerSideEncryptionConfiguration[*]
      .ServerSideEncryptionByDefault.SSEAlgorithm == "AES256"
      OR "aws:kms"
  }
}

rule s3_bucket_logging_required {
  when AWS::S3::Bucket EXISTS {
    Properties.LoggingConfiguration EXISTS
  }
}
```

### 13.3 Azure — Azure Policy and Bicep Linting

Azure Policy (a service that evaluates Azure resources against rules) can enforce at deployment time:
- Require specific tags on all resources
- Require encryption on storage accounts
- Require private endpoints (no public internet exposure)
- Deny creation of resources not on the approved SKU (pricing tier) list

These policies are assigned at the Management Group or Subscription level — they apply regardless of how the resource was created (portal, CLI, Bicep, Terraform).

### 13.4 Terraform — Sentinel and OPA Gates

```
TERRAFORM POLICY GATE (OPA example)

  Policy: all resources must have required tags

  package terraform.tagging

  required_tags := {
    "environment", "owning-team", "agency-code",
    "data-classification", "compliance-scope"
  }

  deny[msg] {
    resource := input.resource_changes[_]
    resource.change.actions[_] == "create"
    tag_keys := {k | resource.change.after.tags[k]}
    missing := required_tags - tag_keys
    count(missing) > 0
    msg := sprintf("Resource %v missing required tags: %v",
                   [resource.address, missing])
  }

Policy gate runs during terraform plan:
  Pass → plan output posted to PR, deployment can proceed
  Fail → PR blocked, missing tags listed with remediation guidance
```

### 13.5 Drift Detection

Deployment is only the beginning. Post-deployment:
- **AWS Config** — continuously evaluates deployed resources against rule sets. Alerts on drift.
- **Azure Policy** — compliance scan runs continuously. Non-compliant resources are reported.
- **Terraform state** — periodic `terraform plan` in read-only mode detects manual changes (out-of-band changes that weren't made through IaC).

Drift alerts go to:
1. Agency's designated contact
2. Our platform team Slack/Teams channel
3. SIEM (Security Information and Event Management) for audit trail

---

## 14. State Government Tagging Standards

Tags are how cloud resources are attributed, governed, audited, and billed. For state government, tags must satisfy: cost allocation (legislative budget accountability), security classification (CJIS, HIPAA, etc.), operational management, and audit requirements.

### 14.1 Why Tags Matter for Government

State governments must demonstrate:
- **Legislative accountability** — every dollar of cloud spend attributed to a budget line
- **Compliance evidence** — which resources contain which classes of data
- **Audit trails** — who deployed what, when, and under which authority
- **Security posture** — classification level determines required controls

Un-tagged or incorrectly-tagged resources fail audit. They may also fail to trigger required security controls that are applied by tag.

### 14.2 Required Tags (All Resources)

| Tag Key | Required Values | Definition |
|---|---|---|
| `environment` | sandbox, dev, test, staging, prod | Lifecycle stage. Determines control level. |
| `agency-code` | State-assigned agency identifier | Which agency owns this resource. Billing attribution. |
| `department-code` | Department within the agency | Sub-attribution for large agencies |
| `cost-center` | Budget line code | Legislative budget attribution |
| `owning-team` | Team identifier | Operational contact |
| `data-classification` | public, internal, confidential, restricted | Sensitivity level. Determines security controls. |
| `compliance-scope` | none, cjis, hipaa, pci, sox, fisma, stateramp | Regulatory requirements. Determines audit controls. |
| `project-name` | Project or program name | Initiative attribution |
| `lifecycle` | active, maintenance, scheduled-decommission, decommissioned | For resource management and cleanup |
| `created-by` | Pipeline identifier or user (automated preferred) | Audit trail for provisioning |
| `created-date` | ISO 8601 date (YYYY-MM-DD) | Audit trail |
| `iac-managed` | true, false | Whether resource is managed by IaC (and which tool) |
| `iac-repo` | GitHub repo URL | Link back to the source of truth |

### 14.3 CJIS-Specific Tag Requirements

Any resource where CJIS-governed data may be processed, stored, or transmitted requires additional tags:

| Tag Key | Required Values | Purpose |
|---|---|---|
| `cjis-data-type` | chi, nlets, ncic, biometric, other | Type of criminal justice data |
| `cjis-agency-ori` | ORI number (FBI-assigned) | Originating agency identifier |
| `cjis-audit-category` | direct, indirect, none | Level of CJIS audit requirement |
| `data-residency` | state-name, CONUS, federal | Where data must physically reside |

### 14.4 Tag Validation Pipeline

Every deployment pipeline includes a tag validation step. It runs before the plan and blocks the pipeline if required tags are missing or use invalid values.

```
Tag validation flow:
  1. Extract tags from IaC plan output
  2. Check: all required tags present?
     → NO: fail with list of missing tags and valid values
  3. Check: all values valid (from approved value lists)?
     → NO: fail with invalid tag and valid options
  4. Check: data-classification and compliance-scope consistent?
     → CJIS-tagged data must have data-classification = restricted
  5. Check: iac-repo points to the actual calling repo?
     → NO: fail (prevents tag spoofing)
  6. PASS: continue to security scan
```

### 14.5 Cost Allocation and Reporting

Tags feed:
- **Cloud billing dashboards** — spend by agency, department, project, environment
- **Budget alerts** — per agency-code + environment combination
- **Chargeback** — agencies with self-managed cloud billed based on agency-code tag
- **Legislative reports** — annual cloud spend report broken down by agency and program

---

## 15. Reference Models from Leading States

Several states have implemented substantial portions of what we are building. Their experiences — including documented failures — are a valuable reference. We study them to learn and then go further.

### 15.1 Key State References

**Colorado — OIT (Office of Information Technology)**
- Early cloud-first policy with centralized procurement and standardized landing zones
- Established shared services model for cloud infrastructure
- Lesson learned: centralized cloud brokerage created bottlenecks; they moved to a federated model with guardrails — which is what our architecture does

**California — CDT (California Department of Technology)**
- "Cloud Smart" strategy: not cloud-first but cloud-right
- Statewide security baseline applied through policy-as-code
- Lesson learned: agencies with no prior cloud experience need more hand-holding than a golden path alone provides — need the managed/hybrid/self-sufficient tiering we've built into §12

**Virginia — VITA (Virginia Information Technologies Agency)**
- Managed services model with strong central governance
- Shared cloud environment with per-agency cost allocation
- Lesson learned: tagging standards implemented late caused retroactive pain — implement tagging on day one (§14 exists because of this lesson)

**Michigan — DTMB (Department of Technology, Management and Budget)**
- Centralized GitHub Enterprise with org-per-agency model
- Security policy-as-code with automated compliance evidence
- Closest to our architecture; their published guidance is worth reading

**Utah — DTS (Division of Technology Services)**
- Early DevSecOps adoption with a COE model similar to ours
- Reusable workflow library published to all agencies
- Lesson learned: COE without executive sponsorship stalls — our COE needs visible leadership backing, not just technical advocates

**Washington — WaTech**
- CJIS-compliant cloud deployment process for law enforcement agencies
- CJIS tagging and control automation (informs §14.3)
- Published their CJIS cloud alignment guide — worth adopting as a reference

### 15.2 Where We Go Further

What most of these states have not yet fully implemented:
- **MCP integration** — AI tools grounded in live infrastructure state
- **Inner source at enterprise scale** — cross-agency template adoption via inner source patterns
- **Unified on-prem + cloud pipeline** — most have separate on-prem and cloud processes
- **Drift detection with automatic remediation** — most detect drift; few auto-remediate within policy
- **Fully automated compliance evidence** — most still have manual evidence-gathering steps

These are the areas where we innovate beyond the reference models.

---

## 16. Copilot Enterprise Strategy (expanded)

### 16.1 Enterprise-Level Settings

- Content exclusions: credential files, CJIS-data paths, production connection strings. Owned by Enterprise Security. Immutable.
- Audit logging: every Copilot interaction logged alongside GitHub events.
- Model access: approved models list maintained at enterprise level.

### 16.2 Knowledge Bases

See §9.2. The state compliance knowledge base (§9.2) is particularly important — grounding Copilot answers in state government requirements rather than generic NIST summaries.

### 16.3 MCP Registry

See §17. The MCP registry is a COE deliverable. Enterprise Security reviews each Tier 1 server. Servers with access to CJIS or restricted data require a formal security review before any tier assignment.

---

## 17. MCP at Enterprise Scale

### 17.1 Central Hosting Pattern

Remote MCP servers behind enterprise SSO/OIDC. Developers configure a URL; the server authorizes by identity. No personal credentials in config files.

### 17.2 Approved Registry

| Server | Tier | Auth | Data classification |
|---|---|---|---|
| GitHub MCP | 1 — freely usable | SSO session | Internal |
| Filesystem (scoped) | 1 — freely usable | None | Internal |
| Fetch / Web | 1 — freely usable | None | Public only |
| Terraform Registry MCP | 1 — freely usable | None | Public |
| Ansible Galaxy MCP | 1 — freely usable | None | Public |
| SQL Dev MCP | 2 — request needed | SSO+OIDC | Confidential (dev only) |
| Azure MCP | 2 — request needed | SSO+OIDC | Confidential (sandbox) |
| AWS MCP | 2 — request needed | SSO+OIDC | Confidential (sandbox) |
| VMware Aria MCP | 2 — request needed | SSO+OIDC | Confidential (dev only) |
| CMDB MCP | 2 — request needed | SSO | Internal |
| Kusto / ADX MCP | 2 — request needed | SSO+OIDC | Confidential |
| ServiceNow MCP | 2 — request needed | SSO+OIDC | Confidential |
| CJIS-data systems | 3 — prohibited | — | Restricted |
| Production databases | 3 — prohibited local | — | Restricted |

---

## 18. Glossary

Every term is also defined inline the first time it appears in a section. This glossary is for quick reference.

**ADR (Architecture Decision Record).** A document recording one decision, its context, and its consequences. Numbered. Immutable once accepted.

**Ansible.** An agentless open-source configuration management and deployment tool. Uses YAML playbooks. Connects via SSH or WinRM. No agent needed on target nodes.

**Aria Automation.** VMware Broadcom's self-service infrastructure portal (formerly vRealize Automation / vRA). Manages blueprints for on-premises VM provisioning.

**Attestation.** A signed, verifiable record of how a software artifact was built.

**BCP (Bulk Copy Program).** A SQL Server command-line utility for high-speed bulk data import/export.

**Bicep.** Microsoft's language for defining Azure infrastructure as code.

**CJIS (Criminal Justice Information Services).** FBI division setting security policy for systems handling criminal justice data.

**CUI (Controlled Unclassified Information).** Federal information that requires safeguarding per law or policy but is not classified.

**cfn-guard.** AWS's open-source policy-as-code tool for CloudFormation templates. Enforces organizational rules at deployment time.

**CloudFormation.** AWS's native IaC service. Defines AWS resources in JSON or YAML templates.

**CodeQL.** GitHub's semantic code analysis engine. Finds security vulnerabilities by analyzing code flow. Supports C#, Java, JavaScript, Python, Go, Ruby, Swift. Does NOT support PowerShell, T-SQL, Terraform, Bicep, Ansible, or Salt — and therefore has limited applicability in this environment. Use only on Enterprise Development repos with supported languages.

**GHAS (GitHub Advanced Security).** A set of GitHub security features including Secret Scanning, Push Protection, Code Scanning (SARIF-based), and Dependabot. Licensed at the enterprise level. Secret Scanning and Push Protection are the highest-priority features for this environment.

**CODEOWNERS.** A file listing who reviews changes to specific paths in a repo.

**COE (Center of Excellence).** A cross-org body producing shared standards, patterns, and tools. Artifact-producing, not just advisory.

**Copilot Enterprise.** GitHub's highest-tier AI coding assistant.

**Drift.** When the actual deployed state of infrastructure differs from what the IaC template defines.

**EMU (Enterprise Managed Users).** GitHub enterprise configuration where all accounts are managed by the organization.

**FedRAMP.** Federal Risk and Authorization Management Program. Cloud service authorization standard for federal agencies.

**GCM (Git Credential Manager).** Secure storage for Git authentication tokens.

**GHAS (GitHub Advanced Security).** GitHub's security feature set including Secret Scanning, Push Protection, Code Scanning (SARIF-based), and Dependabot. Secret Scanning and Push Protection are the highest-priority features for this environment.

**Gitleaks.** Open-source secret scanner. Scans any file type for credential patterns including custom patterns specific to your organization. Runs in CI/CD. Outputs SARIF so findings appear in the GitHub Security tab.

**PSScriptAnalyzer.** Microsoft's PowerShell static analysis tool. Finds security anti-patterns, credential misuse, deprecated cmdlets, and quality violations in PowerShell files. The correct code analysis tool for PowerShell repos — not CodeQL.

**Push Protection.** A GitHub GHAS feature that blocks a commit before it reaches the repo if it contains a recognized secret pattern. The most effective preventive control for credential leakage. Should be enabled enterprise-wide before any other scanning tool.

**SARIF (Static Analysis Results Interchange Format).** A standard format for security tool output. Any tool that outputs SARIF can upload findings to the GitHub Security tab. This is how Semgrep, PSScriptAnalyzer, Gitleaks, Checkov, and other tools produce a unified security view in GitHub.

**Semgrep.** Open-source and commercial rule-based code scanner. Supports PowerShell, SQL, Terraform, Ansible, YAML, and many other languages. Allows custom rules for organization-specific patterns. Outputs SARIF. Fills the gap CodeQL leaves for PowerShell, SQL, and IaC.

**TruffleHog.** Open-source secret scanner specializing in Git history — finds secrets that were committed and then deleted. They still exist in history. Useful for auditing repos that predate push protection.

**gMSA (Group Managed Service Account).** A Windows Active Directory account type for services, with automatically rotated passwords.

**IaC (Infrastructure as Code).** Defining infrastructure in configuration files instead of manual clicks.

**IAM (Identity and Access Management).** Managing who can access what — authentication and authorization systems.

**Inner source.** Open-source practices applied to internal enterprise repos.

**KICS (Keeping Infrastructure as Code Secure).** An open-source IaC security scanner by Checkmarx.

**MCP (Model Context Protocol).** Standard that lets AI tools connect to external systems.

**NSG (Network Security Group).** Azure firewall rule set attached to a subnet or network interface.

**OPA (Open Policy Agent).** Policy-as-code engine. Write policies in Rego; OPA returns allow/deny.

**ORI.** Originating Agency Identifier. An FBI-assigned identifier for CJIS-authorized agencies.

**OIDC (OpenID Connect).** Identity protocol enabling short-lived cloud credentials from GitHub Actions.

**PR (Pull Request).** A proposal to merge code changes, reviewed before merging.

**RACI.** Responsible, Accountable, Consulted, Informed.

**RPO/RTO (Recovery Point/Time Objectives).** RPO = how much data loss is acceptable. RTO = how long a system can be down.

**Ruleset.** A GitHub policy object applying rules to repos matching criteria.

**Salt / SaltStack.** Agent-based configuration management and remote execution tool. Strong on-prem presence.

**SBOM (Software Bill of Materials).** Complete inventory of all components in a build.

**SCIM.** System for Cross-domain Identity Management. Automates user account provisioning.

**SIEM.** Security Information and Event Management. Collects and correlates security events.

**SLSA.** Supply-chain Levels for Software Artifacts. Build provenance verification framework.

**SoD (Separation of Duties).** Builder ≠ auditor. A security control preventing unchecked power.

**SSDF.** Secure Software Development Framework. NIST SP 800-218.

**SSO (Single Sign-On).** One set of credentials grants access to multiple systems.

**StateRAMP.** State-level equivalent of FedRAMP for state government cloud authorization.

**T-SQL (Transact-SQL).** Microsoft's SQL Server dialect.

**Terraform.** Open-source multi-cloud IaC tool by HashiCorp.

**tflint.** Terraform linter. Catches provider-specific issues and style violations.

**Trivy.** Open-source scanner for container images, file systems, and IaC.

**VMware / Broadcom.** VMware (now owned by Broadcom) produces the enterprise hypervisor and on-premises cloud stack we use for on-premises infrastructure.

**Zero Trust.** Architectural principle: never assume trust by location. Verify every identity, device, and request.

---

## 19. Getting Started — COE Roadmap

```
Month 1: Foundation documents
  ☐ Publish this strategy in the COE repo
     → ADR-0001: Adopt Multi-Org DevSecOps Strategy
  ☐ Draft responsibility matrix
     → ADR-0002: Responsibility Allocation v1
  ☐ Tag top 50 repos with custom properties
  ☐ Establish team structure in GitHub (§5)
     → ADR-0003: ETS GitHub Teams and Permissions Model

Month 2: First artifacts
  ☐ Stand up first Copilot knowledge base (Data Platform Standards)
  ☐ Publish first repo template (tpl-powershell-module or tpl-sql-database-project)
  ☐ Publish MCP registry with Tier 1 entries
  ☐ Establish self-hosted runner pool for on-prem workloads

Month 3: Security foundations
  ☐ Turn on audit log streaming → SIEM
     → ADR-0004: Audit Log Streaming
  ☐ Pilot security scan workflow on 5 volunteer repos (evaluate mode)
  ☐ Define tag validation workflow and publish it
  ☐ Begin branch protection standardization on all ETS repos

Month 4: Pipeline standardization
  ☐ Publish on-prem IaC pipeline (Terraform + Ansible + Salt)
  ☐ Publish Azure deployment pipeline (Bicep / Terraform)
  ☐ Publish AWS deployment pipeline (Terraform / CloudFormation)
  ☐ Begin template library (first 2 archetypes)

Month 5-6: Agency enablement
  ☐ Launch agency self-service sandbox environment
  ☐ Publish agency deployment process documentation
  ☐ Implement template guardrail enforcement in pipelines
  ☐ Tag validation goes live (evaluate mode first)
  ☐ First monthly showcase — invite agency teams

Ongoing: Measure, expand, iterate
  ☐ COE weekly working session: move backlog
  ☐ Biweekly review: what shipped, what's next
  ☐ Monthly showcase: build adoption
  ☐ Quarterly: audit org-owner access, review responsibility matrix
```

---

## 20. What This Document Is Not

Not a final plan. Not a mandate. Not a complete reference for every tool or scenario. Customer orgs will have needs we have not anticipated. Enterprise Security will have priorities that reorder ours.

The goal is enough shared vocabulary, enough published artifacts, and enough structured processes that every conversation starts from the same baseline — and enough humility to keep updating that baseline as we learn.

---

*Maintained by the DevSecOps COE. Contributions via PR welcome. Major changes require an ADR. Minor corrections (typos, glossary additions, clarifications) can be submitted as PRs without an ADR. Version history is in the Git log.*
