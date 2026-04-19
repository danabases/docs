# SQL Server Compliance Audit Deployment — GitHub Copilot Project Prompt

Use this as the seed prompt in VS Code with GitHub Copilot (chat or inline)
to continue implementing, debugging, and testing the fleet-wide SQL Server
audit deployment tooling. Paste the whole file into a new Copilot chat,
or keep it open in the workspace so Copilot has it in context.

-----

## Project goal

Build and maintain a PowerShell + T-SQL deployment toolkit that replaces an
aging `.sqlaudit` → zip → FTP collection pipeline across roughly 100 SQL
Server instances (versions 2016 through 2025) with a modern, centrally
orchestrated, AG-aware compliance audit configuration. All work is driven
from a Central Management Server (CMS) using SQL connections only —
no PowerShell remoting, no SMB file operations, no agent on the targets.

## Critical environmental constraints

- **Firewalls are unidirectional: CMS → target only.** Nothing can
  initiate a connection back to the CMS host.
- **No PowerShell remoting.** All orchestration runs via `Invoke-Sqlcmd`
  from the CMS host (or a workstation with CMS access).
- **No SMB from CMS to targets.** Cannot `\\server\share\...` to read or
  write files on targets.
- **`xp_cmdshell` can be enabled temporarily if needed.** The script must
  save the original state, toggle if required, and always restore on exit.
  No blanket policy block exists.
- **Target SQL versions**: 2016, 2017, 2019, 2022, 2025. Cross-version
  compatible T-SQL required. Avoid version-gated features
  (no `STRING_AGG` — use `STUFF + FOR XML PATH`; no
  `SENSITIVE_BATCH_COMPLETED_GROUP`; etc.).
- **Runtime**: Windows PowerShell 5.1 is the primary tested platform.
  PowerShell 7 is acceptable but has known quirks with the
  `SQLSERVER:\SQLRegistration\` provider, especially with named-instance
  CMS hosts.
- **SqlServer module**: 21.1.x or newer (for `-TrustServerCertificate`
  support). Script should detect and warn if older.

## Server storage layout (standardized across the fleet)

- **Data drive**: `.mdf` files, error logs, maintenance logs, profiler
  output. High write activity, should be avoided for new workloads.
- **Transaction log drive**: `.ldf` files only.
- **Tempdb drive**: `\MSSQL\tempdb\` for tempdb files, and `\MSSQL\LOG\`
  for SQL’s operational output (`ERRORLOG`, `SQLAGENT.OUT`, system_health
  XE, default traces). Lower sustained pressure — good candidate for
  new audit workload.
- **Audit drive / folder**: During instance build-out, a dedicated
  `AuditLog` folder is pre-created (path stored in
  `sys.server_file_audits.log_file_path` when an audit is already
  configured). This is the intended destination for future audit files.

## Current state to replace

Each instance already has **some** server audit configured (path visible
in `sys.server_file_audits`), but:

- The current audit’s `log_file_path` is **on the `.ldf` drive** —
  the transaction log drive — which is not the desired location.
- The audit configuration is old, inconsistent across the fleet, and
  its collection pipeline (zip + FTP) is being retired.
- The existing server audit and server audit specification must be
  dropped and replaced with a new standardized version.
- The new audit’s `FILEPATH` should be the pre-existing `AuditLog`
  folder on the **tempdb drive** (not the ldf drive).

## How to determine the target audit path

**Priority order for resolving the audit file path per instance:**

1. **Query `sys.server_file_audits`** for any existing audit’s
   `log_file_path` value. The desired `AuditLog` folder path is often
   already discoverable here — but note: the *currently configured*
   path is on the ldf drive, which is wrong. Need to detect whether
   the path is on the tempdb drive or elsewhere.
1. **If the current audit path is on the tempdb drive**, reuse it
   directly.
1. **If the current audit path is elsewhere** (e.g. on the ldf drive),
   query `SERVERPROPERTY('ErrorLogFileName')` to find the `\MSSQL\LOG\`
   directory on the tempdb drive, then construct the target as
   `<drive>:\<instance-root>\AuditLog\` or a comparable sibling folder
   based on the deployed convention. Confirm the folder exists via
   `xp_cmdshell DIR` before use.
1. **If the `AuditLog` folder exists adjacent to `\MSSQL\LOG\`**
   (common post-install convention), prefer that path over the
   ERRORLOG directory itself — keeps audit files separated from
   SQL’s own operational output.
1. Fall back to the error log directory as last resort.

The path resolution logic should be captured in the pre-check pass and
stored in the in-memory lookup table so the deployment pass can simply
consume the resolved path per row.

## Design history and constraints already decided

Carry these forward; they represent deliberate choices made across
prior iterations:

### Architecture

- **Three-phase execution**:
1. Enumerate CMS paths → server list
1. Pre-check pass → build in-memory lookup (one connection per server)
1. GUID assignment → AG-aware coordination
1. Deploy pass → serial per-instance deployment using lookup
- **Lookup table in memory**, not in a temp table. A
  `Dictionary[string, object]` with `StringComparer.OrdinalIgnoreCase`
  keyed by CMS-registered instance name.
- **Serial processing only.** One instance at a time end-to-end.
  No parallelism. If a run is interrupted, already-deployed
  instances stay consistent.
- **CMS path targeting** via user-supplied `-CmsPath` array.
  Non-recursive by default; `-Recurse` switch included. Multiple paths
  supported, results deduplicated via `Sort-Object -Unique`.
- **Named-instance CMS hosts** (e.g. `CMS01\PROD`): always use
  `-LiteralPath` on provider operations.

### Availability Group handling

- **One GUID per AG**, shared across all replicas. Generated fresh
  per AG at assignment time using `[guid]::NewGuid()`.
- **Standalone instances** get their own unique GUIDs.
- **AG detection** via `sys.availability_groups` joined to
  `sys.availability_replicas`, gated by
  `SERVERPROPERTY('IsHadrEnabled') = 1`.
- **Peer enumeration** uses `STUFF + FOR XML PATH(N'')` for
  cross-version compatibility (no `STRING_AGG`).
- **All AG replicas expected to be registered in the same CMS folder.**
  If the AG topology reports a peer not present in the lookup, warn
  and log — do not silently deploy.
- **Matching CMS names to AG replica names**: case-insensitive.
  CMS can be inconsistent with `@@SERVERNAME` format.

### `xp_cmdshell` handling

Previous iterations included full save/restore logic. The v3 script
removed it because audit files were to be written directly to the
ERRORLOG directory (always writable by SQL service account).

**This project phase reintroduces the need for `xp_cmdshell` only for
directory verification** (DIR check on the `AuditLog` folder). If the
folder is pre-created on every server during install, the DIR check
confirms its existence and accessibility without needing MKDIR.
Script must still:

- Query current `xp_cmdshell` and `show advanced options` state
- Enable both only if disabled
- Run verification
- **Always** restore original config via `finally` block
- Surface config restoration failures loudly — worst case outcome
  is leaving `xp_cmdshell` enabled on a production server
- Re-verify `xp_cmdshell` actually enabled (handles policy blocks that
  let `sp_configure` succeed silently while the feature stays off)

### Compliance audit baseline

26 action groups covering authentication, privilege changes, principal
lifecycle, DDL, audit self-protection, backup/restore, DBCC,
transport security, and user-defined events. See `sqlAuditTemplate`
in the v3 script for the exact list. Do not add or remove groups
without explicit user sign-off — these map to SOX/HIPAA/PCI
compliance checklists.

Audit creation parameters (overridable via script parameters):

- `MAXSIZE = 256 MB`
- `MAX_ROLLOVER_FILES = 50`
- `RESERVE_DISK_SPACE = OFF`
- `QUEUE_DELAY = 1000` (async, default)
- `ON_FAILURE = CONTINUE` (default; user may override with SHUTDOWN
  for regulated-data instances but should not be fleet-wide)
- `AUDIT_GUID = '<assigned-from-lookup>'`

### Output and logging

- **Transcript** in `.\Logs\` — full session output
- **Lookup CSV** in `.\Logs\` — full lookup table for audit trail and
  retry-list building
- **Failure log** in `C:\Temp\` — tab-delimited, one line per failure,
  persistent across runs; format
  `[timestamp]TAB<instance>TAB<stage>TAB<message>`
- **StrictMode 2.0** active (not Latest — too aggressive in catch paths)
- **`$ErrorActionPreference = 'Stop'`** globally
- All result row fields initialized in a `New-LookupRow` factory so
  StrictMode never flags undefined property access in catch/finally

### PowerShell idioms to follow

- `Invoke-Sqlcmd` via a thin `Invoke-Sql` wrapper that conditionally
  adds `-TrustServerCertificate` based on module capability detection
- `Get-SingleRow` helper normalizes `Invoke-Sqlcmd` return shapes
  (null / single object / array)
- `Expand-Template` helper replaces `{{TOKEN}}` placeholders in
  T-SQL templates
- `System.DBNull` guards on every string property from SQL query
  results (not `$null`)
- `@()` wrapping on count-producing pipelines
- `-LiteralPath` everywhere that touches the `SQLSERVER:\` provider
- `SupportsShouldProcess = $true` with `-WhatIf` support; pre-check
  read-only queries always execute so enumeration can be validated
  even in WhatIf mode

## Immediate next-step tasks for Copilot to help with

### Task 1: Implement enhanced path resolution

Modify the pre-check T-SQL to:

1. Query `sys.server_file_audits` for any existing audit’s path
1. Get `SERVERPROPERTY('ErrorLogFileName')` and extract its directory
1. Derive candidate `AuditLog` folder paths (sibling to `\MSSQL\LOG\`)
1. Return all candidates plus metadata indicating which is current,
   which is recommended, and which drive each is on
1. Add columns to the lookup table: `CurrentAuditPath`,
   `RecommendedAuditPath`, `PathSource`, `PathNeedsChange`

### Task 2: Implement existing-audit detection and cleanup

Before deploying the new audit, detect and drop:

1. The current server audit (any name — query
   `sys.server_audits`; the existing audit may not be named
   `Audit_Compliance`)
1. All dependent server audit specifications
1. Any database audit specifications that reference the audit being
   dropped (query `sys.database_audit_specifications` across all
   databases — requires dynamic SQL)
1. Log what was dropped into the failure log under stage
   `'PreExistingAuditRemoval'` (not actually a failure, but useful audit
   trail)

Important: the drop order is SPEC first, then AUDIT. A server audit
cannot be dropped while a specification references it.

### Task 3: Add `AuditLog` folder verification

With `xp_cmdshell` re-introduced for verification only:

1. Build the full resolved audit path per instance
1. Run `DIR "<path>"` via `xp_cmdshell`
1. Parse output for access-denied, path-not-found, or empty results
1. Mark the lookup row with `AuditFolderVerified = $true/$false`
1. Skip deployment for rows where verification fails; log to failure
   log under stage `'AuditFolderMissing'`

### Task 4: Tempdb drive detection

Determine which drive holds the tempdb files by querying
`sys.master_files WHERE database_id = 2`. Use this to:

1. Identify the tempdb drive letter
1. Validate that the resolved audit path is on that drive
1. Warn if the resolved path is on a different drive (suggests
   nonstandard layout for that instance)

### Task 5: Testing and validation

Build a test harness that:

1. Uses a small test CMS group (maybe 3–5 non-prod instances)
1. Captures the full lookup table CSV before and after deployment
1. Validates via comparison that:
- Every reachable instance has `AuditDeployed = true`
- AG replicas share matching GUIDs
- Audit paths resolved to tempdb drive
- Old audit objects are gone
1. Provides a rollback script that can restore the previous audit
   configuration from a pre-deployment snapshot

### Task 6: Collection-side job (separate script)

Once deployment is solid, build the matching collection script that:

1. Uses the same CMS path targeting and lookup table pattern
1. For each deployed audit, uses `sys.fn_get_audit_file()` with the
   resolved audit path to read records directly over the SQL
   connection (no file transfer)
1. Writes records to a central audit database on the CMS host
1. Tracks `event_time` watermark per instance to support incremental
   collection
1. Handles `audit_guid` continuity across AG failovers (same GUID
   means post-failover records are naturally merged)

## Things not to do

- **Do not parallelize the deployment loop.** Serial-only was a
  deliberate choice for failure containment and correlation.
- **Do not use `STRING_AGG`.** Needs to work on SQL 2016.
- **Do not hardcode paths.** Every path is resolved per-instance
  from SQL queries. No assumption that `D:\` or `E:\` is the
  tempdb drive.
- **Do not assume Windows Integrated Authentication will always work.**
  If a target needs SQL auth or a specific service account, the
  script should accept credentials via a `-Credential` parameter
  (future enhancement, not required yet).
- **Do not leave `xp_cmdshell` enabled.** Restoration via `finally`
  is non-negotiable. Even a crash of the script must not leave
  `xp_cmdshell` on.
- **Do not add action groups beyond the compliance baseline** without
  explicit sign-off. `SCHEMA_OBJECT_ACCESS_GROUP` at server scope
  on a busy instance can produce 50+ GB/day. Database audit specs
  scoped to specific tables are the correct pattern for data access
  auditing — but that’s a separate follow-up.
- **Do not deploy a database audit specification in this script.**
  Server-level audit + server audit spec only. Database specs are
  a follow-up because they need table-level targeting decisions per
  database.

## Copyright / creative work reminder

The compliance action group list is Microsoft-documented behavior,
not creative content — use it freely. The T-SQL patterns here
(`STUFF + FOR XML PATH`, `sp_configure` save/restore, etc.) are
standard idioms documented in official Microsoft and community
sources; attribute in comments only when pulling from a specific
blog or article.

## Tone / voice preferences for code comments

Dry, technically direct, opinionated where warranted. Comments should
explain *why* a choice was made, especially for non-obvious patterns
(StrictMode 2.0 vs Latest; `-LiteralPath` usage; DBNull handling).
Avoid motivational language or excessive caveats. If a known sharp
edge exists, call it out once, clearly, and move on.

## Reference materials

- v3 script: `Deploy-ComplianceAudit-v3.md` (in this workspace /
  project artifacts)
- SQL Server Audit action groups reference:
  `learn.microsoft.com/sql/relational-databases/security/auditing/sql-server-audit-action-groups-and-actions`
- AG DMVs reference:
  `learn.microsoft.com/sql/relational-databases/system-catalog-views/sys-availability-replicas-transact-sql`
- SqlServer PowerShell module:
  `learn.microsoft.com/sql/powershell/sql-server-powershell`

## How to engage Copilot with this prompt

**In chat**: Paste this whole file as the first message in a new
Copilot Chat session. Follow with the specific task you want to
work on (e.g., “Implement Task 2: existing-audit detection and
cleanup”). Copilot will have the constraints in context.

**Inline**: Keep this file open in a split editor alongside the
script you’re editing. Copilot’s inline suggestions will pull
context from visible files.

**For debugging**: When something fails, paste the relevant error
plus the stage it came from into chat. The failure log format is
tab-delimited so you can easily extract just the failing instance’s
line and its error for focused debugging.

**For code review**: Ask Copilot to review a change against the
“Things not to do” section and the architectural constraints in
this prompt. That’s often more useful than asking for a generic
review.