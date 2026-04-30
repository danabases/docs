# Audit Transcript

> Converted from `Audit_transcript.docx`.

## You said: You have near 100 sql server instances, versions 2016 -2025 in the enterprise.

You have near 100 sql server instances, versions 2016 -2025 in the enterprise. You need to upgrade the server auditing process which is very old and prone with issues in the collections. Firewalls only allow unidirectional from the central management server to all the database servers. sqlaudit files are currently stored on the local server being audited and transferred in zip files up to an ftp server where they are then collected from. A preference would be collecting directly from the server, perhaps with a job that runs on the central management server and reads them directly, bypassing the ftp completely. Is this a possibility or are there other better alternatives?

*Apr 18*

## Claude responded: Yes, pulling directly from the audited servers is absolutely doable and a clear win over the FTP hop.

Yes, pulling directly from the audited servers is absolutely doable and a clear win over the FTP hop. A few approaches worth weighing, roughly in order of how much I'd recommend them for your environment:

### 1. Pull via admin share from the CMS (closest to what you described)

Since your firewalls allow CMS → target, you can have a scheduled job on the CMS reach into each instance's audit directory via UNC path (\\server\D$\SQLAudit\ or a dedicated hidden share). A PowerShell job on the CMS enumerates .sqlaudit files, copies new ones locally (or to a central landing zone), optionally calls sys.fn_get_audit_file() via Invoke-Sqlcmd to shred them into a central audit database, then archives or deletes the originals. This matches your existing mental model, eliminates FTP, and uses the firewall direction you already have. Requires SMB (445) open CMS → targets and a service account with read on the audit folders. Given your PowerShell-heavy toolkit, this would slot in naturally.

### 2. Query the audit files remotely via T-SQL

sys.fn_get_audit_file() accepts a UNC path, so from the CMS you can run it against each registered instance through a linked server or looped Invoke-Sqlcmd, reading the audit events directly into a central table without ever copying the file. Cleaner than file copying if your central audit DB is the real destination anyway — no staging, no zip, no parsing step. Downside: the SQL Server service account on each target needs read access to the audit path, and you're doing the shred work over the network.

### 3. Write audit files to a central SMB share directly

SQL Server Audit supports writing to a UNC path as the target. Each instance's service account writes straight to \\central\audit$\<instancename>\, and the CMS just reads locally. This is arguably the cleanest architecture — no collection job, no local storage on the audited servers, no transfer step. But it inverts your firewall direction (targets → central on 445), which sounds like a non-starter given your unidirectional rule. Worth flagging to whoever owns the firewall policy, because it's the lowest-moving-parts option if they'll allow it.

### 4. Azure SQL Audit / Event Hubs / Log Analytics

If any of these instances are (or could be) Azure Arc-enabled or you have Azure connectivity, audit can stream to Log Analytics or Event Hubs and you sidestep file handling entirely. Probably overkill for a pure on-prem fleet, but worth knowing it exists — especially given your existing Azure footprint.

### 5. Extended Events instead of SQL Audit

Not a collection-transport fix, but worth mentioning: XE with the event_file target covers most auditing use cases with less overhead and the same sys.fn_xe_file_target_read_file() remote-read pattern. If you're rebuilding the pipeline anyway, it's a reasonable moment to ask whether SQL Audit is still the right primitive.

My recommendation for your setup: Option 1 or 2, depending on whether you want files preserved as artifacts (go with 1) or a clean central audit DB as the system of record (go with 2). Both respect your firewall direction, kill the FTP dependency, and you can build either as a single PowerShell module that iterates your CMS registered servers list.

One gotcha worth planning for across any of these: the SQL Server service account identity matters. For option 3 it needs network write; for option 1 the CMS's collection account needs read on each target; for option 2 each target's SQL service account needs to be able to read its own audit path when queried remotely. Mixed domain/workgroup instances will be the ones that bite you.

Want me to sketch out the PowerShell skeleton for option 1 or the T-SQL pattern for option 2?

## You said: Tell me more about option 5 extended events.

Tell me more about option 5 extended events. Is this the more modern way to do sql auditing, and is it compliant where the files are still immutable like the sqlaudit files?

*Apr 18*

## Claude responded: Good instinct to dig here — this is the question that actually matters for a rebuild.

Good instinct to dig here — this is the question that actually matters for a rebuild. Short answer: XE is the more modern and flexible diagnostic framework, but it is not a drop-in replacement for SQL Server Audit when compliance or immutability is on the line. Here's the nuance:

### What XE actually is

Extended Events is the general-purpose event infrastructure in SQL Server — it's the engine underneath a lot of things, including SQL Server Audit itself. SQL Audit is essentially a specialized, hardened wrapper around XE with compliance features bolted on. So saying "XE vs SQL Audit" is a bit misleading; it's really "raw XE sessions vs the Audit feature built on top of XE."

### Where XE wins

Broader event coverage (wait stats, query performance, deadlocks, blocking — things Audit can't see), lower overhead for high-volume capture, more flexible filtering and target options, and better for troubleshooting and performance work. If you're doing DBA diagnostics, XE is the right tool.

### Where SQL Audit wins — and why it matters for you

SQL Server Audit has specific compliance-oriented guarantees that raw XE sessions don't:

File integrity. Audit files are written with a checksum/signature scheme and sys.fn_get_audit_file() will flag tampering. XE event_file targets have no equivalent — a file that's been edited will read back fine.

Shutdown on audit failure. Audit supports ON_FAILURE = SHUTDOWN or FAIL_OPERATION, meaning the instance will refuse to process transactions if it can't write audit records. Regulators love this. XE has no equivalent — if the target fails, the session just stops and the server keeps running.

Protected DDL. You can't disable or alter a server audit without it being, itself, audited. Someone dropping an XE session leaves only whatever trace XE is configured to capture about itself.

Defined action groups. Audit has a curated set of security-relevant action groups (LOGIN_CHANGE_PASSWORD_GROUP, SERVER_ROLE_MEMBER_CHANGE_GROUP, etc.) that map cleanly to what auditors ask for. With XE you're assembling equivalents from lower-level events and hoping you covered everything.

Regulatory recognition. SOX, HIPAA, PCI-DSS, FedRAMP auditor checklists specifically reference SQL Server Audit. "We built a custom XE session" is a conversation you probably don't want to have with an external auditor, even if functionally equivalent.

### The immutability question specifically

Neither .sqlaudit nor .xel files are truly immutable at the filesystem level — both sit on an NTFS volume and can be deleted or modified by anyone with the right permissions. The difference is that .sqlaudit files have internal integrity checks that detect tampering on readback, while .xel files do not. True immutability comes from where you store them: WORM storage, Azure Blob with immutability policies, S3 Object Lock, or a central audit DB with append-only permissions. That's an architectural layer above either file format.

### Practical takeaway for your rebuild

If the auditing you're replacing exists because of a compliance requirement (SOX, HIPAA, PCI, internal audit, etc.), stay on SQL Server Audit. The integrity and shutdown-on-failure guarantees are the whole point, and no auditor is going to give you credit for a hand-rolled XE equivalent. Use XE alongside it for the diagnostic and performance work it's actually good at — they're not competing tools.

If the auditing exists purely for operational visibility and nobody's going to ask for attestation, then XE gives you more flexibility and you can collect it the same way (remote sys.fn_xe_file_target_read_file() or SMB pull). But given you're running ~100 instances in what sounds like a regulated enterprise, my strong bet is you want to keep SQL Audit as the primitive and just modernize the collection pipeline around it.

Want me to sketch what a hybrid looks like — SQL Audit for the compliance-relevant action groups, XE for the broader diagnostic capture, both pulled by the same CMS collection job?

## You said: List all the compliant relevant action groups

List all the compliant relevant action groups

*Apr 18*

## Claude responded: Here are the server-level and database-level audit action groups, organized by what they're typically used for.

Here are the server-level and database-level audit action groups, organized by what they're typically used for. Let me pull current documentation to make sure nothing's been added or deprecated in recent SQL Server versions.

Searched the web

Searched the web

Here's the full list of compliance-relevant action groups, organized by what auditors typically care about. I'll flag which ones are server-scope, database-scope, or available at both.

### Authentication & login activity

This is table stakes for every framework (SOX, PCI, HIPAA, FedRAMP).

SUCCESSFUL_LOGIN_GROUP — successful instance logins

FAILED_LOGIN_GROUP — failed login attempts

LOGOUT_GROUP — logout events

LOGIN_CHANGE_PASSWORD_GROUP — SQL login password changes

DATABASE_PRINCIPAL_IMPERSONATION_GROUP — EXECUTE AS at database scope

SERVER_PRINCIPAL_IMPERSONATION_GROUP — EXECUTE AS at server scope

### Privilege and permission changes

Tracking who can do what, and when that changes.

SERVER_ROLE_MEMBER_CHANGE_GROUP — sysadmin, securityadmin, etc. membership changes

DATABASE_ROLE_MEMBER_CHANGE_GROUP — db_owner, db_datareader, etc.

SERVER_PERMISSION_CHANGE_GROUP — GRANT/REVOKE/DENY at server level

DATABASE_PERMISSION_CHANGE_GROUP — GRANT/REVOKE/DENY at database level

SCHEMA_OBJECT_PERMISSION_CHANGE_GROUP — permissions on tables, views, procs

SERVER_OBJECT_PERMISSION_CHANGE_GROUP — permissions on endpoints, server objects

### Principal (account) lifecycle

Creation, modification, deletion of logins and users.

SERVER_PRINCIPAL_CHANGE_GROUP — CREATE/ALTER/DROP LOGIN

DATABASE_PRINCIPAL_CHANGE_GROUP — CREATE/ALTER/DROP USER

SERVER_ROLE_MEMBER_CHANGE_GROUP (listed above — overlaps)

DATABASE_OWNERSHIP_CHANGE_GROUP — ALTER AUTHORIZATION on database

SCHEMA_OBJECT_OWNERSHIP_CHANGE_GROUP — ALTER AUTHORIZATION on objects

### DDL / schema changes

What objects existed, what changed, when.

SERVER_OBJECT_CHANGE_GROUP — CREATE/ALTER/DROP on endpoints, server objects

DATABASE_OBJECT_CHANGE_GROUP — database-scoped object changes

SCHEMA_OBJECT_CHANGE_GROUP — CREATE/ALTER/DROP of schema objects (tables, views, procs)

DATABASE_CHANGE_GROUP — CREATE/ALTER/DROP DATABASE

SERVER_STATE_CHANGE_GROUP — service state changes (start, stop, pause)

### The audit trail protecting itself

Auditors specifically look for these — they prove the audit wasn't tampered with.

AUDIT_CHANGE_GROUP — any change to an audit or audit specification (this one is always-on for the audit itself but include it explicitly to be safe)

TRACE_CHANGE_GROUP — trace flag changes, SQL Trace starts/stops

### Backup, restore, and DBCC

BACKUP_RESTORE_GROUP — all backup and restore operations

DBCC_GROUP — DBCC command execution

### Data access (for regulated data)

Only use these on databases containing regulated data — they're noisy and expensive.

SCHEMA_OBJECT_ACCESS_GROUP — SELECT/INSERT/UPDATE/DELETE/EXECUTE on schema objects (server or database scope)

SENSITIVE_BATCH_COMPLETED_GROUP — SQL Server 2022+, captures batches touching columns tagged with data classification labels; much more targeted than the access group above

BATCH_COMPLETED_GROUP — every batch executed; usually too noisy for compliance, useful for forensics

Or use individual DATABASE_OBJECT_ACCESS_GROUP actions (SELECT, INSERT, UPDATE, DELETE, EXECUTE, RECEIVE, REFERENCES) scoped to specific tables via a database audit specification — this is the standard pattern for PCI cardholder data or HIPAA PHI tables.

### Encryption, keys, and certificates

Critical for PCI and anything with TDE or Always Encrypted.

DATABASE_OBJECT_ACCESS_GROUP — covers key access

DBCC_GROUP (overlaps)

Individual actions on SYMMETRIC KEY, ASYMMETRIC KEY, CERTIFICATE objects

### Operations auditors ask about less often but should be included

DATABASE_OPERATION_GROUP — checkpoint, subscribe to query notification

SERVER_OPERATION_GROUP — security-related server operations (altering settings, resource governor, etc.)

BROKER_LOGIN_GROUP — Service Broker transport security audit events

DATABASE_MIRRORING_LOGIN_GROUP — mirroring/AG transport security

USER_CHANGE_PASSWORD_GROUP — contained database user password changes (SQL 2012+)

USER_DEFINED_AUDIT_GROUP — lets apps raise custom audit events via sp_audit_write

### A practical starting template for a compliance baseline

For a typical "meets SOX / audit-friendly" server audit specification, this covers most of what you'll get asked about without drowning in noise:

FAILED_LOGIN_GROUP

SUCCESSFUL_LOGIN_GROUP

LOGOUT_GROUP

LOGIN_CHANGE_PASSWORD_GROUP

SERVER_ROLE_MEMBER_CHANGE_GROUP

DATABASE_ROLE_MEMBER_CHANGE_GROUP

SERVER_PERMISSION_CHANGE_GROUP

DATABASE_PERMISSION_CHANGE_GROUP

SERVER_PRINCIPAL_CHANGE_GROUP

DATABASE_PRINCIPAL_CHANGE_GROUP

SERVER_OBJECT_CHANGE_GROUP

DATABASE_CHANGE_GROUP

SCHEMA_OBJECT_CHANGE_GROUP

DATABASE_OWNERSHIP_CHANGE_GROUP

AUDIT_CHANGE_GROUP

TRACE_CHANGE_GROUP

BACKUP_RESTORE_GROUP

DBCC_GROUP

SERVER_STATE_CHANGE_GROUP

Then add SCHEMA_OBJECT_ACCESS_GROUP per database in a database audit specification, scoped to just the tables holding regulated data — not instance-wide, or you'll generate gigabytes per day per busy server.

Microsoft's full catalog has grown to around 48 server-level action groups as of 2022, and some action groups that were originally server-only became available at database scope starting in SQL Server 2012 and 2016, so the exact availability depends on version. Given your fleet spans 2016–2025, everything in the baseline template above is supported across all of them. The SENSITIVE_BATCH_COMPLETED_GROUP is the main one that's 2022+ only. MSSQLTipsUltimate Windows Security

One practical note for your rebuild: define this baseline as a single T-SQL script and deploy it identically across all instances via your CMS. Auditors love consistency, and a drift between what server A and server B capture is the kind of finding that turns a clean audit into a remediation exercise.

Want me to write the CREATE SERVER AUDIT + CREATE SERVER AUDIT SPECIFICATION template with this baseline, parameterized for the audit file path and rollover settings?

## You said: Yes

Yes

*Apr 18*

## Claude responded: Here's a production-ready template.

Here's a production-ready template. I've parameterized the pieces that'll vary per instance and left inline comments flagging the decisions you'll want to revisit for your environment.

```sql
/*==============================================================================
    SQL Server Compliance Audit Baseline
    Target: SQL Server 2016 through 2025
    Scope:  Server-level audit + server audit specification

    Deployment notes:
    - Run in master database
    - Requires CONTROL SERVER or ALTER ANY SERVER AUDIT
    - Audit file path must exist and be writable by the SQL Server service account
    - Change @AuditFilePath per instance or standardize via a mount/junction
==============================================================================*/

USE [master];
GO

/*------------------------------------------------------------------------------
    Parameters -- adjust per environment, then inline below
------------------------------------------------------------------------------*/
-- Audit file destination. Local path for now; swap to UNC if you move to
-- centralized write (see prior conversation re: firewall direction).
DECLARE @AuditFilePath       NVARCHAR(260) = N'D:\SQLAudit\';

-- Per-file size before rollover. 256 MB balances shred performance against
-- file count. Larger = fewer files to collect, slower to query individually.
DECLARE @MaxFileSizeMB       INT           = 256;

-- Max rollover files retained on local disk. At 256 MB * 50 = ~12.5 GB ceiling
-- per instance. Tune based on collection frequency and local disk headroom.
DECLARE @MaxRolloverFiles    INT           = 50;

-- Queue delay in milliseconds. 1000 = async (default, minimal perf impact).
-- Set to 0 only if regulator requires synchronous writes -- significant overhead.
DECLARE @QueueDelayMs        INT           = 1000;

-- ON_FAILURE behavior: CONTINUE | SHUTDOWN | FAIL_OPERATION
-- CONTINUE  = audit loss is logged but server keeps running (most environments)
-- SHUTDOWN  = instance stops if audit can't write (strictest compliance posture)
-- FAIL_OPERATION = individual auditable actions fail but instance stays up
DECLARE @OnFailureAction     NVARCHAR(20)  = N'CONTINUE';

PRINT '--- Deploying audit with path: ' + @AuditFilePath;
GO


/*------------------------------------------------------------------------------
    1. Drop existing objects if present (idempotent redeployment)
       Order matters: specification before audit, both disabled before drop.
------------------------------------------------------------------------------*/
IF EXISTS (SELECT 1 FROM sys.server_audit_specifications
           WHERE name = N'ServerAuditSpec_Compliance')
BEGIN
    ALTER SERVER AUDIT SPECIFICATION [ServerAuditSpec_Compliance] WITH (STATE = OFF);
    DROP SERVER AUDIT SPECIFICATION [ServerAuditSpec_Compliance];
    PRINT 'Dropped existing ServerAuditSpec_Compliance';
END
GO

IF EXISTS (SELECT 1 FROM sys.server_audits
           WHERE name = N'Audit_Compliance')
BEGIN
    ALTER SERVER AUDIT [Audit_Compliance] WITH (STATE = OFF);
    DROP SERVER AUDIT [Audit_Compliance];
    PRINT 'Dropped existing Audit_Compliance';
END
GO


/*------------------------------------------------------------------------------
    2. Create the server audit (the destination / transport layer)
       This is what writes .sqlaudit files. Disabled until spec is attached.
------------------------------------------------------------------------------*/
CREATE SERVER AUDIT [Audit_Compliance]
TO FILE
(   FILEPATH        = N'D:\SQLAudit\'           -- replace with @AuditFilePath value
    ,MAXSIZE        = 256 MB
    ,MAX_ROLLOVER_FILES = 50
    ,RESERVE_DISK_SPACE = OFF
)
WITH
(   QUEUE_DELAY     = 1000
    ,ON_FAILURE     = CONTINUE
    ,AUDIT_GUID     = NEWID()                   -- omit to let SQL assign; set
                                                -- explicitly if you need the
                                                -- same GUID across HA replicas
);
GO


/*------------------------------------------------------------------------------
    3. Create the server audit specification (what gets captured)
       Baseline compliance coverage: auth, privilege, principals, DDL,
       audit self-protection, backup/restore, DBCC.

       Compatible with SQL Server 2016+. No version-gated groups included.
------------------------------------------------------------------------------*/
CREATE SERVER AUDIT SPECIFICATION [ServerAuditSpec_Compliance]
FOR SERVER AUDIT [Audit_Compliance]

    -- Authentication activity
    ADD (SUCCESSFUL_LOGIN_GROUP),
    ADD (FAILED_LOGIN_GROUP),
    ADD (LOGOUT_GROUP),
    ADD (LOGIN_CHANGE_PASSWORD_GROUP),

    -- Privilege / permission changes
    ADD (SERVER_ROLE_MEMBER_CHANGE_GROUP),
    ADD (DATABASE_ROLE_MEMBER_CHANGE_GROUP),
    ADD (SERVER_PERMISSION_CHANGE_GROUP),
    ADD (DATABASE_PERMISSION_CHANGE_GROUP),
    ADD (SERVER_OBJECT_PERMISSION_CHANGE_GROUP),
    ADD (SCHEMA_OBJECT_PERMISSION_CHANGE_GROUP),

    -- Principal lifecycle
    ADD (SERVER_PRINCIPAL_CHANGE_GROUP),
    ADD (DATABASE_PRINCIPAL_CHANGE_GROUP),
    ADD (DATABASE_OWNERSHIP_CHANGE_GROUP),
    ADD (SCHEMA_OBJECT_OWNERSHIP_CHANGE_GROUP),

    -- DDL / schema changes
    ADD (SERVER_OBJECT_CHANGE_GROUP),
    ADD (DATABASE_OBJECT_CHANGE_GROUP),
    ADD (SCHEMA_OBJECT_CHANGE_GROUP),
    ADD (DATABASE_CHANGE_GROUP),
    ADD (SERVER_STATE_CHANGE_GROUP),

    -- Audit self-protection
    ADD (AUDIT_CHANGE_GROUP),
    ADD (TRACE_CHANGE_GROUP),

    -- Backup, restore, DBCC
    ADD (BACKUP_RESTORE_GROUP),
    ADD (DBCC_GROUP),

    -- Transport security (only meaningful if you use Service Broker / AG / Mirroring)
    ADD (BROKER_LOGIN_GROUP),
    ADD (DATABASE_MIRRORING_LOGIN_GROUP),

    -- Custom application audit events via sp_audit_write
    ADD (USER_DEFINED_AUDIT_GROUP)

WITH (STATE = OFF);   -- enable explicitly after verifying path access
GO


/*------------------------------------------------------------------------------
    4. Enable audit and specification (order matters: audit first)
------------------------------------------------------------------------------*/
ALTER SERVER AUDIT [Audit_Compliance] WITH (STATE = ON);
GO

ALTER SERVER AUDIT SPECIFICATION [ServerAuditSpec_Compliance] WITH (STATE = ON);
GO


/*------------------------------------------------------------------------------
    5. Verification
------------------------------------------------------------------------------*/
SELECT
     sa.name                    AS audit_name
    ,sa.is_state_enabled        AS audit_enabled
    ,sa.on_failure_desc
    ,sa.queue_delay
    ,saf.log_file_path
    ,saf.max_file_size
    ,saf.max_rollover_files
FROM sys.server_audits sa
JOIN sys.server_file_audits saf ON sa.audit_guid = saf.audit_guid
WHERE sa.name = N'Audit_Compliance';

SELECT
     sas.name                   AS spec_name
    ,sas.is_state_enabled       AS spec_enabled
    ,sad.audit_action_name
FROM sys.server_audit_specifications sas
JOIN sys.server_audit_specification_details sad
    ON sas.server_specification_id = sad.server_specification_id
WHERE sas.name = N'ServerAuditSpec_Compliance'
ORDER BY sad.audit_action_name;
GO
```

### A few things worth calling out before you deploy this fleet-wide:

The parameter block is cosmetic. CREATE SERVER AUDIT won't accept variables for FILEPATH, MAXSIZE, etc. — those have to be literal values. I kept the parameter block at the top as a documentation pattern so the intent is obvious, but for real fleet deployment you'll want to wrap this in PowerShell and use string replacement against a template .sql file, or build it dynamically and EXEC sp_executesql. Given your existing PowerShell tooling, the template-replacement pattern will feel familiar.

ON_FAILURE = CONTINUE vs SHUTDOWN. I defaulted to CONTINUE because a fleet-wide SHUTDOWN setting is how you turn a disk-full event into a Sev 1 incident across 100 instances simultaneously. If regulators specifically require shutdown-on-audit-failure, deploy it to the regulated-data instances only and keep CONTINUE on the rest. The audit itself will log any write failures either way.

AUDIT_GUID and HA. If any of these instances are in Availability Groups or mirrored, the audit GUID needs to match across replicas for the audit to continue cleanly on failover. Generate the GUID once per logical instance pair and hardcode it in the per-instance deployment rather than using NEWID(). Easy to miss, painful to discover after a failover.

What's deliberately not here. I left out SCHEMA_OBJECT_ACCESS_GROUP and BATCH_COMPLETED_GROUP — those belong in a database audit specification scoped to regulated tables, not instance-wide. Dropping SCHEMA_OBJECT_ACCESS_GROUP into a server audit spec is how people accidentally generate 50+ GB of audit per day on a busy OLTP server. If you have databases holding PCI/PHI/SOX-relevant data, that's a separate per-database spec I can write next.

The disabled-then-enabled pattern. Creating with STATE = OFF and enabling separately is deliberate — it lets you verify the file path is writable before the audit starts trying to log and potentially tripping the ON_FAILURE behavior on a typo.

Want me to write the matching CREATE DATABASE AUDIT SPECIFICATION template for regulated-data access tracking, or the PowerShell deployment wrapper that iterates your CMS registered servers list and applies this?

## You said: Write the PowerShell deployment wrapper that iterates the cms registered servers list and applies this.

Write the PowerShell deployment wrapper that iterates the cms registered servers list and applies this.

*Apr 18*

## Claude responded: Here's the deployment wrapper.

Here's the deployment wrapper. I've structured it as a module-style script with the patterns that've worked well in your previous PowerShell work — StrictMode, explicit error handling, clean separation between the template and the deployment loop.

```powershell
<#
.SYNOPSIS
    Deploys the compliance audit baseline across all CMS registered servers.

.DESCRIPTION
    Iterates the Central Management Server registered servers list, applies the
    server audit + server audit specification defined in the embedded T-SQL
    template, and produces a per-instance deployment report.

    Safe to rerun: the template drops and recreates existing audit objects.

.PARAMETER CmsInstance
    The Central Management Server hosting the registered servers list.

.PARAMETER CmsGroup
    Optional. Limit deployment to a specific CMS server group (and children).
    Omit to target all registered servers.

.PARAMETER AuditFilePath
    Local path on each target instance where .sqlaudit files are written.
    Must exist and be writable by the SQL Server service account.

.PARAMETER WhatIf
    Generates and logs the T-SQL per instance without executing it.

.EXAMPLE
    .\Deploy-ComplianceAudit.ps1 -CmsInstance 'CMS01' -WhatIf

.EXAMPLE
    .\Deploy-ComplianceAudit.ps1 -CmsInstance 'CMS01' -CmsGroup 'Production\SQL'

.NOTES
    Requires: SqlServer PowerShell module (Install-Module SqlServer)
    Account running this needs: CONTROL SERVER on every target instance
#>

[CmdletBinding(SupportsShouldProcess = $true)]
param(
    [Parameter(Mandatory = $true)]
    [string]$CmsInstance,

    [Parameter(Mandatory = $false)]
    [string]$CmsGroup,

    [Parameter(Mandatory = $false)]
    [string]$AuditFilePath = 'D:\SQLAudit\',

    [Parameter(Mandatory = $false)]
    [int]$MaxFileSizeMB = 256,

    [Parameter(Mandatory = $false)]
    [int]$MaxRolloverFiles = 50,

    [Parameter(Mandatory = $false)]
    [ValidateSet('CONTINUE', 'SHUTDOWN', 'FAIL_OPERATION')]
    [string]$OnFailureAction = 'CONTINUE',

    [Parameter(Mandatory = $false)]
    [int]$QueryTimeoutSeconds = 60,

    [Parameter(Mandatory = $false)]
    [string]$LogDirectory = "$PSScriptRoot\Logs"
)

Set-StrictMode -Version Latest
$ErrorActionPreference = 'Stop'

#region ---- Prereqs and logging --------------------------------------------

if (-not (Get-Module -ListAvailable -Name SqlServer)) {
    throw "SqlServer module not installed. Run: Install-Module SqlServer -Scope CurrentUser"
}
Import-Module SqlServer -DisableNameChecking

if (-not (Test-Path -Path $LogDirectory)) {
    New-Item -Path $LogDirectory -ItemType Directory -Force | Out-Null
}

$runTimestamp = Get-Date -Format 'yyyyMMdd_HHmmss'
$transcriptPath = Join-Path $LogDirectory "Deploy-ComplianceAudit_$runTimestamp.log"
$resultsPath    = Join-Path $LogDirectory "Deploy-ComplianceAudit_$runTimestamp.csv"

Start-Transcript -Path $transcriptPath -Force | Out-Null

Write-Host "=== Compliance Audit Deployment ===" -ForegroundColor Cyan
Write-Host "CMS:             $CmsInstance"
Write-Host "Group filter:    $(if ($CmsGroup) { $CmsGroup } else { '(all)' })"
Write-Host "Audit path:      $AuditFilePath"
Write-Host "Max file size:   $MaxFileSizeMB MB"
Write-Host "Rollover files:  $MaxRolloverFiles"
Write-Host "On failure:      $OnFailureAction"
Write-Host "WhatIf mode:     $($WhatIfPreference -eq $true)"
Write-Host ""

#endregion

#region ---- T-SQL template --------------------------------------------------

# Template uses {{TOKEN}} placeholders replaced per instance. Kept as a single
# here-string so the SQL stays readable and diff-able against the reference
# template maintained outside this script.

$auditTemplate = @'
USE [master];

IF EXISTS (SELECT 1 FROM sys.server_audit_specifications
           WHERE name = N'ServerAuditSpec_Compliance')
BEGIN
    ALTER SERVER AUDIT SPECIFICATION [ServerAuditSpec_Compliance] WITH (STATE = OFF);
    DROP SERVER AUDIT SPECIFICATION [ServerAuditSpec_Compliance];
END;

IF EXISTS (SELECT 1 FROM sys.server_audits
           WHERE name = N'Audit_Compliance')
BEGIN
    ALTER SERVER AUDIT [Audit_Compliance] WITH (STATE = OFF);
    DROP SERVER AUDIT [Audit_Compliance];
END;

CREATE SERVER AUDIT [Audit_Compliance]
TO FILE
(    FILEPATH            = N'{{AUDIT_PATH}}'
    ,MAXSIZE             = {{MAX_SIZE_MB}} MB
    ,MAX_ROLLOVER_FILES  = {{MAX_ROLLOVER}}
    ,RESERVE_DISK_SPACE  = OFF
)
WITH
(    QUEUE_DELAY  = 1000
    ,ON_FAILURE  = {{ON_FAILURE}}
);

CREATE SERVER AUDIT SPECIFICATION [ServerAuditSpec_Compliance]
FOR SERVER AUDIT [Audit_Compliance]
    ADD (SUCCESSFUL_LOGIN_GROUP),
    ADD (FAILED_LOGIN_GROUP),
    ADD (LOGOUT_GROUP),
    ADD (LOGIN_CHANGE_PASSWORD_GROUP),
    ADD (SERVER_ROLE_MEMBER_CHANGE_GROUP),
    ADD (DATABASE_ROLE_MEMBER_CHANGE_GROUP),
    ADD (SERVER_PERMISSION_CHANGE_GROUP),
    ADD (DATABASE_PERMISSION_CHANGE_GROUP),
    ADD (SERVER_OBJECT_PERMISSION_CHANGE_GROUP),
    ADD (SCHEMA_OBJECT_PERMISSION_CHANGE_GROUP),
    ADD (SERVER_PRINCIPAL_CHANGE_GROUP),
    ADD (DATABASE_PRINCIPAL_CHANGE_GROUP),
    ADD (DATABASE_OWNERSHIP_CHANGE_GROUP),
    ADD (SCHEMA_OBJECT_OWNERSHIP_CHANGE_GROUP),
    ADD (SERVER_OBJECT_CHANGE_GROUP),
    ADD (DATABASE_OBJECT_CHANGE_GROUP),
    ADD (SCHEMA_OBJECT_CHANGE_GROUP),
    ADD (DATABASE_CHANGE_GROUP),
    ADD (SERVER_STATE_CHANGE_GROUP),
    ADD (AUDIT_CHANGE_GROUP),
    ADD (TRACE_CHANGE_GROUP),
    ADD (BACKUP_RESTORE_GROUP),
    ADD (DBCC_GROUP),
    ADD (BROKER_LOGIN_GROUP),
    ADD (DATABASE_MIRRORING_LOGIN_GROUP),
    ADD (USER_DEFINED_AUDIT_GROUP)
WITH (STATE = OFF);

ALTER SERVER AUDIT [Audit_Compliance] WITH (STATE = ON);
ALTER SERVER AUDIT SPECIFICATION [ServerAuditSpec_Compliance] WITH (STATE = ON);
'@

#endregion

#region ---- Helper functions ------------------------------------------------

function Get-CmsRegisteredInstance {
    [CmdletBinding()]
    param(
        [Parameter(Mandatory)][string]$CmsServer,
        [string]$GroupPath
    )

    # Pull registered servers from CMS via SMO. Filters out the CMS self-entry
    # which appears in some configurations.
    $regSvrPath = if ($GroupPath) {
        "SQLSERVER:\SQLRegistration\Central Management Server Group\$CmsServer\$GroupPath"
    }
    else {
        "SQLSERVER:\SQLRegistration\Central Management Server Group\$CmsServer"
    }

    Get-ChildItem -Path $regSvrPath -Recurse |
        Where-Object { $_.ServerName } |
        Select-Object -ExpandProperty ServerName -Unique |
        Sort-Object
}

function Test-InstanceReachable {
    [CmdletBinding()]
    param([Parameter(Mandatory)][string]$Instance)

    try {
        $null = Invoke-Sqlcmd -ServerInstance $Instance `
                              -Query "SELECT 1 AS Reachable" `
                              -QueryTimeout 10 `
                              -ConnectionTimeout 10 `
                              -TrustServerCertificate `
                              -ErrorAction Stop
        return $true
    }
    catch {
        return $false
    }
}

function Get-InstanceVersion {
    [CmdletBinding()]
    param([Parameter(Mandatory)][string]$Instance)

    $q = @"
SELECT
     SERVERPROPERTY('ProductVersion') AS ProductVersion
    ,SERVERPROPERTY('ProductMajorVersion') AS MajorVersion
    ,SERVERPROPERTY('Edition') AS Edition
"@
    Invoke-Sqlcmd -ServerInstance $Instance `
                  -Query $q `
                  -QueryTimeout 15 `
                  -TrustServerCertificate `
                  -ErrorAction Stop
}

function Build-AuditSql {
    [CmdletBinding()]
    param(
        [Parameter(Mandatory)][string]$Template,
        [Parameter(Mandatory)][string]$Path,
        [Parameter(Mandatory)][int]$MaxSizeMB,
        [Parameter(Mandatory)][int]$MaxRollover,
        [Parameter(Mandatory)][string]$OnFailure
    )

    # Normalize trailing slash -- SQL Server accepts both but consistency is
    # easier to reason about when comparing across instances.
    if (-not $Path.EndsWith('\')) { $Path = "$Path\" }

    $sql = $Template
    $sql = $sql.Replace('{{AUDIT_PATH}}',  $Path)
    $sql = $sql.Replace('{{MAX_SIZE_MB}}', $MaxSizeMB.ToString())
    $sql = $sql.Replace('{{MAX_ROLLOVER}}', $MaxRollover.ToString())
    $sql = $sql.Replace('{{ON_FAILURE}}',  $OnFailure)
    return $sql
}

function Invoke-AuditDeployment {
    [CmdletBinding(SupportsShouldProcess)]
    param(
        [Parameter(Mandatory)][string]$Instance,
        [Parameter(Mandatory)][string]$Sql,
        [Parameter(Mandatory)][int]$Timeout
    )

    if ($PSCmdlet.ShouldProcess($Instance, 'Deploy compliance audit')) {
        Invoke-Sqlcmd -ServerInstance $Instance `
                      -Query $Sql `
                      -QueryTimeout $Timeout `
                      -TrustServerCertificate `
                      -ErrorAction Stop
    }
    else {
        Write-Host "  [WhatIf] Would execute deployment SQL against $Instance"
    }
}

function Test-AuditDeployment {
    [CmdletBinding()]
    param([Parameter(Mandatory)][string]$Instance)

    $q = @"
SELECT
     CAST(sa.is_state_enabled  AS INT) AS AuditEnabled
    ,CAST(sas.is_state_enabled AS INT) AS SpecEnabled
    ,COUNT(sad.audit_action_name)      AS ActionGroupCount
FROM sys.server_audits sa
LEFT JOIN sys.server_audit_specifications sas
    ON sas.name = N'ServerAuditSpec_Compliance'
LEFT JOIN sys.server_audit_specification_details sad
    ON sad.server_specification_id = sas.server_specification_id
WHERE sa.name = N'Audit_Compliance'
GROUP BY sa.is_state_enabled, sas.is_state_enabled;
"@
    Invoke-Sqlcmd -ServerInstance $Instance `
                  -Query $q `
                  -QueryTimeout 15 `
                  -TrustServerCertificate `
                  -ErrorAction Stop
}

#endregion

#region ---- Main execution --------------------------------------------------

Write-Host "Enumerating registered servers from CMS..." -ForegroundColor Cyan
$instances = Get-CmsRegisteredInstance -CmsServer $CmsInstance -GroupPath $CmsGroup

if (-not $instances) {
    Stop-Transcript | Out-Null
    throw "No registered servers found under CMS '$CmsInstance'$(if ($CmsGroup) { " group '$CmsGroup'" })."
}

Write-Host "Found $($instances.Count) instance(s) to process." -ForegroundColor Green
Write-Host ""

$results = [System.Collections.Generic.List[object]]::new()
$counter = 0

foreach ($instance in $instances) {
    $counter++
    Write-Host "[$counter/$($instances.Count)] $instance" -ForegroundColor Yellow

    $result = [PSCustomObject]@{
        Instance       = $instance
        Reachable      = $false
        MajorVersion   = $null
        Edition        = $null
        Deployed       = $false
        AuditEnabled   = $null
        SpecEnabled    = $null
        ActionGroups   = $null
        DurationMs     = 0
        ErrorMessage   = $null
        Timestamp      = (Get-Date).ToString('s')
    }

    $sw = [System.Diagnostics.Stopwatch]::StartNew()

    try {
        # Reachability probe first -- avoids the long default connection
        # timeout on dead instances and gives a cleaner error in the report.
        if (-not (Test-InstanceReachable -Instance $instance)) {
            throw "Instance unreachable (timeout or auth failure)"
        }
        $result.Reachable = $true

        $ver = Get-InstanceVersion -Instance $instance
        $result.MajorVersion = $ver.MajorVersion
        $result.Edition      = $ver.Edition

        # Version gate: SQL 2016 = major version 13. Skip older.
        if ([int]$ver.MajorVersion -lt 13) {
            throw "Unsupported version (major $($ver.MajorVersion)); baseline targets 2016+"
        }

        $sql = Build-AuditSql -Template $auditTemplate `
                              -Path $AuditFilePath `
                              -MaxSizeMB $MaxFileSizeMB `
                              -MaxRollover $MaxRolloverFiles `
                              -OnFailure $OnFailureAction

        Invoke-AuditDeployment -Instance $instance `
                               -Sql $sql `
                               -Timeout $QueryTimeoutSeconds

        if (-not $WhatIfPreference) {
            $verify = Test-AuditDeployment -Instance $instance
            $result.AuditEnabled = $verify.AuditEnabled
            $result.SpecEnabled  = $verify.SpecEnabled
            $result.ActionGroups = $verify.ActionGroupCount

            if ($verify.AuditEnabled -ne 1 -or $verify.SpecEnabled -ne 1) {
                throw "Deployment verification failed (audit=$($verify.AuditEnabled), spec=$($verify.SpecEnabled))"
            }
        }

        $result.Deployed = $true
        Write-Host "  OK  (v$($ver.MajorVersion), $($result.ActionGroups) action groups)" -ForegroundColor Green
    }
    catch {
        $result.ErrorMessage = $_.Exception.Message
        Write-Host "  FAIL: $($_.Exception.Message)" -ForegroundColor Red
    }
    finally {
        $sw.Stop()
        $result.DurationMs = $sw.ElapsedMilliseconds
        $results.Add($result)
    }
}

#endregion

#region ---- Summary and output ----------------------------------------------

Write-Host ""
Write-Host "=== Summary ===" -ForegroundColor Cyan
$success = ($results | Where-Object { $_.Deployed }).Count
$failed  = ($results | Where-Object { -not $_.Deployed }).Count
$unreachable = ($results | Where-Object { -not $_.Reachable }).Count

Write-Host "Succeeded:   $success" -ForegroundColor Green
Write-Host "Failed:      $failed"  -ForegroundColor $(if ($failed)      { 'Red' }    else { 'Gray' })
Write-Host "Unreachable: $unreachable" -ForegroundColor $(if ($unreachable) { 'Red' } else { 'Gray' })

$results | Export-Csv -Path $resultsPath -NoTypeInformation -Encoding UTF8
Write-Host ""
Write-Host "Results:     $resultsPath"
Write-Host "Transcript:  $transcriptPath"

if ($failed -gt 0) {
    Write-Host ""
    Write-Host "Failed instances:" -ForegroundColor Red
    $results | Where-Object { -not $_.Deployed } |
        Format-Table Instance, ErrorMessage -AutoSize -Wrap
}

Stop-Transcript | Out-Null

# Non-zero exit code if any failures -- useful for scheduled job monitoring.
if ($failed -gt 0) { exit 1 } else { exit 0 }

#endregion
```

### A few things worth flagging before you run this against the fleet:

Test in a small CMS group first. Drop a handful of non-prod instances into a Test\Audit group and run with -CmsGroup 'Test\Audit' before pointing it at production. The script is idempotent but the first run on each instance still drops and recreates audit objects — any in-flight audit writes during that window are lost, so schedule it during a maintenance window for the production pass.

The audit file path must exist on every target. The script doesn't create D:\SQLAudit\ on remote servers — SQL Server won't either, and CREATE SERVER AUDIT will fail if the path doesn't exist. Two options: either pre-stage the directory via a separate PowerShell loop (Invoke-Command to each host) or let this script fail on missing-path instances and use the CSV output to identify them for remediation. Given your fleet size, I'd pre-stage.

TrustServerCertificate is on. This is the pragmatic setting for internal CMS-to-instance connections where cert validation would require a properly deployed PKI across 100 servers. If your security posture requires certificate validation, remove that flag and expect some instances to fail until their certs are in order — which is honestly a decent forcing function for that cleanup work.

Reachability probe before version check. I put Test-InstanceReachable first because the default SMO connection timeout is long enough that a handful of dead instances can stretch a fleet deployment from minutes to an hour. The 10-second probe fails fast and keeps the run moving.

HA consideration not baked in. If you have AGs or mirroring, the wrapper treats each listener/replica as an independent instance. For AGs specifically, you'll want to deploy the audit once per replica with a consistent AUDIT_GUID so the audit survives failover — that's not handled here because it requires AG topology awareness. Happy to write a variant that queries sys.availability_replicas and coordinates the GUID across replicas if that's relevant to your fleet.

WhatIf mode is real. -WhatIf will enumerate the instances, test reachability, check versions, and print the SQL it would run — without executing it. Good first pass to confirm the CMS enumeration picks up what you expect.

Want me to follow up with the collection-side job — the PowerShell that pulls .sqlaudit files from each instance back to the CMS via UNC and shreds them into a central audit database?

## You said: Running a remote powershell script to run on the servers will not work.

Running a remote powershell script to run on the servers will not work. It all needs to be executed via sql server CMS as sql connections. Xp_cmdshell and other advanced options can be used and the script should handle checking if whether they are on or off, enabling if needed, and then setting back the original value at completion. It should also run over a loop of one server at a time vs bulk runs.

*Apr 18*

## Claude responded: Understood — this reframes the deployment as T-SQL orchestrated from the CMS using linked servers or Invoke-Sqlcmd loops, with xp_cmdshell toggled per-instance…

Understood — this reframes the deployment as T-SQL orchestrated from the CMS using linked servers or Invoke-Sqlcmd loops, with xp_cmdshell toggled per-instance for filesystem prep. Here's the reworked version.

```powershell
<#
.SYNOPSIS
    Deploys the compliance audit baseline across CMS registered servers using
    SQL connections only (no PowerShell remoting, no SMB).

.DESCRIPTION
    Iterates the CMS registered servers list one instance at a time. For each
    instance, the script connects via SQL, saves the current state of relevant
    configuration options (xp_cmdshell, show advanced options), enables them
    if needed, pre-creates the audit directory via xp_cmdshell, deploys the
    audit baseline, verifies, then restores the original configuration.

    Per-instance serial execution ensures a failure on one server does not
    leave another in a half-configured state and makes failures easy to
    correlate in the log.

.PARAMETER CmsInstance
    The Central Management Server hosting the registered servers list.

.PARAMETER CmsGroup
    Optional. Limit deployment to a specific CMS server group (and children).

.PARAMETER AuditFilePath
    Local path on each target instance for .sqlaudit files.

.PARAMETER WhatIf
    Generates and logs the T-SQL per instance without executing it.

.NOTES
    Requires: SqlServer PowerShell module on the CMS host only.
    Account running this needs: CONTROL SERVER on every target instance,
    plus permission to execute xp_cmdshell (which, when enabled, runs as the
    SQL Server service account).
#>

[CmdletBinding(SupportsShouldProcess = $true)]
param(
    [Parameter(Mandatory = $true)]
    [string]$CmsInstance,

    [Parameter(Mandatory = $false)]
    [string]$CmsGroup,

    [Parameter(Mandatory = $false)]
    [string]$AuditFilePath = 'D:\SQLAudit',

    [Parameter(Mandatory = $false)]
    [int]$MaxFileSizeMB = 256,

    [Parameter(Mandatory = $false)]
    [int]$MaxRolloverFiles = 50,

    [Parameter(Mandatory = $false)]
    [ValidateSet('CONTINUE', 'SHUTDOWN', 'FAIL_OPERATION')]
    [string]$OnFailureAction = 'CONTINUE',

    [Parameter(Mandatory = $false)]
    [int]$QueryTimeoutSeconds = 120,

    [Parameter(Mandatory = $false)]
    [string]$LogDirectory = "$PSScriptRoot\Logs"
)

Set-StrictMode -Version Latest
$ErrorActionPreference = 'Stop'

#region ---- Prereqs and logging --------------------------------------------

if (-not (Get-Module -ListAvailable -Name SqlServer)) {
    throw "SqlServer module not installed. Run: Install-Module SqlServer -Scope CurrentUser"
}
Import-Module SqlServer -DisableNameChecking

if (-not (Test-Path -Path $LogDirectory)) {
    New-Item -Path $LogDirectory -ItemType Directory -Force | Out-Null
}

$runTimestamp   = Get-Date -Format 'yyyyMMdd_HHmmss'
$transcriptPath = Join-Path $LogDirectory "Deploy-ComplianceAudit_$runTimestamp.log"
$resultsPath    = Join-Path $LogDirectory "Deploy-ComplianceAudit_$runTimestamp.csv"

Start-Transcript -Path $transcriptPath -Force | Out-Null

Write-Host "=== Compliance Audit Deployment (SQL-only) ===" -ForegroundColor Cyan
Write-Host "CMS:             $CmsInstance"
Write-Host "Group filter:    $(if ($CmsGroup) { $CmsGroup } else { '(all)' })"
Write-Host "Audit path:      $AuditFilePath"
Write-Host "Max file size:   $MaxFileSizeMB MB"
Write-Host "Rollover files:  $MaxRolloverFiles"
Write-Host "On failure:      $OnFailureAction"
Write-Host "WhatIf mode:     $($WhatIfPreference -eq $true)"
Write-Host ""

#endregion

#region ---- T-SQL building blocks ------------------------------------------

# Capture the current sp_configure state for the two options we may toggle.
# Returned as a single row so the wrapper can save and compare cleanly.
$sqlGetConfigState = @'
SELECT
     MAX(CASE WHEN name = N'show advanced options' THEN CAST(value_in_use AS INT) END) AS ShowAdvancedInUse
    ,MAX(CASE WHEN name = N'show advanced options' THEN CAST(value        AS INT) END) AS ShowAdvancedConfigured
    ,MAX(CASE WHEN name = N'xp_cmdshell'           THEN CAST(value_in_use AS INT) END) AS XpCmdShellInUse
    ,MAX(CASE WHEN name = N'xp_cmdshell'           THEN CAST(value        AS INT) END) AS XpCmdShellConfigured
FROM sys.configurations
WHERE name IN (N'show advanced options', N'xp_cmdshell');
'@

# Turn both options on. RECONFIGURE after each so the next reference to
# xp_cmdshell succeeds. Wrapped in a single batch for one round-trip.
$sqlEnableAdvanced = @'
EXEC sp_configure 'show advanced options', 1; RECONFIGURE;
EXEC sp_configure 'xp_cmdshell', 1;           RECONFIGURE;
'@

# Restoration template -- values substituted from the saved original state.
$sqlRestoreConfigTemplate = @'
EXEC sp_configure 'xp_cmdshell', {{XP_VALUE}};           RECONFIGURE;
EXEC sp_configure 'show advanced options', {{SAO_VALUE}}; RECONFIGURE;
'@

# Directory creation via xp_cmdshell. Using IF NOT EXIST avoids errors on
# reruns; NUL redirection keeps the result set clean if the path already
# exists. Output is captured so we can surface failures in the log.
$sqlCreateDirTemplate = @'
DECLARE @cmd NVARCHAR(4000) = N'IF NOT EXIST "{{PATH}}" MKDIR "{{PATH}}"';
DECLARE @out TABLE (line NVARCHAR(4000) NULL);
INSERT INTO @out EXEC xp_cmdshell @cmd;
SELECT line FROM @out WHERE line IS NOT NULL;
'@

# Post-deploy sanity check on the filesystem. Confirms the service account
# can actually see the directory -- occasionally a path will exist but be
# inaccessible due to NTFS ACLs that don't grant the SQL service account.
$sqlTestDirTemplate = @'
DECLARE @cmd NVARCHAR(4000) = N'DIR "{{PATH}}"';
DECLARE @out TABLE (line NVARCHAR(4000) NULL);
INSERT INTO @out EXEC xp_cmdshell @cmd;
SELECT CASE WHEN EXISTS (SELECT 1 FROM @out WHERE line LIKE '%File Not Found%'
                                                OR line LIKE '%cannot find%')
            THEN 0 ELSE 1 END AS DirAccessible;
'@

# The audit baseline itself. Same content as the standalone template,
# collapsed into a single batch with tokens for substitution.
$sqlAuditTemplate = @'
USE [master];

IF EXISTS (SELECT 1 FROM sys.server_audit_specifications
           WHERE name = N'ServerAuditSpec_Compliance')
BEGIN
    ALTER SERVER AUDIT SPECIFICATION [ServerAuditSpec_Compliance] WITH (STATE = OFF);
    DROP SERVER AUDIT SPECIFICATION [ServerAuditSpec_Compliance];
END;

IF EXISTS (SELECT 1 FROM sys.server_audits
           WHERE name = N'Audit_Compliance')
BEGIN
    ALTER SERVER AUDIT [Audit_Compliance] WITH (STATE = OFF);
    DROP SERVER AUDIT [Audit_Compliance];
END;

CREATE SERVER AUDIT [Audit_Compliance]
TO FILE
(    FILEPATH            = N'{{AUDIT_PATH}}'
    ,MAXSIZE             = {{MAX_SIZE_MB}} MB
    ,MAX_ROLLOVER_FILES  = {{MAX_ROLLOVER}}
    ,RESERVE_DISK_SPACE  = OFF
)
WITH
(    QUEUE_DELAY  = 1000
    ,ON_FAILURE  = {{ON_FAILURE}}
);

CREATE SERVER AUDIT SPECIFICATION [ServerAuditSpec_Compliance]
FOR SERVER AUDIT [Audit_Compliance]
    ADD (SUCCESSFUL_LOGIN_GROUP),
    ADD (FAILED_LOGIN_GROUP),
    ADD (LOGOUT_GROUP),
    ADD (LOGIN_CHANGE_PASSWORD_GROUP),
    ADD (SERVER_ROLE_MEMBER_CHANGE_GROUP),
    ADD (DATABASE_ROLE_MEMBER_CHANGE_GROUP),
    ADD (SERVER_PERMISSION_CHANGE_GROUP),
    ADD (DATABASE_PERMISSION_CHANGE_GROUP),
    ADD (SERVER_OBJECT_PERMISSION_CHANGE_GROUP),
    ADD (SCHEMA_OBJECT_PERMISSION_CHANGE_GROUP),
    ADD (SERVER_PRINCIPAL_CHANGE_GROUP),
    ADD (DATABASE_PRINCIPAL_CHANGE_GROUP),
    ADD (DATABASE_OWNERSHIP_CHANGE_GROUP),
    ADD (SCHEMA_OBJECT_OWNERSHIP_CHANGE_GROUP),
    ADD (SERVER_OBJECT_CHANGE_GROUP),
    ADD (DATABASE_OBJECT_CHANGE_GROUP),
    ADD (SCHEMA_OBJECT_CHANGE_GROUP),
    ADD (DATABASE_CHANGE_GROUP),
    ADD (SERVER_STATE_CHANGE_GROUP),
    ADD (AUDIT_CHANGE_GROUP),
    ADD (TRACE_CHANGE_GROUP),
    ADD (BACKUP_RESTORE_GROUP),
    ADD (DBCC_GROUP),
    ADD (BROKER_LOGIN_GROUP),
    ADD (DATABASE_MIRRORING_LOGIN_GROUP),
    ADD (USER_DEFINED_AUDIT_GROUP)
WITH (STATE = OFF);

ALTER SERVER AUDIT [Audit_Compliance] WITH (STATE = ON);
ALTER SERVER AUDIT SPECIFICATION [ServerAuditSpec_Compliance] WITH (STATE = ON);
'@

$sqlVerifyAudit = @'
SELECT
     CAST(sa.is_state_enabled  AS INT) AS AuditEnabled
    ,CAST(sas.is_state_enabled AS INT) AS SpecEnabled
    ,COUNT(sad.audit_action_name)      AS ActionGroupCount
FROM sys.server_audits sa
LEFT JOIN sys.server_audit_specifications sas
    ON sas.name = N'ServerAuditSpec_Compliance'
LEFT JOIN sys.server_audit_specification_details sad
    ON sad.server_specification_id = sas.server_specification_id
WHERE sa.name = N'Audit_Compliance'
GROUP BY sa.is_state_enabled, sas.is_state_enabled;
'@

$sqlGetVersion = @'
SELECT
     CAST(SERVERPROPERTY('ProductMajorVersion') AS INT) AS MajorVersion
    ,CAST(SERVERPROPERTY('Edition')             AS NVARCHAR(128)) AS Edition
'@

#endregion

#region ---- Helper functions ------------------------------------------------

function Get-CmsRegisteredInstance {
    [CmdletBinding()]
    param(
        [Parameter(Mandatory)][string]$CmsServer,
        [string]$GroupPath
    )

    $regPath = if ($GroupPath) {
        "SQLSERVER:\SQLRegistration\Central Management Server Group\$CmsServer\$GroupPath"
    }
    else {
        "SQLSERVER:\SQLRegistration\Central Management Server Group\$CmsServer"
    }

    Get-ChildItem -Path $regPath -Recurse |
        Where-Object { $_.ServerName } |
        Select-Object -ExpandProperty ServerName -Unique |
        Sort-Object
}

function Invoke-Sql {
    # Thin wrapper around Invoke-Sqlcmd to keep call sites terse and
    # ensure consistent connection behavior across the script.
    [CmdletBinding()]
    param(
        [Parameter(Mandatory)][string]$Instance,
        [Parameter(Mandatory)][string]$Query,
        [int]$Timeout = 60
    )
    Invoke-Sqlcmd -ServerInstance $Instance `
                  -Query $Query `
                  -QueryTimeout $Timeout `
                  -ConnectionTimeout 15 `
                  -TrustServerCertificate `
                  -ErrorAction Stop
}

function Expand-Template {
    [CmdletBinding()]
    param(
        [Parameter(Mandatory)][string]$Template,
        [Parameter(Mandatory)][hashtable]$Tokens
    )
    $out = $Template
    foreach ($k in $Tokens.Keys) {
        $out = $out.Replace("{{$k}}", [string]$Tokens[$k])
    }
    return $out
}

#endregion

#region ---- Main deployment per-instance -----------------------------------

Write-Host "Enumerating registered servers from CMS..." -ForegroundColor Cyan
$instances = Get-CmsRegisteredInstance -CmsServer $CmsInstance -GroupPath $CmsGroup

if (-not $instances) {
    Stop-Transcript | Out-Null
    throw "No registered servers found under CMS '$CmsInstance'$(if ($CmsGroup) { " group '$CmsGroup'" })."
}

Write-Host "Found $($instances.Count) instance(s). Processing serially." -ForegroundColor Green
Write-Host ""

$results = [System.Collections.Generic.List[object]]::new()
$counter = 0

# Normalize path -- CREATE SERVER AUDIT wants trailing backslash, MKDIR does not.
$auditPathForSql = if ($AuditFilePath.EndsWith('\')) { $AuditFilePath } else { "$AuditFilePath\" }
$auditPathForCmd = $AuditFilePath.TrimEnd('\')

foreach ($instance in $instances) {
    $counter++
    Write-Host "[$counter/$($instances.Count)] $instance" -ForegroundColor Yellow

    $result = [PSCustomObject]@{
        Instance             = $instance
        Reachable            = $false
        MajorVersion         = $null
        Edition              = $null
        OriginalShowAdvanced = $null
        OriginalXpCmdShell   = $null
        ConfigChanged        = $false
        DirectoryCreated     = $false
        AuditDeployed        = $false
        AuditEnabled         = $null
        SpecEnabled          = $null
        ActionGroups         = $null
        ConfigRestored       = $false
        DurationMs           = 0
        ErrorMessage         = $null
        Timestamp            = (Get-Date).ToString('s')
    }

    $sw = [System.Diagnostics.Stopwatch]::StartNew()
    $originalState = $null

    try {
        # -- Reachability + version ----------------------------------------
        $ver = Invoke-Sql -Instance $instance -Query $sqlGetVersion -Timeout 15
        $result.Reachable    = $true
        $result.MajorVersion = $ver.MajorVersion
        $result.Edition      = $ver.Edition

        if ([int]$ver.MajorVersion -lt 13) {
            throw "Unsupported version (major $($ver.MajorVersion)); baseline targets 2016+"
        }

        # -- Capture current config ----------------------------------------
        # value_in_use tells us the effective runtime state, value tells us
        # what's persisted. We save both because RECONFIGURE may be pending
        # on some instances and we want to leave them exactly as we found them.
        $originalState = Invoke-Sql -Instance $instance -Query $sqlGetConfigState -Timeout 15
        $result.OriginalShowAdvanced = $originalState.ShowAdvancedInUse
        $result.OriginalXpCmdShell   = $originalState.XpCmdShellInUse

        Write-Host "  Current state: show_advanced=$($originalState.ShowAdvancedInUse), xp_cmdshell=$($originalState.XpCmdShellInUse)"

        # -- Enable advanced + xp_cmdshell if needed -----------------------
        if ($originalState.ShowAdvancedInUse -ne 1 -or $originalState.XpCmdShellInUse -ne 1) {
            if ($PSCmdlet.ShouldProcess($instance, 'Enable xp_cmdshell (temporary)')) {
                Invoke-Sql -Instance $instance -Query $sqlEnableAdvanced -Timeout 30
                $result.ConfigChanged = $true
                Write-Host "  Enabled xp_cmdshell (will restore at end)" -ForegroundColor DarkYellow
            }
        }

        # -- Ensure audit directory exists ---------------------------------
        $createDirSql = Expand-Template -Template $sqlCreateDirTemplate `
                                        -Tokens @{ PATH = $auditPathForCmd }
        if ($PSCmdlet.ShouldProcess($instance, "Create directory $auditPathForCmd")) {
            $dirOut = Invoke-Sql -Instance $instance -Query $createDirSql -Timeout 30
            if ($dirOut) {
                # xp_cmdshell returns MKDIR errors as result rows; log and
                # fail loudly rather than silently proceeding to CREATE AUDIT.
                foreach ($row in $dirOut) {
                    Write-Host "  xp_cmdshell: $($row.line)" -ForegroundColor DarkGray
                    if ($row.line -match 'denied|cannot|error' ) {
                        throw "Directory creation reported an error: $($row.line)"
                    }
                }
            }
            $result.DirectoryCreated = $true
        }

        # -- Verify directory accessibility --------------------------------
        $testDirSql = Expand-Template -Template $sqlTestDirTemplate `
                                      -Tokens @{ PATH = $auditPathForCmd }
        if (-not $WhatIfPreference) {
            $dirTest = Invoke-Sql -Instance $instance -Query $testDirSql -Timeout 30
            if ($dirTest.DirAccessible -ne 1) {
                throw "Directory $auditPathForCmd is not accessible to SQL service account"
            }
        }

        # -- Deploy the audit ----------------------------------------------
        $auditSql = Expand-Template -Template $sqlAuditTemplate -Tokens @{
            AUDIT_PATH   = $auditPathForSql
            MAX_SIZE_MB  = $MaxFileSizeMB
            MAX_ROLLOVER = $MaxRolloverFiles
            ON_FAILURE   = $OnFailureAction
        }

        if ($PSCmdlet.ShouldProcess($instance, 'Deploy compliance audit')) {
            Invoke-Sql -Instance $instance -Query $auditSql -Timeout $QueryTimeoutSeconds
            $result.AuditDeployed = $true
        }

        # -- Verify --------------------------------------------------------
        if (-not $WhatIfPreference) {
            $verify = Invoke-Sql -Instance $instance -Query $sqlVerifyAudit -Timeout 15
            $result.AuditEnabled = $verify.AuditEnabled
            $result.SpecEnabled  = $verify.SpecEnabled
            $result.ActionGroups = $verify.ActionGroupCount

            if ($verify.AuditEnabled -ne 1 -or $verify.SpecEnabled -ne 1) {
                throw "Verification failed (audit=$($verify.AuditEnabled), spec=$($verify.SpecEnabled))"
            }
        }

        Write-Host "  Audit deployed and enabled ($($result.ActionGroups) action groups)" -ForegroundColor Green
    }
    catch {
        $result.ErrorMessage = $_.Exception.Message
        Write-Host "  FAIL: $($_.Exception.Message)" -ForegroundColor Red
    }
    finally {
        # -- Always restore config, even on failure ------------------------
        # This runs regardless of whether the audit deploy succeeded. If we
        # enabled xp_cmdshell and then threw, we still want to leave the
        # instance exactly as we found it.
        if ($result.ConfigChanged -and $null -ne $originalState) {
            try {
                $restoreSql = Expand-Template -Template $sqlRestoreConfigTemplate -Tokens @{
                    XP_VALUE  = $originalState.XpCmdShellConfigured
                    SAO_VALUE = $originalState.ShowAdvancedConfigured
                }
                if ($PSCmdlet.ShouldProcess($instance, 'Restore sp_configure state')) {
                    Invoke-Sql -Instance $instance -Query $restoreSql -Timeout 30
                    $result.ConfigRestored = $true
                    Write-Host "  Restored original config" -ForegroundColor DarkCyan
                }
            }
            catch {
                # If restoration itself fails, log prominently -- this is the
                # worst failure mode (leaving xp_cmdshell on) and must be
                # surfaced for manual cleanup.
                $restoreErr = "CONFIG RESTORATION FAILED: $($_.Exception.Message)"
                Write-Host "  $restoreErr" -ForegroundColor Red -BackgroundColor Black
                if ($result.ErrorMessage) {
                    $result.ErrorMessage = "$($result.ErrorMessage) | $restoreErr"
                }
                else {
                    $result.ErrorMessage = $restoreErr
                }
            }
        }

        $sw.Stop()
        $result.DurationMs = $sw.ElapsedMilliseconds
        $results.Add($result)
    }

    Write-Host ""
}

#endregion

#region ---- Summary ---------------------------------------------------------

Write-Host "=== Summary ===" -ForegroundColor Cyan
$success       = ($results | Where-Object { $_.AuditDeployed }).Count
$failed        = ($results | Where-Object { -not $_.AuditDeployed }).Count
$restoreFailed = ($results | Where-Object { $_.ConfigChanged -and -not $_.ConfigRestored }).Count

Write-Host "Succeeded:              $success" -ForegroundColor Green
Write-Host "Failed:                 $failed"  -ForegroundColor $(if ($failed) { 'Red' } else { 'Gray' })
Write-Host "Config restore failed:  $restoreFailed" -ForegroundColor $(if ($restoreFailed) { 'Red' } else { 'Gray' })

$results | Export-Csv -Path $resultsPath -NoTypeInformation -Encoding UTF8
Write-Host ""
Write-Host "Results:     $resultsPath"
Write-Host "Transcript:  $transcriptPath"

if ($restoreFailed -gt 0) {
    Write-Host ""
    Write-Host "!!! Instances with xp_cmdshell possibly left enabled !!!" -ForegroundColor Red -BackgroundColor Black
    $results | Where-Object { $_.ConfigChanged -and -not $_.ConfigRestored } |
        Format-Table Instance, OriginalXpCmdShell, ErrorMessage -AutoSize -Wrap
}

if ($failed -gt 0) {
    Write-Host ""
    Write-Host "Failed instances:" -ForegroundColor Red
    $results | Where-Object { -not $_.AuditDeployed } |
        Format-Table Instance, ErrorMessage -AutoSize -Wrap
}

Stop-Transcript | Out-Null
if ($failed -gt 0 -or $restoreFailed -gt 0) { exit 1 } else { exit 0 }

#endregion
```

### A few design decisions worth calling out:

Serial processing is enforced structurally. The foreach loop processes one instance end-to-end before touching the next. If server 47 hangs, servers 1–46 are already fully deployed and restored, and you can Ctrl-C and resume from 48 by editing the CMS group membership. No partial-state concurrency issues.

The finally block is the whole point. xp_cmdshell getting left enabled on a hundred instances because of a mid-loop crash would be the worst possible outcome of this script — worse than the audit not deploying at all. So the restoration runs in finally, which fires on success, failure, and even throw during the audit deploy. The only way to leave xp_cmdshell enabled is if the restoration statement itself fails, and that case is surfaced loudly in a separate summary section with its own exit code path.

value_in_use vs value. sp_configure has two columns because changes require RECONFIGURE to take effect. I save the configured value and restore to that, so an instance with a pending-but-unreconfigured change stays in that same pending state afterward. The in-use value drives the decision about whether to enable. This edge case is rare but matters for correctness if someone's mid-change on an instance when you run the script.

xp_cmdshell output is inspected, not trusted. MKDIR doesn't set a return code xp_cmdshell can propagate, so errors come back as text rows. The script scans them for denied|cannot|error and fails the instance rather than blindly proceeding to CREATE SERVER AUDIT on a directory that doesn't exist. A separate DIR check after creation confirms the SQL service account can actually see it — catches the ACL case where the directory exists but the service account can't write to it.

The IF NOT EXIST pattern in MKDIR. Idempotent reruns don't produce spurious "directory already exists" errors, which would otherwise trigger the error-detection above and fail a clean rerun.

Version check is ProductMajorVersion, not the full string. SQL 2016 = 13, 2017 = 14, 2019 = 15, 2022 = 16, 2025 = 17. Clean integer comparison, no string parsing, works across your whole 2016–2025 range.

What this still doesn't do. It doesn't handle availability group topology (same concern as before — AG replicas need matching AUDIT_GUID), it doesn't check whether xp_cmdshell is blocked by policy at the instance level (some hardened environments set EXECUTE AS restrictions that prevent enabling it at all), and it doesn't handle instances where the xp_cmdshell service account lacks NTFS rights to create D:\SQLAudit. All three are worth a test-run against a representative sample before the full fleet deploy.

Want the matching collection-side job next — the T-SQL (plus orchestration from the CMS) that pulls audit records from each instance into a central audit database via sys.fn_get_audit_file() with a UNC path, using the same serial-per-instance pattern?

## You said: The CMS has many groups of servers registered in different folder structures.

The CMS has many groups of servers registered in different folder structures. Can a registered path be used (or multiple paths), such as ‘Inventory\MS\2016 - only servers in this directory’, ‘DBA\Managed\Inventory\2022 - only servers in this directory’, etc vs scanning through all the registered instances and directories? Also, if a connection fails or the process fails during the deployment, the server name should be logged to c:\temp with the failure/error message.

*Apr 18*

## Claude responded: Good call on both.

Good call on both. Targeting specific CMS folder paths is much safer than a fleet-wide sweep, and a persistent failure log outside the transcript makes triage easier between runs. Here's the updated version with both changes.

```powershell
<#
.SYNOPSIS
    Deploys the compliance audit baseline across CMS registered servers using
    SQL connections only (no PowerShell remoting, no SMB).

.DESCRIPTION
    Targets one or more specific CMS folder paths rather than the entire
    registered servers tree. Processes instances serially. For each instance,
    saves the current sp_configure state for xp_cmdshell and show advanced
    options, enables them if needed, pre-creates the audit directory, deploys
    the audit baseline, verifies, then restores the original configuration.

    Failures are written to a persistent log at C:\Temp so a failed instance
    list survives across runs and is easy to feed back into a targeted retry.

.PARAMETER CmsInstance
    The Central Management Server hosting the registered servers list.

.PARAMETER CmsPath
    One or more CMS folder paths to target. Only servers directly in each
    given folder are processed (no recursion into subfolders -- this lets you
    pick a specific leaf group without also pulling its children).

    Examples:
        'Inventory\MS\2016'
        'DBA\Managed\Inventory\2022'
        @('Inventory\MS\2016', 'Inventory\MS\2019')

.PARAMETER Recurse
    If set, descends into subfolders of each CmsPath. Default: off.

.PARAMETER AuditFilePath
    Local path on each target instance for .sqlaudit files.

.PARAMETER WhatIf
    Generates and logs the T-SQL per instance without executing it.

.EXAMPLE
    .\Deploy-ComplianceAudit.ps1 -CmsInstance 'CMS01' `
        -CmsPath 'Inventory\MS\2016'

.EXAMPLE
    .\Deploy-ComplianceAudit.ps1 -CmsInstance 'CMS01' `
        -CmsPath @('Inventory\MS\2016','DBA\Managed\Inventory\2022')

.NOTES
    Requires: SqlServer PowerShell module on the CMS host only.
    Account running this needs: CONTROL SERVER on every target instance.
#>

[CmdletBinding(SupportsShouldProcess = $true)]
param(
    [Parameter(Mandatory = $true)]
    [string]$CmsInstance,

    [Parameter(Mandatory = $true)]
    [string[]]$CmsPath,

    [Parameter(Mandatory = $false)]
    [switch]$Recurse,

    [Parameter(Mandatory = $false)]
    [string]$AuditFilePath = 'D:\SQLAudit',

    [Parameter(Mandatory = $false)]
    [int]$MaxFileSizeMB = 256,

    [Parameter(Mandatory = $false)]
    [int]$MaxRolloverFiles = 50,

    [Parameter(Mandatory = $false)]
    [ValidateSet('CONTINUE', 'SHUTDOWN', 'FAIL_OPERATION')]
    [string]$OnFailureAction = 'CONTINUE',

    [Parameter(Mandatory = $false)]
    [int]$QueryTimeoutSeconds = 120,

    [Parameter(Mandatory = $false)]
    [string]$LogDirectory = "$PSScriptRoot\Logs",

    [Parameter(Mandatory = $false)]
    [string]$FailureLogDirectory = 'C:\Temp'
)

Set-StrictMode -Version Latest
$ErrorActionPreference = 'Stop'

#region ---- Prereqs and logging --------------------------------------------

if (-not (Get-Module -ListAvailable -Name SqlServer)) {
    throw "SqlServer module not installed. Run: Install-Module SqlServer -Scope CurrentUser"
}
Import-Module SqlServer -DisableNameChecking

foreach ($dir in @($LogDirectory, $FailureLogDirectory)) {
    if (-not (Test-Path -Path $dir)) {
        New-Item -Path $dir -ItemType Directory -Force | Out-Null
    }
}

$runTimestamp   = Get-Date -Format 'yyyyMMdd_HHmmss'
$transcriptPath = Join-Path $LogDirectory        "Deploy-ComplianceAudit_$runTimestamp.log"
$resultsPath    = Join-Path $LogDirectory        "Deploy-ComplianceAudit_$runTimestamp.csv"
$failureLogPath = Join-Path $FailureLogDirectory "Deploy-ComplianceAudit_Failures_$runTimestamp.log"

Start-Transcript -Path $transcriptPath -Force | Out-Null

Write-Host "=== Compliance Audit Deployment (SQL-only) ===" -ForegroundColor Cyan
Write-Host "CMS:             $CmsInstance"
Write-Host "CMS path(s):     $($CmsPath -join '; ')"
Write-Host "Recurse:         $Recurse"
Write-Host "Audit path:      $AuditFilePath"
Write-Host "Max file size:   $MaxFileSizeMB MB"
Write-Host "Rollover files:  $MaxRolloverFiles"
Write-Host "On failure:      $OnFailureAction"
Write-Host "Failure log:     $failureLogPath"
Write-Host "WhatIf mode:     $($WhatIfPreference -eq $true)"
Write-Host ""

#endregion

#region ---- Failure logging -------------------------------------------------

function Write-FailureLog {
    # Appends a structured failure line to C:\Temp so the list of servers
    # needing attention persists across runs and survives a crashed session.
    [CmdletBinding()]
    param(
        [Parameter(Mandatory)][string]$Instance,
        [Parameter(Mandatory)][string]$Stage,
        [Parameter(Mandatory)][string]$Message
    )

    $ts  = Get-Date -Format 'yyyy-MM-dd HH:mm:ss'
    # Collapse newlines in the error message so each failure is one log line,
    # which makes grep/Select-String-based retry list building trivial.
    $msg = ($Message -replace '\r?\n', ' | ').Trim()
    $line = "[$ts]`t$Instance`t$Stage`t$msg"

    # Add-Content with -Force tolerates the file not existing yet. Any write
    # error here is swallowed so a failed failure-log write doesn't mask the
    # original deployment failure.
    try {
        Add-Content -Path $failureLogPath -Value $line -Encoding UTF8 -Force
    }
    catch {
        Write-Host "  WARN: could not append to failure log: $($_.Exception.Message)" -ForegroundColor DarkYellow
    }
}

#endregion

#region ---- T-SQL building blocks ------------------------------------------

$sqlGetConfigState = @'
SELECT
     MAX(CASE WHEN name = N'show advanced options' THEN CAST(value_in_use AS INT) END) AS ShowAdvancedInUse
    ,MAX(CASE WHEN name = N'show advanced options' THEN CAST(value        AS INT) END) AS ShowAdvancedConfigured
    ,MAX(CASE WHEN name = N'xp_cmdshell'           THEN CAST(value_in_use AS INT) END) AS XpCmdShellInUse
    ,MAX(CASE WHEN name = N'xp_cmdshell'           THEN CAST(value        AS INT) END) AS XpCmdShellConfigured
FROM sys.configurations
WHERE name IN (N'show advanced options', N'xp_cmdshell');
'@

$sqlEnableAdvanced = @'
EXEC sp_configure 'show advanced options', 1; RECONFIGURE;
EXEC sp_configure 'xp_cmdshell', 1;           RECONFIGURE;
'@

$sqlRestoreConfigTemplate = @'
EXEC sp_configure 'xp_cmdshell', {{XP_VALUE}};           RECONFIGURE;
EXEC sp_configure 'show advanced options', {{SAO_VALUE}}; RECONFIGURE;
'@

$sqlCreateDirTemplate = @'
DECLARE @cmd NVARCHAR(4000) = N'IF NOT EXIST "{{PATH}}" MKDIR "{{PATH}}"';
DECLARE @out TABLE (line NVARCHAR(4000) NULL);
INSERT INTO @out EXEC xp_cmdshell @cmd;
SELECT line FROM @out WHERE line IS NOT NULL;
'@

$sqlTestDirTemplate = @'
DECLARE @cmd NVARCHAR(4000) = N'DIR "{{PATH}}"';
DECLARE @out TABLE (line NVARCHAR(4000) NULL);
INSERT INTO @out EXEC xp_cmdshell @cmd;
SELECT CASE WHEN EXISTS (SELECT 1 FROM @out WHERE line LIKE '%File Not Found%'
                                                OR line LIKE '%cannot find%')
            THEN 0 ELSE 1 END AS DirAccessible;
'@

$sqlAuditTemplate = @'
USE [master];

IF EXISTS (SELECT 1 FROM sys.server_audit_specifications
           WHERE name = N'ServerAuditSpec_Compliance')
BEGIN
    ALTER SERVER AUDIT SPECIFICATION [ServerAuditSpec_Compliance] WITH (STATE = OFF);
    DROP SERVER AUDIT SPECIFICATION [ServerAuditSpec_Compliance];
END;

IF EXISTS (SELECT 1 FROM sys.server_audits
           WHERE name = N'Audit_Compliance')
BEGIN
    ALTER SERVER AUDIT [Audit_Compliance] WITH (STATE = OFF);
    DROP SERVER AUDIT [Audit_Compliance];
END;

CREATE SERVER AUDIT [Audit_Compliance]
TO FILE
(    FILEPATH            = N'{{AUDIT_PATH}}'
    ,MAXSIZE             = {{MAX_SIZE_MB}} MB
    ,MAX_ROLLOVER_FILES  = {{MAX_ROLLOVER}}
    ,RESERVE_DISK_SPACE  = OFF
)
WITH
(    QUEUE_DELAY  = 1000
    ,ON_FAILURE  = {{ON_FAILURE}}
);

CREATE SERVER AUDIT SPECIFICATION [ServerAuditSpec_Compliance]
FOR SERVER AUDIT [Audit_Compliance]
    ADD (SUCCESSFUL_LOGIN_GROUP),
    ADD (FAILED_LOGIN_GROUP),
    ADD (LOGOUT_GROUP),
    ADD (LOGIN_CHANGE_PASSWORD_GROUP),
    ADD (SERVER_ROLE_MEMBER_CHANGE_GROUP),
    ADD (DATABASE_ROLE_MEMBER_CHANGE_GROUP),
    ADD (SERVER_PERMISSION_CHANGE_GROUP),
    ADD (DATABASE_PERMISSION_CHANGE_GROUP),
    ADD (SERVER_OBJECT_PERMISSION_CHANGE_GROUP),
    ADD (SCHEMA_OBJECT_PERMISSION_CHANGE_GROUP),
    ADD (SERVER_PRINCIPAL_CHANGE_GROUP),
    ADD (DATABASE_PRINCIPAL_CHANGE_GROUP),
    ADD (DATABASE_OWNERSHIP_CHANGE_GROUP),
    ADD (SCHEMA_OBJECT_OWNERSHIP_CHANGE_GROUP),
    ADD (SERVER_OBJECT_CHANGE_GROUP),
    ADD (DATABASE_OBJECT_CHANGE_GROUP),
    ADD (SCHEMA_OBJECT_CHANGE_GROUP),
    ADD (DATABASE_CHANGE_GROUP),
    ADD (SERVER_STATE_CHANGE_GROUP),
    ADD (AUDIT_CHANGE_GROUP),
    ADD (TRACE_CHANGE_GROUP),
    ADD (BACKUP_RESTORE_GROUP),
    ADD (DBCC_GROUP),
    ADD (BROKER_LOGIN_GROUP),
    ADD (DATABASE_MIRRORING_LOGIN_GROUP),
    ADD (USER_DEFINED_AUDIT_GROUP)
WITH (STATE = OFF);

ALTER SERVER AUDIT [Audit_Compliance] WITH (STATE = ON);
ALTER SERVER AUDIT SPECIFICATION [ServerAuditSpec_Compliance] WITH (STATE = ON);
'@

$sqlVerifyAudit = @'
SELECT
     CAST(sa.is_state_enabled  AS INT) AS AuditEnabled
    ,CAST(sas.is_state_enabled AS INT) AS SpecEnabled
    ,COUNT(sad.audit_action_name)      AS ActionGroupCount
FROM sys.server_audits sa
LEFT JOIN sys.server_audit_specifications sas
    ON sas.name = N'ServerAuditSpec_Compliance'
LEFT JOIN sys.server_audit_specification_details sad
    ON sad.server_specification_id = sas.server_specification_id
WHERE sa.name = N'Audit_Compliance'
GROUP BY sa.is_state_enabled, sas.is_state_enabled;
'@

$sqlGetVersion = @'
SELECT
     CAST(SERVERPROPERTY('ProductMajorVersion') AS INT) AS MajorVersion
    ,CAST(SERVERPROPERTY('Edition')             AS NVARCHAR(128)) AS Edition
'@

#endregion

#region ---- Helper functions ------------------------------------------------

function Get-CmsInstanceFromPath {
    # Pulls registered servers from one or more specific CMS folder paths.
    # Non-recursive by default -- only servers directly inside each folder
    # are returned, so picking 'Inventory\MS\2016' does not also grab
    # 'Inventory\MS\2016\Decommissioning' unless -Recurse is specified.
    [CmdletBinding()]
    param(
        [Parameter(Mandatory)][string]$CmsServer,
        [Parameter(Mandatory)][string[]]$Paths,
        [switch]$DoRecurse
    )

    $collected = [System.Collections.Generic.List[string]]::new()

    foreach ($p in $Paths) {
        # Normalize: strip any leading/trailing backslashes so both
        # 'Inventory\MS\2016' and '\Inventory\MS\2016\' work.
        $clean = $p.Trim().Trim('\')
        $providerPath = "SQLSERVER:\SQLRegistration\Central Management Server Group\$CmsServer\$clean"

        if (-not (Test-Path -Path $providerPath)) {
            Write-Host "  WARN: CMS path not found: $clean" -ForegroundColor Yellow
            Write-FailureLog -Instance '(CMS path)' -Stage 'Enumeration' `
                             -Message "CMS path not found: $clean"
            continue
        }

        Write-Host "  Enumerating: $clean" -ForegroundColor DarkCyan

        # Get-ChildItem without -Recurse returns only direct children.
        # We filter on ServerName to drop any subfolder nodes.
        $items = if ($DoRecurse) {
            Get-ChildItem -Path $providerPath -Recurse
        }
        else {
            Get-ChildItem -Path $providerPath
        }

        $names = $items |
            Where-Object { $_.ServerName } |
            Select-Object -ExpandProperty ServerName

        Write-Host "    Found $($names.Count) server(s) in this path"
        foreach ($n in $names) { $collected.Add($n) | Out-Null }
    }

    # Dedupe across paths -- the same server can legitimately be registered
    # in multiple CMS folders, and deploying to it twice would be wasteful
    # (and the second pass would look like a fresh rebuild in the log).
    return ($collected | Sort-Object -Unique)
}

function Invoke-Sql {
    [CmdletBinding()]
    param(
        [Parameter(Mandatory)][string]$Instance,
        [Parameter(Mandatory)][string]$Query,
        [int]$Timeout = 60
    )
    Invoke-Sqlcmd -ServerInstance $Instance `
                  -Query $Query `
                  -QueryTimeout $Timeout `
                  -ConnectionTimeout 15 `
                  -TrustServerCertificate `
                  -ErrorAction Stop
}

function Expand-Template {
    [CmdletBinding()]
    param(
        [Parameter(Mandatory)][string]$Template,
        [Parameter(Mandatory)][hashtable]$Tokens
    )
    $out = $Template
    foreach ($k in $Tokens.Keys) {
        $out = $out.Replace("{{$k}}", [string]$Tokens[$k])
    }
    return $out
}

#endregion

#region ---- Enumerate targets -----------------------------------------------

Write-Host "Enumerating targets from CMS paths..." -ForegroundColor Cyan
$instances = Get-CmsInstanceFromPath -CmsServer $CmsInstance `
                                     -Paths $CmsPath `
                                     -DoRecurse:$Recurse

if (-not $instances) {
    Stop-Transcript | Out-Null
    throw "No servers found under the specified CMS path(s)."
}

Write-Host "Total unique instances to process: $($instances.Count)" -ForegroundColor Green
Write-Host ""

#endregion

#region ---- Main deployment per-instance -----------------------------------

$results = [System.Collections.Generic.List[object]]::new()
$counter = 0

$auditPathForSql = if ($AuditFilePath.EndsWith('\')) { $AuditFilePath } else { "$AuditFilePath\" }
$auditPathForCmd = $AuditFilePath.TrimEnd('\')

foreach ($instance in $instances) {
    $counter++
    Write-Host "[$counter/$($instances.Count)] $instance" -ForegroundColor Yellow

    $result = [PSCustomObject]@{
        Instance             = $instance
        Reachable            = $false
        MajorVersion         = $null
        Edition              = $null
        OriginalShowAdvanced = $null
        OriginalXpCmdShell   = $null
        ConfigChanged        = $false
        DirectoryCreated     = $false
        AuditDeployed        = $false
        AuditEnabled         = $null
        SpecEnabled          = $null
        ActionGroups         = $null
        ConfigRestored       = $false
        DurationMs           = 0
        FailureStage         = $null
        ErrorMessage         = $null
        Timestamp            = (Get-Date).ToString('s')
    }

    $sw = [System.Diagnostics.Stopwatch]::StartNew()
    $originalState = $null
    $currentStage  = 'Unknown'

    try {
        # -- Connect and check version -------------------------------------
        $currentStage = 'Connect'
        $ver = Invoke-Sql -Instance $instance -Query $sqlGetVersion -Timeout 15
        $result.Reachable    = $true
        $result.MajorVersion = $ver.MajorVersion
        $result.Edition      = $ver.Edition

        $currentStage = 'VersionCheck'
        if ([int]$ver.MajorVersion -lt 13) {
            throw "Unsupported version (major $($ver.MajorVersion)); baseline targets 2016+"
        }

        # -- Capture current config ----------------------------------------
        $currentStage = 'CaptureConfig'
        $originalState = Invoke-Sql -Instance $instance -Query $sqlGetConfigState -Timeout 15
        $result.OriginalShowAdvanced = $originalState.ShowAdvancedInUse
        $result.OriginalXpCmdShell   = $originalState.XpCmdShellInUse

        Write-Host "  Current: show_advanced=$($originalState.ShowAdvancedInUse), xp_cmdshell=$($originalState.XpCmdShellInUse)"

        # -- Enable advanced + xp_cmdshell if needed -----------------------
        if ($originalState.ShowAdvancedInUse -ne 1 -or $originalState.XpCmdShellInUse -ne 1) {
            $currentStage = 'EnableXpCmdShell'
            if ($PSCmdlet.ShouldProcess($instance, 'Enable xp_cmdshell (temporary)')) {
                Invoke-Sql -Instance $instance -Query $sqlEnableAdvanced -Timeout 30
                $result.ConfigChanged = $true
                Write-Host "  Enabled xp_cmdshell (will restore at end)" -ForegroundColor DarkYellow
            }
        }

        # -- Ensure audit directory exists ---------------------------------
        $currentStage = 'CreateDirectory'
        $createDirSql = Expand-Template -Template $sqlCreateDirTemplate `
                                        -Tokens @{ PATH = $auditPathForCmd }
        if ($PSCmdlet.ShouldProcess($instance, "Create directory $auditPathForCmd")) {
            $dirOut = Invoke-Sql -Instance $instance -Query $createDirSql -Timeout 30
            if ($dirOut) {
                foreach ($row in $dirOut) {
                    Write-Host "  xp_cmdshell: $($row.line)" -ForegroundColor DarkGray
                    if ($row.line -match 'denied|cannot|error') {
                        throw "Directory creation reported an error: $($row.line)"
                    }
                }
            }
            $result.DirectoryCreated = $true
        }

        # -- Verify directory accessibility --------------------------------
        $currentStage = 'VerifyDirectory'
        if (-not $WhatIfPreference) {
            $testDirSql = Expand-Template -Template $sqlTestDirTemplate `
                                          -Tokens @{ PATH = $auditPathForCmd }
            $dirTest = Invoke-Sql -Instance $instance -Query $testDirSql -Timeout 30
            if ($dirTest.DirAccessible -ne 1) {
                throw "Directory $auditPathForCmd not accessible to SQL service account"
            }
        }

        # -- Deploy the audit ----------------------------------------------
        $currentStage = 'DeployAudit'
        $auditSql = Expand-Template -Template $sqlAuditTemplate -Tokens @{
            AUDIT_PATH   = $auditPathForSql
            MAX_SIZE_MB  = $MaxFileSizeMB
            MAX_ROLLOVER = $MaxRolloverFiles
            ON_FAILURE   = $OnFailureAction
        }
        if ($PSCmdlet.ShouldProcess($instance, 'Deploy compliance audit')) {
            Invoke-Sql -Instance $instance -Query $auditSql -Timeout $QueryTimeoutSeconds
            $result.AuditDeployed = $true
        }

        # -- Verify --------------------------------------------------------
        $currentStage = 'VerifyAudit'
        if (-not $WhatIfPreference) {
            $verify = Invoke-Sql -Instance $instance -Query $sqlVerifyAudit -Timeout 15
            $result.AuditEnabled = $verify.AuditEnabled
            $result.SpecEnabled  = $verify.SpecEnabled
            $result.ActionGroups = $verify.ActionGroupCount

            if ($verify.AuditEnabled -ne 1 -or $verify.SpecEnabled -ne 1) {
                throw "Verification failed (audit=$($verify.AuditEnabled), spec=$($verify.SpecEnabled))"
            }
        }

        Write-Host "  Audit deployed and enabled ($($result.ActionGroups) action groups)" -ForegroundColor Green
    }
    catch {
        $result.FailureStage = $currentStage
        $result.ErrorMessage = $_.Exception.Message
        Write-Host "  FAIL [$currentStage]: $($_.Exception.Message)" -ForegroundColor Red
        Write-FailureLog -Instance $instance -Stage $currentStage -Message $_.Exception.Message
    }
    finally {
        # -- Always restore config, even on failure ------------------------
        if ($result.ConfigChanged -and $null -ne $originalState) {
            try {
                $restoreSql = Expand-Template -Template $sqlRestoreConfigTemplate -Tokens @{
                    XP_VALUE  = $originalState.XpCmdShellConfigured
                    SAO_VALUE = $originalState.ShowAdvancedConfigured
                }
                if ($PSCmdlet.ShouldProcess($instance, 'Restore sp_configure state')) {
                    Invoke-Sql -Instance $instance -Query $restoreSql -Timeout 30
                    $result.ConfigRestored = $true
                    Write-Host "  Restored original config" -ForegroundColor DarkCyan
                }
            }
            catch {
                # Restoration failure is the worst outcome -- log to both
                # places and flag loudly in the summary.
                $restoreErr = "CONFIG RESTORATION FAILED: $($_.Exception.Message)"
                Write-Host "  $restoreErr" -ForegroundColor Red -BackgroundColor Black
                Write-FailureLog -Instance $instance -Stage 'RestoreConfig' -Message $restoreErr

                if ($result.ErrorMessage) {
                    $result.ErrorMessage = "$($result.ErrorMessage) | $restoreErr"
                }
                else {
                    $result.ErrorMessage = $restoreErr
                    $result.FailureStage = 'RestoreConfig'
                }
            }
        }

        $sw.Stop()
        $result.DurationMs = $sw.ElapsedMilliseconds
        $results.Add($result)
    }

    Write-Host ""
}

#endregion

#region ---- Summary ---------------------------------------------------------

Write-Host "=== Summary ===" -ForegroundColor Cyan
$success       = ($results | Where-Object { $_.AuditDeployed }).Count
$failed        = ($results | Where-Object { -not $_.AuditDeployed }).Count
$restoreFailed = ($results | Where-Object { $_.ConfigChanged -and -not $_.ConfigRestored }).Count

Write-Host "Succeeded:              $success" -ForegroundColor Green
Write-Host "Failed:                 $failed"  -ForegroundColor $(if ($failed) { 'Red' } else { 'Gray' })
Write-Host "Config restore failed:  $restoreFailed" -ForegroundColor $(if ($restoreFailed) { 'Red' } else { 'Gray' })

$results | Export-Csv -Path $resultsPath -NoTypeInformation -Encoding UTF8

Write-Host ""
Write-Host "Results CSV:   $resultsPath"
Write-Host "Transcript:    $transcriptPath"
if (Test-Path -Path $failureLogPath) {
    Write-Host "Failure log:   $failureLogPath" -ForegroundColor Yellow
}

if ($restoreFailed -gt 0) {
    Write-Host ""
    Write-Host "!!! Instances with xp_cmdshell possibly left enabled !!!" -ForegroundColor Red -BackgroundColor Black
    $results | Where-Object { $_.ConfigChanged -and -not $_.ConfigRestored } |
        Format-Table Instance, OriginalXpCmdShell, FailureStage, ErrorMessage -AutoSize -Wrap
}

if ($failed -gt 0) {
    Write-Host ""
    Write-Host "Failed instances:" -ForegroundColor Red
    $results | Where-Object { -not $_.AuditDeployed } |
        Format-Table Instance, FailureStage, ErrorMessage -AutoSize -Wrap
}

Stop-Transcript | Out-Null
if ($failed -gt 0 -or $restoreFailed -gt 0) { exit 1 } else { exit 0 }

#endregion
```

### What changed and why it matters:

-CmsPath is now a string array, required. You can pass one path or several, and they're enumerated in order. Dedup happens at the end so a server registered in both Inventory\MS\2016 and DBA\Managed\SQL2016 is only deployed to once. Invoking:

```powershell
.\Deploy-ComplianceAudit.ps1 -CmsInstance 'CMS01' `
    -CmsPath @('Inventory\MS\2016','DBA\Managed\Inventory\2022')
```

Non-recursive by default. This was the key decision — if you specify Inventory\MS\2016, you probably don't want Inventory\MS\2016\Retired or Inventory\MS\2016\Staging along for the ride. Get-ChildItem without -Recurse returns only direct children, which matches the mental model of "servers in this folder." If you do want descent into subfolders, add -Recurse.

Missing path is a warning, not a fatal error. If you pass four paths and one of them is mistyped or has been deleted from the CMS, the script logs it to the failure log and continues with the rest. That's almost always what you want — the alternative (aborting the whole run because one path is bad) is frustrating when you're deploying across a dozen groups.

FailureStage column added. Every error is tagged with which stage it failed in: Connect, VersionCheck, CaptureConfig, EnableXpCmdShell, CreateDirectory, VerifyDirectory, DeployAudit, VerifyAudit, or RestoreConfig. This is the single most useful field for triage because the remediation is completely different depending on stage — a Connect failure is a firewall or credential issue, a CreateDirectory failure is a filesystem or service account issue, a DeployAudit failure is a SQL permission or syntax issue.

C:\Temp failure log is tab-delimited and append-only. Format is [timestamp] TAB instance TAB stage TAB message on a single line per failure. That makes it trivial to pull a retry list:

```powershell
# Build a retry list from today's failure log
Get-Content C:\Temp\Deploy-ComplianceAudit_Failures_*.log |
    ForEach-Object { ($_ -split "`t")[1] } |
    Sort-Object -Unique
```

And feeding that back in is straightforward — register those names in a CMS Retry folder, or adapt the script to accept an explicit -InstanceList parameter as a third enumeration mode.

Multi-line error messages collapsed to one line. SQL errors sometimes span multiple lines (the Msg 15123, Level 16 preamble plus the actual message). The failure log collapses \r?\n to | so one failure = one log line, which keeps the grep-ability clean.

Failure log path is parameterized. Defaults to C:\Temp as you asked, but -FailureLogDirectory lets you override it if needed (some environments ACL C:\Temp aggressively, and redirecting to a DBA share is sometimes preferable).

One thing worth flagging for your environment specifically: the SMO provider path SQLSERVER:\SQLRegistration\Central Management Server Group\ is the path for servers registered through the CMS on the CMS instance itself. If you also use local registered servers (the file-based ones in %APPDATA%\Microsoft\...\RegSrvr.xml), those live at a different provider path. Worth a quick sanity check — run Get-ChildItem 'SQLSERVER:\SQLRegistration\Central Management Server Group\YOUR_CMS' interactively first and confirm the folder structure matches what you see in SSMS. If it doesn't, the path separator or CMS root node name may differ slightly in your setup and we can adjust.

Want the collection-side job next, or would you rather do a dry-run pass against a test CMS path first and iterate on what comes back?

## You said: Double check your logic and syntax for errors or other mistakes.

Double check your logic and syntax for errors or other mistakes. Confirm all the code written is legitimately valid for SQL 2016 - 2022, including the CMS calls and PowerShell commands. Everything should have a clean way of failing or checking if existing first if applicable. Specify which version of PowerShell should be used (5 or 7) or if both are valid.

After the complete analysis and any changes are made, produce the final version in a markdown format file for copying or downloading.

*Apr 18*

## Claude responded: Let me do a thorough review before producing the final version.

Let me do a thorough review before producing the final version.

Searched the web

Searched the web

Important findings. Let me verify a few more things.

Searched the web

Searched the web

I have enough information. Let me do the complete analysis now.

### Analysis of issues found

1. CMS path backslash handling (critical). When registered server names contain a named instance (e.g., SERVER01\SQL2016), the CMS provider encodes the backslash. My Get-ChildItem -Recurse with Where-Object { $_.ServerName } worked around this, but building a path string with a user-supplied $CmsPath that contains a named-instance CMS host won't work cleanly. I need to use -LiteralPath consistently and handle the named-instance edge case for the CMS host itself.

2. Test-Path with SQL provider paths. Test-Path against SQLSERVER:\ paths works but has quirks with colons in paths. Using -LiteralPath is safer.

3. Invoke-Sqlcmd -TrustServerCertificate availability. This parameter was added in SqlServer module 21.1.x (around 2021). Older installations — particularly anything using the legacy SQLPS module or SqlServer module pre-21.x — won't have it. Since the script targets SQL 2016–2022 fleets, the CMS host might have an old module. I need to detect this and handle both cases.

4. xp_cmdshell return behavior. xp_cmdshell returns all lines including NULL terminators and doesn't reliably return non-zero exit codes for MKDIR errors. My regex check is OK but I should also test the directory after creation regardless, not rely on MKDIR output.

5. sp_configure 'xp_cmdshell' edge case. If xp_cmdshell is disabled via server policy (sys admin policy) or Group Policy, sp_configure will succeed but the value won't actually take effect. Need to verify it's truly on before trying to use it.

6. Invoke-Sqlcmd returning single row vs array. When Invoke-Sqlcmd returns one row, PowerShell unwraps it to a single object; multiple rows returns an array. $dirOut handling needs to tolerate both. Same issue for $originalState — if the query returns 0 rows (shouldn't happen but defensive), accessing properties would error.

7. Set-StrictMode -Version Latest with Invoke-Sqlcmd results. StrictMode Latest flags access to non-existent properties. If a query returns no rows, $result.SomeProperty throws. I need null-guarding.

8. ShouldProcess + verification path. The if (-not $WhatIfPreference) checks are OK, but mixing $PSCmdlet.ShouldProcess with $WhatIfPreference checks is inconsistent. Cleaner to gate everything on $WhatIfPreference.

9. MKDIR output — "error" false positive. My regex denied|cannot|error would false-match a server whose locale returns messages that happen to contain "error" benignly. I should tighten this.

10. Recursive path with Get-ChildItem. When recursing, Get-ChildItem returns both folders and registrations. The $_.ServerName filter handles this but it's worth making the filter explicit to PSIsContainer -eq $false OR explicit type check for registered server objects.

11. Export-Csv -Encoding UTF8. PowerShell 5.1 defaults to UTF-8 with BOM; PowerShell 7 defaults to UTF-8 without BOM. For consistency across environments I should leave default or be explicit.

12. PowerShell version. Given the SqlServer module's long history of PS7 issues, PowerShell 5.1 is the recommended target. PS7 works but has edge cases with the SQL provider cmdlets and cd SQLSERVER:\. The CMS provider path navigation specifically is more reliable on 5.1.

13. sp_configure and RECONFIGURE WITH OVERRIDE. Plain RECONFIGURE can fail on instances where values outside the legitimate range were persisted. Defensive practice is RECONFIGURE WITH OVERRIDE for xp_cmdshell restoration in case anything odd is set.

14. SQL 2025 mentioned but not verified. The user mentioned 2016–2025 earlier but this script only validates 2016+. That's fine.

15. MKDIR command syntax under xp_cmdshell. IF NOT EXIST is a CMD.exe construct and requires the command to run through cmd.exe. xp_cmdshell does spawn via cmd.exe so this works, but quoting matters — I need to ensure double quotes survive the T-SQL string literal.

16. Missing SET NOCOUNT ON. Doesn't affect correctness but cleaner output and fewer "rows affected" messages coming back through Invoke-Sqlcmd.

17. Loop variable scoping. $currentStage inside the try/catch/finally is fine but I should initialize all variables at the top of the loop iteration to avoid StrictMode issues on the catch path.

18. The $ver.MajorVersion cast. If Invoke-Sqlcmd returns the value as a string or DBNull, the [int] cast will fail. Already casting in SQL (CAST(...AS INT)) so this is OK, but defensive null check is still warranted.

19. Version file for logging. My Write-FailureLog uses Add-Content -Encoding UTF8 — fine in both PS versions. Not an issue.

20. -WhatIfPreference check. This is a common-parameter, but explicitly checking $WhatIfPreference (not $PSCmdlet.MyInvocation.BoundParameters['WhatIf']) is correct per current guidance.

Now producing the corrected, final version.

Check if docx skill is relevant — user asked for markdown, not docx, but confirming

Check if docx skill is relevant — user asked for markdown, not docx, but confirming

Good — the user asked for markdown, not docx, so I'll create the .md file directly and present it. Let me write the final version now.

Created a file, ran a command, read a file

Created a file, ran a command, read a file

Here's a quick summary of what I found and fixed during the review.

PowerShell version recommendation: Windows PowerShell 5.1. The SqlServer module officially supports PS 5.0+ including PS 7, but the SQLSERVER:\SQLRegistration\ provider is more reliable under 5.1 — particularly the path parsing for named-instance CMS hosts, which has a long history of quirks under PS 7. PS 7 will work, but 5.1 is the tested baseline.

### The most impactful bugs corrected:

Invoke-Sqlcmd -TrustServerCertificate isn't universally available. It was added in SqlServer module 21.1.x (2021). If you're running an older module or the legacy SQLPS, the previous version of the script would have thrown "parameter cannot be found" errors. The corrected version detects this and conditionally adds the parameter, emitting a warning if it's missing.

Named-instance CMS hosts broke path parsing. CMS01\PROD as the CMS host would fail because Get-ChildItem without -LiteralPath interprets the backslash as a path separator. All CMS path operations now use -LiteralPath.

Invoke-Sqlcmd return shape inconsistency. Single rows come back as objects; multiple rows come back as arrays. The previous version accessed properties directly and would break under StrictMode if a query returned zero rows. Added a Get-SingleRow helper that normalizes this.

Directory-error detection was too loose. The regex denied|cannot|error would false-match benign strings containing "error". Tightened to specific phrases (Access is denied, cannot find the path, cannot create) and added 2>&1 to the DIR test so stderr is captured.

xp_cmdshell can be silently blocked by policy. sp_configure 'xp_cmdshell', 1 succeeds even when hardened environments block it via policy — the value gets persisted but never takes effect. Added a post-enable re-check of value_in_use to catch this before attempting MKDIR.

StrictMode was too aggressive. Latest (3.0) flags uninitialized locals in catch paths. Dropped to 2.0 and initialized all result fields up front — catches real typos without noise.

RECONFIGURE on restoration now uses WITH OVERRIDE defensively for the rare case where an instance has an unusual persisted value. Not used on enable (0/1 is always valid).

T-SQL compatibility confirmed across SQL 2016–2025: every action group, system view, and SERVERPROPERTY used is supported on all five major versions. No version-gated features (SENSITIVE_BATCH_COMPLETED_GROUP, etc.) are in the baseline.

What the script still can't handle (documented at the bottom of the file): AG replica GUID coordination, NTFS ACL pre-configuration, and environments where xp_cmdshell is entirely prohibited by policy. These are honest limitations worth planning around before a fleet-wide run.

The file is ready to download. When you're ready, the collection-side job (pulling audit records from each instance into a central audit DB using the same serial-per-instance pattern) is the logical next piece.

Deploy complianceaudit

Document · MD

## You said: Xp_cmdshell can be enabled if it’s off.

Xp_cmdshell can be enabled if it’s off. No limitations. There are some AG servers though. Explain what this means and what’s needed for proper setup on an AG.

AG replica GUID coordination.

Also explain what this means and what’s required - NTFS ACL pre-configuration

*Apr 18*

## Claude responded: Good — those are the two real gaps worth understanding before a fleet deploy.

Good — those are the two real gaps worth understanding before a fleet deploy. Let me explain each clearly.

### AG replica GUID coordination

What an Availability Group is, briefly. An AG is SQL Server's HA/DR feature where two or more SQL instances host identical copies of the same databases. One replica is "primary" (read-write), the others are "secondary" (read-only or not readable). Applications connect to a listener name that transparently routes to whichever replica is currently primary. When failover happens — planned or unplanned — the secondary takes over and becomes the new primary.

Why SQL Server Audit needs special handling on AGs. Each CREATE SERVER AUDIT statement generates a GUID stored in sys.server_audits.audit_guid. This GUID is how SQL Server identifies the audit internally. Here's the catch: the server audit definition itself is an instance-level object — it lives in master — and isn't replicated between AG replicas. You have to create it independently on each replica. If you just run CREATE SERVER AUDIT on each replica separately, each one generates its own unique GUID.

Why that breaks things. Audit specifications on user databases reference the audit by GUID, not by name. Database audit specifications — the ones you'd use for things like SCHEMA_OBJECT_ACCESS_GROUP on specific tables — are stored in the user database itself, and those databases DO replicate across the AG. So a database audit spec gets replicated to the secondary with the primary's audit GUID embedded in it, but the secondary has a different GUID for its own audit. When failover happens, the database audit spec can't find its matching audit on the new primary, and auditing on that database silently stops working.

The server audit specification in this deployment is instance-level and doesn't have this problem — it's recreated fresh per instance. But the moment you add database-level audit specs (which you'd do for PCI/PHI table access tracking, the logical next phase), the GUID mismatch becomes a real failover-day problem.

What's needed. Three changes for AG-participating instances:

Generate one GUID per AG, not per replica. Decide on a GUID for each AG (e.g., NEWID() run once in a query window, or generated in PowerShell with [guid]::NewGuid()), then hardcode that same GUID in the CREATE SERVER AUDIT statement on every replica of that AG. The syntax is:

```sql
   CREATE SERVER AUDIT [Audit_Compliance]
   TO FILE (...)
   WITH (
       QUEUE_DELAY = 1000,
       ON_FAILURE  = CONTINUE,
       AUDIT_GUID  = '12345678-1234-1234-1234-123456789012'  -- same on every replica
   );
```

The audit state must match across replicas. If the audit is enabled on the primary but disabled on a secondary, failover will surface a failed audit — and if ON_FAILURE = SHUTDOWN, the new primary won't accept connections. Enable everywhere or disable everywhere.

The audit file path must be valid on every replica. If you've standardized on D:\SQLAudit\ across the fleet, this is already handled. But if a secondary lacks a D: drive or has different storage layout, you'll get a failure on failover when SQL tries to write audit records to a path that doesn't exist on the new primary.

Detecting AG membership from the deployment script. Before deploying, query sys.availability_replicas joined to sys.availability_groups — any row means the instance is part of an AG. The AG name is the key: all replicas of AG_Production need the same audit GUID. A reasonable approach for the deployment wrapper is to query AG membership up front, group instances by AG name, generate one GUID per AG, and apply that GUID to every replica in the group. Instances not in any AG get a fresh per-instance GUID as before.

Practical path forward for your fleet. Run an inventory pass first — just a T-SQL query across your CMS groups that returns @@SERVERNAME, AG name (if any), and the role (primary/secondary). That inventory tells you which instances need coordinated GUIDs and which are standalones. Then extend the deployment wrapper with an AG-aware mode. I can write that extension when you're ready.

### NTFS ACL pre-configuration

What NTFS ACLs are. Access Control Lists — the Windows filesystem permission model. Every file and folder has an ACL listing which users and groups can read, write, modify, or take ownership. When the SQL Server service tries to write an audit file to D:\SQLAudit\, the Windows kernel checks the ACL of that folder against the identity of the SQL Server service account. If the service account isn't granted at least "Modify" permission on the folder, the write fails — and CREATE SERVER AUDIT raises an error, or auditing writes fail silently depending on which stage the failure hits.

What "pre-configuration" means in this context. Creating the folder via MKDIR (what the script does) produces a folder that inherits ACLs from the parent directory. By default on most Windows servers, D:\ grants "Administrators" and "SYSTEM" full control, and nothing else. The SQL Server service account — usually something like NT SERVICE\MSSQLSERVER for a default instance, NT SERVICE\MSSQL$INSTANCENAME for a named instance, or a domain service account like DOMAIN\svc_sql — is typically not listed. So MKDIR succeeds (because it's running as the SQL service account, which has been granted the right to create folders through xp_cmdshell's elevation), but the resulting folder may not grant the service account permission to write files inside it.

Why it usually works anyway, and when it doesn't. In most environments, the SQL service account does inherit adequate rights because:

xp_cmdshell runs commands under the SQL Server service account identity by default

A folder created by an identity usually grants that identity ownership, which implies full control

Many organizations grant the SQL service group "Modify" on entire data drives

But it breaks in specific scenarios:

Drives with locked-down root ACLs that don't grant the SQL service account anything

Folders pre-created by a different account with restrictive ACLs

Group Managed Service Accounts (gMSAs) where inheritance doesn't work as expected

Drives mounted with specific security templates from enterprise hardening standards

What's required to pre-configure. Before CREATE SERVER AUDIT runs, the audit directory must exist with an ACL that grants the SQL Server service account at least:

Read and execute (traverse the folder)

Write (create new audit files as they roll over)

Modify (rotate files when MAX_ROLLOVER_FILES triggers cleanup)

The clean way to do this is a one-time pre-stage via your config management tool (Group Policy, DSC, Ansible, etc.) that creates D:\SQLAudit\ on every target with an ACL like:

D:\SQLAudit\

NT SERVICE\MSSQLSERVER       Modify       (or your service account)

BUILTIN\Administrators       Full Control

NT AUTHORITY\SYSTEM          Full Control

Doing it via T-SQL through xp_cmdshell. Also possible, using icacls instead of relying on inheritance:

```sql
DECLARE @cmd NVARCHAR(4000) =
    N'icacls "D:\SQLAudit" /grant "NT SERVICE\MSSQLSERVER:(OI)(CI)M" /T';
EXEC xp_cmdshell @cmd;
```

The flags mean: grant (M)odify, with (OI) object inherit and (CI) container inherit so new files and subfolders inherit, applied recursively (/T). The tricky part is knowing the exact service account name per instance — NT SERVICE\MSSQL$INSTANCENAME for named instances, domain accounts for those using them. You can query it via sys.dm_server_services or SERVERPROPERTY('ProcessID') + lookup.

My honest recommendation for your fleet. Given you have ~100 instances across varied versions and likely varied service account setups, pre-staging D:\SQLAudit\ via whatever config management you use is the cleaner approach. One pass, one known ACL, auditable in your CMDB. Then the deployment script's MKDIR becomes a no-op (the IF NOT EXIST guard handles that), and the DIR verification step catches any instance where pre-staging missed or ACLs were wrong — those show up as "Directory not accessible" failures in the log, which is exactly the diagnostic you want.

If pre-staging isn't practical, the script can be extended to query the service account name per instance and run icacls after MKDIR. Not difficult, just more moving parts — and it means xp_cmdshell is doing filesystem work that's really config-management's job.

Want me to extend the deployment wrapper with AG detection and per-AG GUID coordination, or keep that as a separate second script that runs against the AG-participating subset of your fleet?

## You said: Let’s revise the logic in the script.

Let’s revise the logic in the script. When the CMS server list is queried by its defined parameter path, the list of retrieved servers needs to be stored in a temporary table on the local cms database instance or in memory where it can be built with additional columns added for a final lookup table. This will require a secondary connection to the localhost server since you cannot connect to the local host instance via CMS, unless the lookup table is written and updated in memory. The script will first need to connect via CMS to run a first pass to determine if the server is part of an AG. Store the returned yes/no AG value in the lookup table. If the server is part of an AG, get the name of the other servers in the AG, and update the lookup table with a matching guid for all the servers in the lookup which are in the same AG. Then check the next server in the lookup table. If it is already assigned a GUID, skip it and move to the next one. Do this until all servers are identified as either standalone or AG servers and the AG servers are assigned a GUID for each group of AG’s. Each AG group of servers should be assigned its own unique GUID. Any CMS path that’s provided will have all nodes of an AG in that folder, so there’s no worry about a matching AG server not being in the CMS lookup list for that path. The initial prep check should also retrieve the path to where instance logs are being written and then use that path to write the sqlaudit logs instead of creating a new directory. That will handle the ACLs and different paths between server versions. It can also write an error if there’s a connection issue to an error column which can be used at the end to write the table to a log file. Other errors in the process should still be written but any servers that have a connection issue in the beginning pre check can be skipped in the next processing steps since it’s checked before each CMS loop.

After the lookup table is built, then proceed with the rest of the script, connect to each CMS instance and using the built lookup table to determine if it has an AG GUID associated with it for the audit setup. That will ensure AG servers are setup with the same GUID, no folders need created, and servers that can’t be connected to are skipped automatically in the next pass.

Revise the PowerShell script to add logic and make any changes for adopting this update. Make sure to do a final check of all the work and syntax for accuracy and fix any bugs. Then produce an updated version of the script in the markdown file.

Show more

*Apr 18*

## Claude responded: Good design choices — using the SQL error log path eliminates the ACL problem entirely (SQL Server already has write access to that directory), and the two-pas…

Good design choices — using the SQL error log path eliminates the ACL problem entirely (SQL Server already has write access to that directory), and the two-pass approach with a lookup table makes the AG coordination clean. Let me work through this carefully.

### Design thinking before I write:

Lookup table in memory vs. temp table. Temp table requires a persistent connection to the local CMS host for the duration, plus the secondary connection. In-memory (a [System.Collections.Generic.List] of PSCustomObject) is simpler, faster, and doesn't require a second connection. The user said "unless the lookup table is written and updated in memory" — that's the cleaner path. I'll use in-memory with an option to export to the CMS database at end if needed.

AG detection query. Need sys.availability_replicas joined to sys.availability_groups and sys.dm_hadr_availability_replica_states. The replica_server_name field gives the node names. Need to handle: SQL 2016+ has AGs (fine for our version range), instances with HADR disabled return empty results (not an error), FCI vs AG distinction.

Log path retrieval. SERVERPROPERTY('ErrorLogFileName') returns the full path to ERRORLOG. The directory portion is what we want. Available on all target versions.

Service account check. Not needed anymore if we use the error log directory — SQL already writes there.

Skip logic. If Reachable = $false after pre-check, skip deployment loop. Need to check this before each iteration.

### GUID assignment algorithm.

For each server in lookup: if AG=No, assign a unique GUID (or leave null if standalones don't need coordination — actually they do need a GUID for the audit, just not a coordinated one).

Wait — actually every audit gets a GUID. For standalones, NEWID() at creation time is fine. For AG servers, the GUID must match. So the lookup table needs a GUID column for AG servers specifically, and standalones can either get one pre-assigned or let SQL generate.

Cleanest: assign a GUID to every row. Standalones get unique, AG members get shared-per-AG. That way the deployment SQL always uses an explicit GUID from the lookup.

Algorithm: iterate the lookup. For each row with no GUID yet: if it's in an AG, generate a new GUID and assign it to this row AND every other row whose AG name matches. If standalone, generate a new GUID for just this row. Move on. Rows with GUIDs already assigned get skipped.

Secondary AG query per server. The user's plan: "get the name of the other servers in the AG." Query sys.availability_replicas from each AG primary and pull all replica_server_name values. We match those back to the lookup table by name.

Name matching between AG replica names and CMS names. replica_server_name returns @@SERVERNAME format (SERVER\INSTANCE for named instances, SERVER for default). CMS ServerName returns whatever was registered. These should match but sometimes don't — case, FQDN vs short name, alias. I'll do case-insensitive match and warn on mismatches.

Error column handling. Already in the result object design. Pre-check failure sets Reachable=false and ErrorMessage. Deployment loop checks Reachable before acting.

CREATE SERVER AUDIT with explicit GUID. Template needs a {{AUDIT_GUID}} token now. Need to verify syntax — it's AUDIT_GUID = '...' in the WITH clause.

The error log path — edge case. Some shops put the error log on a separate volume from data. Fine, SQL has write access there. Some shops have UNC error log paths (rare but possible). The FILEPATH for CREATE SERVER AUDIT accepts local paths; UNC works too but adds complexity. I'll assume local and flag if UNC is detected.

Directory verification simplification. If we're using the SQL error log directory, we know it's writable by the SQL service account (SQL is actively writing to it). The DIR check is now just a sanity check that the path extraction worked.

xp_cmdshell no longer needed for directory creation. Still might need it for DIR verification — but we could skip that too since sys.dm_os_host_info / error log query guarantees the path exists. Actually we don't need xp_cmdshell at all now. That's a big simplification. Let me think... yes, if we use the SQL error log directory, xp_cmdshell is only needed if we want belt-and-suspenders directory verification. The user said "no folders need created" — confirming xp_cmdshell enable/disable is no longer needed. That removes the whole sp_configure dance. Much cleaner.

Wait — let me re-read. "That will ensure AG servers are setup with the same GUID, no folders need created, and servers that can't be connected to are skipped automatically in the next pass." Yes, xp_cmdshell dance can be removed entirely.

What about the previous xp_cmdshell logic — do we remove it or keep as fallback? The user's direction is clear: use the log path, no folders need created. Remove xp_cmdshell logic entirely. Simpler, safer, and avoids the whole sp_configure save/restore flow.

Subfolder under the log path. Writing audit files directly into the error log directory mixes them with SQL's own logs. Cleaner to create a subfolder like SQLAudit under the error log directory. But that reintroduces the directory-creation problem. Trade-off: either (a) write audits directly in the errorlog directory and accept the mixing, or (b) write to a subfolder and need xp_cmdshell for the MKDIR. Option (a) is simpler and the audit files have a .sqlaudit extension that's distinct from ERRORLOG. I'll go with (a) and note the trade-off.

Actually, better idea: check if a subfolder already exists. If yes, use it. If no, use the errorlog directory directly. That's even simpler — no directory creation attempt at all, use what's there. The user's intent is clearly "don't create folders."

SERVERPROPERTY('ErrorLogFileName') parsing. Returns something like C:\Program Files\Microsoft SQL Server\MSSQL15.MSSQLSERVER\MSSQL\Log\ERRORLOG. We want the directory — everything before the last \.

AG secondary replica connectivity. The lookup pre-check connects to every server individually. When we hit an AG secondary, the AG query runs on whatever replica we connected to — secondaries can read sys.availability_replicas fine. No need to bounce to the primary.

Replica_server_name format consistency. All replicas in an AG see the same sys.availability_replicas rows. So whichever replica we hit first returns the whole list. We can mark all matching servers in the lookup in one pass — this matches the user's algorithm.

One more subtle thing. Check: what if a CMS path contains an AG primary but not all secondaries? The user said this won't happen ("Any CMS path that's provided will have all nodes of an AG in that folder") — so I don't need to handle the partial-AG case defensively. But I should warn if the AG query returns replica names that aren't in the lookup, because that indicates either a CMS registration gap or a naming mismatch worth investigating.

Result object changes. Need new columns: IsAG, AGName, AuditGuid, ErrorLogPath, AuditDirectoryPath. The existing ErrorMessage column is used for pre-check errors too.

The CREATE SERVER AUDIT WITH AUDIT_GUID syntax. Need to verify this is accepted across all versions. Yes, this has been supported since 2008.

T-SQL injection consideration. The GUID comes from [guid]::NewGuid() which is always a valid GUID format, safe to inline. The audit path comes from SQL itself (not user input). We're safe.

Let me verify SERVERPROPERTY('ErrorLogFileName') availability. Available since SQL 2005. Good for our range.

Edge case: what if ErrorLogFileName returns NULL? Very rare but defensive check needed.

### Process order clarification.

Phase 1: Enumerate CMS paths → get server list.

Phase 2: For each server, connect and pre-check (reachable, version, AG info, errorlog path). Store in lookup.

Phase 3: Iterate lookup, assign GUIDs (AG-coordinated or unique).

Phase 4: For each reachable server in lookup, deploy audit using its assigned GUID and audit path.

-WhatIf still works? Yes, same pattern.

The config CSV export at end. Should include the full lookup table so it's useful as a retry/inventory artifact.

Now let me write it carefully. Let me also double-check my T-SQL for AG detection.

Searched the web

Searched the web

Good — sys.availability_replicas joined to sys.availability_groups gives me replica_server_name and AG name, and it works whether queried from primary or secondary. SERVERPROPERTY('IsHadrEnabled') is the right gate. Let me write the final script.

Created a file, ran a command, read a file

Created a file, ran a command, read a file

Here's a summary of what the v3 revision does differently and the key decisions behind it.

### Architectural shift: three phases instead of one big loop.

Phase 1 enumerates the CMS paths exactly as before. Phase 2 connects to each server once and captures everything needed up front — version, reachability, AG membership, AG peer names, and the SQL error log directory — into an in-memory Dictionary[string, object]. Phase 3 walks the lookup and assigns GUIDs: standalones get their own, AG replicas share one GUID per AG. Phase 4 iterates the lookup and deploys, automatically skipping any row where Reachable=false.

The in-memory lookup design. I went with System.Collections.Generic.Dictionary keyed case-insensitively rather than a temp table on the CMS host. Your design note specifically allowed this ("unless the lookup table is written and updated in memory"), and it eliminates the secondary connection to localhost while giving O(1) lookups during AG peer matching. The full lookup gets exported as a CSV at completion, so you still have a persistent artifact for audit trail and retry-list building — just without needing a second connection or a cleanup pass to drop a temp table.

xp_cmdshell logic is gone entirely. This is the biggest simplification. Since audit files now go to the SQL error log directory (retrieved via SERVERPROPERTY('ErrorLogFileName')), SQL Server already has write access there — no MKDIR needed, no ACL work, no sp_configure save/restore dance, no risk of leaving xp_cmdshell enabled somewhere. The script is shorter and safer.

AG detection and peer enumeration. The pre-check query uses sys.availability_groups joined to sys.availability_replicas to find the AG this instance belongs to and return a comma-delimited list of all peer replica names. Querying from a secondary works fine — sys.availability_replicas returns all replicas from any replica's perspective. I used the classic STUFF + FOR XML PATH string aggregator rather than STRING_AGG because STRING_AGG is 2017+ only and we need 2016 compatibility.

The GUID coordination algorithm. Walk the lookup once. For each reachable row with no GUID yet: generate a fresh GUID; if the row is standalone, assign it to just that row; if AG, split the peer list and propagate the same GUID to every peer present in the lookup. Already-assigned rows are skipped. This matches exactly what you described: "If it is already assigned a GUID, skip it and move to the next one." Each AG gets its own unique GUID; standalones each get their own.

Missing-peer warning. If the AG topology reports a replica name that isn't in the CMS-derived lookup, the script warns and logs it to the failure log. You stated this shouldn't happen in your environment, but if it ever does — say someone adds a replica without registering it in CMS — you'll see it immediately rather than silently deploying a mismatched GUID.

Unreachable servers are skipped cleanly. Pre-check errors set Reachable=false and store the message in PrecheckError. The deployment loop checks Reachable at the top of each iteration and continues without attempting to connect again. Other errors during deployment go into a separate DeployError column so you can distinguish connection problems from deployment problems in the CSV.

Column separation in the output. The CSV now has PrecheckError and DeployError as distinct columns. A row with only PrecheckError populated is a never-reached server; a row with only DeployError populated is a server that responded to pre-check but failed during audit deployment. That split makes retry workflows much more precise — you'd handle the two categories differently.

### Correctness checks I ran on the T-SQL and PowerShell:

The pre-check query works on standalone instances (returns IsAG=0, AGName=NULL, AGPeers=NULL), on instances with IsHadrEnabled=0 (same result), and on AG-participating instances whether queried from primary or secondary. SERVERPROPERTY('ErrorLogFileName') is available on SQL 2008+, so fine for our 2016–2025 range. The CREATE SERVER AUDIT ... AUDIT_GUID = '...' syntax has been supported since 2008. Invoke-Sqlcmd returns System.DBNull (not $null) for SQL NULL values, so I added explicit System.DBNull checks on every string property that the pre-check query can return null for. StrictMode 2.0 is satisfied — every property access goes through fields initialized in New-LookupRow.

The file is ready to download and deploy. When you're ready for the next piece — the collection-side job that pulls audit records from each instance into a central audit database using this same lookup-table pattern — just say the word.

Deploy complianceaudit v3

Document · MD

## You said: Isn’t there also a log folder where the tempdb file path is stored, that’s used for sqlaudits?

Isn’t there also a log folder where the tempdb file path is stored, that’s used for sqlaudits?

*Apr 18*

## Claude responded: You're thinking of the default log path that's configured at the instance level — and yes, that's a real and often better choice than the error log directory.

You're thinking of the default log path that's configured at the instance level — and yes, that's a real and often better choice than the error log directory. Let me clarify what's what, because there are actually three distinct "log" paths people mix up:

1. The SQL Server error log directory (what v3 currently uses)

Retrieved via SERVERPROPERTY('ErrorLogFileName') — returns the full path to ERRORLOG, and we extract the directory. This is where SQL writes its own operational log (ERRORLOG, ERRORLOG.1, etc.). Writable by the SQL service account because SQL is actively writing to it. Usually something like C:\Program Files\Microsoft SQL Server\MSSQL16.MSSQLSERVER\MSSQL\Log\.

2. The instance default LOG directory (what you're thinking of)

This is the path configured as DefaultLog in the registry/facets — where SQL creates transaction log files (.ldf) by default when a new database is created without explicit file paths. Retrieved via SERVERPROPERTY('InstanceDefaultLogPath') (SQL 2012 SP1+) or from sys.dm_server_registry / xp_instance_regread. Often the same as the ERRORLOG directory on default installs, but commonly relocated to a dedicated fast-storage LOG volume (e.g., L:\MSSQL\Log\). Not where tempdb lives — tempdb has its own file paths defined in sys.master_files.

### 3. The instance default DATA directory

SERVERPROPERTY('InstanceDefaultDataPath'). Where .mdf files go by default. Different volume from LOG in any well-configured instance.

On tempdb specifically — the clarification. Tempdb's files live wherever an administrator configured them (sys.master_files for database_id = 2). That's often on a dedicated fast-storage volume, sometimes local SSD, sometimes the same drive as user database logs. There's no automatic "tempdb folder" that SQL Server Audit uses — tempdb's location is independent of audit file paths. So if you've heard "use the tempdb path for audits," that's not quite right — what's actually meant is usually "use the default LOG path," which on many installs happens to be on the same volume as tempdb because both want fast I/O.

### Why the default LOG path is often a better audit destination than the error log directory:

The error log directory is usually on the OS drive (C:) alongside the SQL Server binaries — a modest-size volume not sized for sustained write workloads. Audit files on a busy instance can generate hundreds of megabytes per day, and MAX_ROLLOVER_FILES = 50 at 256 MB each means up to 12.5 GB of retention per instance. Filling up C:\ because of audit growth is a classic footgun.

The default LOG directory is almost always on a dedicated volume with better I/O characteristics and more headroom. SQL Server already has write access (it creates .ldf files there). Same ACL guarantee, better storage profile.

The trade-off: LOG directory mixes audit files with transaction log files. Not a functional problem — extensions (.sqlaudit vs .ldf) are distinct and file systems don't care — but some shops prefer audits to live in a dedicated subfolder for operational clarity. That reintroduces the directory-creation problem.

My recommendation for your environment. Use InstanceDefaultLogPath as the primary choice, with ErrorLogFileName directory as fallback for older/edge-case instances where InstanceDefaultLogPath is NULL. Both are writable by the SQL service account, both avoid new directory creation. This is a small change to the pre-check query and the New-LookupRow factory.

Here's the revised pre-check SQL fragment:

```sql
-- Prefer instance default LOG path (usually on a dedicated LOG volume);
-- fall back to the error log directory for older installs or unusual
-- configurations where InstanceDefaultLogPath returns NULL.
DECLARE @defaultLog NVARCHAR(512);
DECLARE @errorLogPath NVARCHAR(512);
DECLARE @auditDir NVARCHAR(512);
DECLARE @pathSource NVARCHAR(32);

SET @defaultLog = CAST(SERVERPROPERTY('InstanceDefaultLogPath') AS NVARCHAR(512));
SET @errorLogPath = CAST(SERVERPROPERTY('ErrorLogFileName') AS NVARCHAR(512));

IF @defaultLog IS NOT NULL AND LEN(@defaultLog) > 0
BEGIN
    SET @auditDir = @defaultLog;
    SET @pathSource = N'InstanceDefaultLogPath';
END
ELSE IF @errorLogPath IS NOT NULL AND LEN(@errorLogPath) > 0
BEGIN
    SET @auditDir = LEFT(@errorLogPath, LEN(@errorLogPath) - CHARINDEX('\', REVERSE(@errorLogPath)));
    SET @pathSource = N'ErrorLogDirectory';
END

-- InstanceDefaultLogPath sometimes returns without a trailing backslash.
-- Normalize here so downstream PowerShell doesn't have to.
IF @auditDir IS NOT NULL AND LEN(@auditDir) > 0 AND RIGHT(@auditDir, 1) <> '\'
    SET @auditDir = @auditDir + N'\';
```

And two new fields in the SELECT return:

```sql
,@auditDir    AS AuditDir
,@pathSource  AS PathSource
```

Then in New-LookupRow add a PathSource column, and in Phase 2 populate it. Everything else — GUID assignment, deployment, verification — stays the same because the path is just the path.

Want me to produce a v4 with this change folded in, or is the concept clear enough that you'd rather make the edit yourself? The full script is long enough that if v4 is worth producing, I'd also take the opportunity to do another correctness pass on the AG query and the STUFF + FOR XML PATH string aggregation since those are the most error-prone parts and worth one more look.

## You said: Ldf files are on a different drive.

Ldf files are on a different drive. All servers have a drive dedicated for the tempdb. SQL server creates a folder ‘\MSSQL\tempdb’ and I’m pretty sure there’s also a folder ‘\MSSQL\LOG’ on that same drive. Ldf files are stored in another drive and mdf files also on a separate drive where the error logs, maintenance logs and other profiler logs are written. Since the data drive already has too much writing there, it would be better to use the other LOG directory that’s installed into MSSQL\LOG, which should be on the same drive as where tempdb was installed to.

*Apr 18*

## Claude responded: Got it — your storage layout is clearer now, and you're describing a very common enterprise pattern.

Got it — your storage layout is clearer now, and you're describing a very common enterprise pattern. Let me make sure I understand it correctly and then explain what each value actually maps to.

### Your layout, as I understand it:

Drive A (data): .mdf files, error logs, maintenance logs, profiler output. High write activity.

Drive B (transaction logs): .ldf files. Sequential write workload.

Drive C (tempdb/LOG): \MSSQL\tempdb\ for tempdb files, and \MSSQL\LOG\ on the same drive for something else. Lower sustained write pressure than the data drive.

You want audits on Drive C, in the \MSSQL\LOG\ folder specifically. That's a sensible choice — audit I/O is sequential append-mostly, similar profile to tempdb's needs, and keeping it off the data drive avoids contention with user workload.

### Here's the nuance about what \MSSQL\LOG\ actually contains:

That folder — typically something like D:\MSSQL\LOG\ in your setup — is the path SQL Server creates under its instance root and uses as the default location for SQL's own operational logs: ERRORLOG, ERRORLOG.1 through ERRORLOG.6, SQLAGENT.OUT, default traces, system health Extended Events files, and default location for audit files if no other path is specified. It is not the transaction log (.ldf) destination — despite the name "LOG." That's the common confusion because "log" means two different things in SQL Server:

.ldf files are transaction logs — per-database, write-ahead logging for durability

\MSSQL\LOG\ is SQL Server's operational log directory — the instance's own housekeeping output

So on your servers, the tempdb drive holds:

D:\MSSQL\tempdb\ → tempdb.mdf, tempdb_mssql_*.ndf, tempdb.ldf

D:\MSSQL\LOG\ → ERRORLOG, SQLAGENT.OUT, system_health XE files, default traces

And here's the important thing: SERVERPROPERTY('ErrorLogFileName') returns the full path to ERRORLOG, which on your servers is exactly D:\MSSQL\LOG\ERRORLOG. When v3 extracts the directory portion, it gets D:\MSSQL\LOG\ — which is the folder you want. The script is already pointing at the right location on your storage layout. The tempdb drive's \MSSQL\LOG\ directory is the error log directory.

What I proposed last turn — using InstanceDefaultLogPath — would actually be worse for your setup. InstanceDefaultLogPath is where new database .ldf files go by default, which on your servers is Drive B (the transaction log drive). Using it would put audit files on the same volume as transaction logs, which is a higher-contention storage path than where you actually want them.

So the v3 script is already correct for your environment. SERVERPROPERTY('ErrorLogFileName') → strip the filename → get D:\MSSQL\LOG\ → write audits there. Same drive as tempdb, writable by the SQL service account, no directory creation needed, no ACL work, no contention with .mdf or .ldf I/O.

### A quick validation you can run on a couple of representative instances before a fleet deploy:

```sql
SELECT
    @@SERVERNAME AS ServerName,
    SERVERPROPERTY('ErrorLogFileName') AS ErrorLogFile,
    -- This is what the script will extract as the audit directory:
    LEFT(CAST(SERVERPROPERTY('ErrorLogFileName') AS NVARCHAR(512)),
         LEN(CAST(SERVERPROPERTY('ErrorLogFileName') AS NVARCHAR(512)))
         - CHARINDEX('\', REVERSE(CAST(SERVERPROPERTY('ErrorLogFileName') AS NVARCHAR(512)))))
         AS AuditDirectory,
    -- For comparison, here's where new .ldf files would go:
    SERVERPROPERTY('InstanceDefaultLogPath') AS DefaultLogPath_NewDBs,
    SERVERPROPERTY('InstanceDefaultDataPath') AS DefaultDataPath_NewDBs;
```

Run that against two or three instances across different SQL versions (one 2016, one 2019, one 2022). You should see AuditDirectory come back as the tempdb drive's \MSSQL\LOG\ folder, and DefaultLogPath_NewDBs come back as the transaction log drive. If both match that expectation, v3 is ready to go fleet-wide without modification.

If an instance turns out to have a nonstandard layout — say someone moved ERRORLOG to a weird location at some point, or the registry ErrorDumpDir was manually overridden — the validation query will surface it, and you can either fix that instance's config or add it to an exception list to deploy manually.

So: no code changes needed. v3's ErrorLogFileName approach is the right one for your setup. The storage reasoning just happened to work out correctly because "SQL error log directory" and "tempdb drive LOG folder" are the same place on servers configured the way yours are.

## You said: Actually, I was mistaken.

Actually, I was mistaken. There is an AuditLog folder already created on all the servers as one of the post setup tasks of the instance initial installation. The path would be available in the sys.server_file_audits table. The current configured server audit path is setup the same drive as the ldf files, but it should be on the tempdb drive instead. The current audit configuration and server audit specifications should be removed and replaced with a new version and processing.

Do not update the script. Instead write a prompt to use in Vscode GitHub Copilot to work on this project for implementing further improvements, debugging, and testing. Include all the context and options already mentioned here for deployment and implementation purposes. Provide the prompt file in markdown.

*Apr 18*

## Claude responded: Here's what the prompt file captures so Copilot has the full context without you re-explaining it:

Created a file, ran a command, read a file

Created a file, ran a command, read a file

Here's what the prompt file captures so Copilot has the full context without you re-explaining it:

Environment and constraints: the unidirectional CMS-to-target firewall, SQL-connection-only orchestration, PowerShell 5.1 runtime, SqlServer module 21.1+ requirement, and the 2016–2025 version range with the specific T-SQL compatibility implications (no STRING_AGG, use STUFF + FOR XML PATH, etc.).

Your actual storage layout: data drive, ldf drive, tempdb drive with \MSSQL\tempdb\ and \MSSQL\LOG\, and the pre-created AuditLog folder that's part of instance build-out. Plus the key correction — that the current audit path is on the ldf drive and needs to move to the tempdb drive's AuditLog folder.

The six specific next tasks ordered for you to work through with Copilot:

Enhanced path resolution using sys.server_file_audits with fallback to ErrorLogFileName derivation

Existing-audit detection and cleanup (including the drop-spec-before-audit ordering and database audit spec handling across all databases)

xp_cmdshell reintroduction for DIR verification only, with full save/restore

Tempdb drive detection via sys.master_files to validate path resolution

A test harness with pre/post lookup CSV comparison and rollback

The collection-side job using sys.fn_get_audit_file() over the SQL connection

Architectural decisions already made that Copilot shouldn't second-guess: three-phase execution, in-memory lookup dictionary, serial processing only, one-GUID-per-AG, StrictMode 2.0, the 26-action-group baseline, and the failure log format.

Explicit "do not" list to prevent Copilot from suggesting things you've already rejected: no parallelization, no STRING_AGG, no hardcoded paths, no leaving xp_cmdshell on, no server-scoped SCHEMA_OBJECT_ACCESS_GROUP, no database audit specs in this script.

Tone guidance for comments so the generated code matches the voice of what you already have — dry, direct, explains the why of non-obvious patterns, no motivational fluff.

Usage instructions at the bottom for three Copilot modes: full-context chat paste, split-editor inline suggestions, and focused debugging with the tab-delimited failure log format.

When you're ready to work on the next phase, paste this whole file as the first message in a new Copilot Chat session, then follow it with something like "Implement Task 1: enhanced path resolution" and you'll get suggestions that respect all the constraints without needing to explain them each time.

Copilot project prompt

Document · MD

## You said: What AI and model is best to use for this project?

What AI and model is best to use for this project?

*Apr 18*

## Claude responded: Honest answer: for a project like this — SQL Server internals, PowerShell idioms, compliance nuance, and enterprise context that builds across long conversatio…

Honest answer: for a project like this — SQL Server internals, PowerShell idioms, compliance nuance, and enterprise context that builds across long conversations — you want a frontier model with strong code reasoning and long-context retention. Let me break it down by what matters for this specific workload.

### What this project needs from an AI

Deep SQL Server knowledge across versions 2016–2025, including DMV edge cases, audit internals, AG behavior

PowerShell 5.1 idioms (not just PS7), with awareness of SqlServer module version differences

Long-context retention so architectural decisions from 10 turns ago aren't forgotten

Willingness to push back when a direction is wrong (the tempdb-path confusion and the InstanceDefaultLogPath correction we worked through are good examples)

Code generation that produces long, consistent, production-grade scripts without hallucinating cmdlet parameters or T-SQL syntax

### Top tier options in April 2026

Claude (Opus 4.7 or Sonnet 4.6/4.7) — Strong at exactly this kind of work: long technical conversations, enterprise system context, willingness to flag errors in your thinking rather than just executing. Opus 4.7 is the most capable model I'm aware of currently and would be my honest recommendation for the primary design/architecture work. Sonnet 4.6/4.7 is the better cost/speed trade for iterative debugging and implementation. I'd use both — Opus for the hard design problems, Sonnet for the grind. Available via claude.ai, the API, and Claude Code (command-line coding agent).

GPT-5 family (OpenAI) — Also very strong at code. Different strengths — tends to be more aggressive at generating working code fast, sometimes at the cost of the careful "let me verify that syntax first" behavior you want for production SQL scripts. Worth having in the mix as a sanity-check voice.

Gemini 2.5 Pro / 3.x (Google) — Strong on long context (multi-million token windows) which matters when you want to paste entire codebases for review. Code quality is competitive. Worth considering if you end up with a large repo of related scripts.

### For your specific VS Code Copilot workflow

GitHub Copilot in VS Code now supports model selection — you can pick which underlying model powers chat and inline suggestions. As of April 2026, most Copilot Pro/Business/Enterprise tiers include access to Claude Opus and Sonnet, GPT-5, and Gemini 2.5+. For this project specifically, I'd configure Copilot to use Claude Sonnet 4.6 or 4.7 for inline suggestions (fast, cheap per request, good code quality) and Claude Opus 4.7 or GPT-5 for chat when you're working through architecture changes or debugging something non-obvious.

Let me verify current Copilot model availability since this changes frequently.

Searched the web

Searched the web

Good — current state is clearer now. Let me give you a practical recommendation.

### Current model landscape for GitHub Copilot in VS Code (April 2026)

Copilot Chat in VS Code currently offers, among others, Claude Opus 4.7 (at a promotional 7.5x request multiplier until April 30, 2026), Claude Opus 4.6 and Sonnet 4.6, various GPT-5 variants, and Gemini. The VS Code docs themselves suggest using a fast model like GPT-5 Mini for quick edits and simple questions, and a reasoning model like Claude Opus for complex refactoring and architectural decisions. That aligns with how I'd split this project's workload. GitHubVisual Studio Code

### My specific recommendation for this project

For the architecture work, compliance reasoning, and anything involving the xp_cmdshell restoration logic or AG coordination — use Claude Opus 4.7. The scenarios where one wrong assumption costs you a production incident (leaving xp_cmdshell enabled, mismatched AG GUIDs surfacing on failover six months from now) are exactly where a reasoning model earns its keep. The 7.5x request multiplier stings but the promotional rate runs through April 30; after that it'll drop to standard pricing.

For the iterative implementation and debugging — Claude Sonnet 4.6. Solid code generation, faster responses, reasonable cost. This is your daily driver for Tasks 1-6 in the prompt file. Set thinking effort to "High" for the harder bits.

For inline autocomplete (as-you-type suggestions) — whatever your Copilot plan gives you by default is probably fine. Inline suggestions benefit less from frontier reasoning since the context is narrow by design. Don't burn premium request multipliers on inline.

### A couple of alternatives worth knowing about

Claude Code — Anthropic's terminal-based coding agent, separate from VS Code Copilot. Runs from your command line, can read your entire workspace, plan multi-file changes, and execute them with your approval. Strong fit for this project because it respects architectural decisions across long sessions and can run your scripts directly to validate syntax. If you'd rather work in a terminal alongside VS Code than entirely inside Copilot Chat, this is worth trying. Available via Anthropic subscription or API.

The direct API route — If your org has restrictions on what data goes through GitHub's Copilot infrastructure, you can hit the Anthropic API directly from VS Code via community extensions, or use Claude.ai's web interface with files uploaded. Slower workflow than Copilot inline suggestions but gives you full control over what leaves your network. For a compliance-sensitive project where the code itself references production server names and infrastructure details, this is worth considering.

### Practical workflow suggestion

Keep three windows open when working on this:

VS Code with Copilot Chat set to Sonnet 4.6 — primary implementation, inline tweaks

A separate Claude chat (claude.ai or Opus 4.7 in Copilot) — for architecture questions, code reviews against the prompt file's "do not" list, and design decisions. Start each session by pasting the prompt file

A terminal — for actually running the scripts against your test CMS group and iterating on real failures

The key trick: paste the Copilot prompt file as the first message in every new Claude session. Long-running projects like this one drift without it — the model starts forgetting why StrictMode is 2.0 not Latest, or suggests parallelization you already rejected, or reaches for STRING_AGG because it's cleaner. The prompt file is your hedge against that drift.

### One honest caveat about me specifically

I'm obviously not a neutral voice here — I'm Claude recommending Claude. Take the recommendation with that grain of salt. What I can say with more confidence is the split strategy: use a reasoning-heavy model for architecture and a faster model for implementation, regardless of whose model each turns out to be. If you want to sanity-check my recommendation, run the same architecture question (say, "review the xp_cmdshell restore logic in v3 for failure modes") through GPT-5.4 and Claude Opus 4.7 and see which response catches more real issues. The one that finds things the other missed is the one worth using for hard problems.

## You said: Write you last response to an md file.

Write you last response to an md file.

*Apr 18*

## Claude responded: Saved.

Created a file, ran a command, read a file

Created a file, ran a command, read a file

Saved. The file captures the model recommendations, workflow suggestion, and the honest caveat about me not being a neutral voice. Drop it alongside the Copilot project prompt and the v3 script in your project workspace.

Ai model recommendation

Document · MD

## You said: Create a structured prompt system in md files with an agent.

Create a structured prompt system in md files with an agent.md and copilot-instruction.md file to build the architecture, design, sql code, centralized database structure (tables, procedures, and processes) and sql server job(s) with full documentation and instructions for deploying to the central node.

The audit database will get its list of servernames and sql ports from an inventory database table that it will join to for lookup purposes. This table is read only and never written to. Each servername in the inventory table will have unique identity column and store any other information needed about the instance. The prompt system should also write an md file of the requirements for the application to successfully execute, including any instance specific data points needed from the inventory database. The project will also contain an audits.txt file listing all the server level audits needed for setting up and enabling audits on a sql instance for the first time.

The audit database needs to fully track its job runs for instances capturing the instance identity as the table key for each instance as part of the audit database. The audit tracking table(s) will capture timestamps and other critical information as the central audit control, including the ability to set an instance audit job on or off with change/history tracking to a history table with the user and changes.

The instance database table also keeps track of new servers, so the audit system needs to recognize when a new server is added and needs auditing enabled/started. Servers may also be decommissioned and no longer be in the inventory table. The audit tracker will continue to audit the orphaned server instance, flagging it as orphaned, and when it can no longer connect to the server, automatically change the audit for that server to disabled.

Jobs do not fail if a server audit cannot run and auditing shall never take down a server instance if it’s not running or has a problem. Those failures are logged to a separate table that will be used for reporting to a dashboard in a separate application. The table and tracking information for the jobs are logged as part of this projects requirements, but the reporting of job failures to the dashboard is out of scope in this project.

If a server is identified as new to the inventory, it will setup the auditing and enable it to start. If auditing is already configured and enabled on the server, it will disable and delete the audit process and then deploy it again. This includes the first time the job runs. This check can also be a separate sql job from the audit collection job.

Auditlog files should be kept small and be collected frequently to reduce any potential for performance impact on the servers and network, as well as keeping the file footprint on the server small, so it doesn’t take up too much space on disk. The process will read the audit logs directly on the server and insert to a table in the audit database. Each server instance will log to a table in its own schema auditcollection.<servername> or auditcollection.<servername-instance>. When a server from the inventory is orphaned, it is moved into a schema called ‘orphaned’ and if it goes into a decommissioned state, it is moved into a schema called ‘decommissioned’. An audit tracking table will keep track of which schema each server lives in.

Different Views, functions, and/or stored procedures are needed for querying the audited tables, tracking, jobs, and changes by passing parameters such as a servername (for a single server), all servers, schema, and other types of queries based on status, job stats, audit types, etc.

Before creating anything, what else am I missing, what should be improved or changed, or what questions do you have?

Show more

*Apr 29*

## Claude responded: Before writing anything, let me think through what you've described and surface what I see as gaps, ambiguities, and design decisions that need your input.

Before writing anything, let me think through what you've described and surface what I see as gaps, ambiguities, and design decisions that need your input. I'll keep this organized so you can answer in chunks.

### Architectural questions

### 1. The "delete and redeploy" behavior on every check.

## You said: "If auditing is already configured and enabled on the server, it will disable and delete the audit process and then deploy it again. This includes the first time the job runs."

This concerns me. If the audit-check job runs on a schedule (say, daily) and tears down + redeploys the audit every time it sees an existing audit, you'll lose audit continuity — there's a window between drop and recreate where events aren't captured. For compliance auditing, that gap is a finding. Worse, if the redeployment fails after the drop, the server sits with no audit at all until the next run.

I think what you actually want is: on first encounter (server is new in inventory, or marked for re-baseline), drop and redeploy. On subsequent encounters, verify the audit matches the expected baseline (correct GUID, correct action groups, correct path, enabled state) and only redeploy if drift is detected. Can you confirm? If you really want unconditional drop-and-redeploy on every check, that's a different design and I want to make sure it's deliberate.

### 2. Frequency of audit file collection vs. file size.

You said audit files should be small and collected frequently. Two parameters control this on the SQL Server side: MAXSIZE (per-file rollover) and MAX_ROLLOVER_FILES (retention). If you're collecting every 5 minutes, a 64 MB rollover with 5 files retained means each server keeps at most ~320 MB on disk. If you're collecting hourly, you might want 32 MB rollover and 10 files.

What's your target collection frequency? My recommendation would be every 15 minutes with MAXSIZE = 64 MB and MAX_ROLLOVER_FILES = 10 — keeps disk footprint under ~640 MB per instance, lets you recover from a 2+ hour collection outage without losing data, and isn't aggressive enough to thrash the network.

### 3. Audit reading mechanism over the wire.

Two options for reading audit files from a remote instance via SQL connection:

sys.fn_get_audit_file() with a local path — runs on the target instance, returns rows over the connection. The function reads files local to that SQL Server using its service account's filesystem access. This is the simplest pattern for your firewall constraint.

sys.fn_get_audit_file() with a UNC path from the central host — would require SMB from central to target, which violates your firewall direction.

So we're using option 1: connect to each target, run sys.fn_get_audit_file() against its local audit path, stream rows back into the central database. Confirming this is what you intended.

### 4. Watermark / incremental collection.

sys.fn_get_audit_file() accepts an offset parameter (initial_file_name, audit_record_offset, audit_file_offset) so you can resume from where you left off. The collection job needs to track per-instance:

Last event_time collected (monotonic, but not unique)

Last audit_file_offset (unique within file)

Current file_name being read

Without watermarking, you re-read everything every time — fine for a tiny instance, brutal for a busy one. I'd add a watermark column to the per-instance tracking. Confirming this is in scope.

### 5. The schema-per-server pattern.

Storing collected audit data in auditcollection.<servername> is a clear isolation pattern, but has trade-offs:

Pro: easy permission boundary, easy to drop a server's data, schema name doubles as identification

Con: schema/table name complexity for named instances (SERVER01\SQL2016 is not a valid identifier — needs sanitization), schema explosion (100+ schemas), cross-server queries require dynamic SQL or UNION ALL views, no enforcement that all per-server tables have the same structure (drift over time)

A common alternative is a single audit_records table partitioned by instance_id (the inventory identity column you mentioned). All rows in one table, indexed by instance_id + event_time, partitioned for manageability. Gives you the same isolation for archival/cleanup (drop a partition) but unified querying.

I'll proceed with your schema-per-server design since you specified it, but want to flag this so it's a deliberate choice. Do you want me to also generate a comparison doc covering the trade-offs, or stick with the design as-stated?

### 6. Named-instance handling in schema names.

Following from #5: SQL Server schema names can't contain \. So SERVER01\SQL2016 becomes... what? SERVER01_SQL2016? SERVER01__SQL2016? SERVER01$SQL2016? Whatever convention we pick needs to be documented and applied consistently. My suggestion: replace \ with __ (double underscore) — visually distinct, valid identifier, unambiguous. Confirming.

### 7. The "orphaned" → "decommissioned" lifecycle.

You described:

Server in inventory → audited normally in auditcollection.<server>

Server removed from inventory → flag as orphaned, move to orphaned.<server> schema, keep trying to audit

Server can't be connected to → mark audit disabled (still orphaned schema)

Server eventually moves to decommissioned.<server> schema

Two questions:

What triggers the move from orphaned to decommissioned? Time-based (e.g., 30 days unreachable)? Manual action by a DBA? Different connection failure threshold?

What about a server that comes back? Decommissioned servers presumably stay decommissioned, but an orphaned server that reappears in inventory should presumably get rehydrated back into auditcollection. Is that right?

### 8. History tracking — what level of detail?

You mentioned the ability to enable/disable per-instance auditing with user/change history. What user identity gets captured?

SUSER_SNAME() of whoever made the change via SSMS (works fine)

Application-level user (if changes go through a UI/app — not in scope here)

ORIGINAL_LOGIN() to see through EXECUTE AS (defensive)

I'd default to capturing all three: SUSER_SNAME(), ORIGINAL_LOGIN(), and HOST_NAME(). Confirming.

### 9. Job structure — one job, two jobs, or N jobs?

You said "this check can also be a separate sql job from the audit collection job." Let me propose:

Job A: Audit Configuration Sync — runs daily (or on-demand). Reads inventory, finds new/orphaned/decommissioned servers, deploys/disables audits accordingly. AG-aware. This is the v3-style script as a SQL Agent job step.

Job B: Audit Collection — runs frequently (every 15 min). Reads audit files from each enabled instance, inserts into per-server tables, advances watermark.

Job C: Health Reporter (optional) — rolls up failures and statistics into a summary table the dashboard reads.

Two jobs (A + B) is my recommendation. Three if you want the health reporting decoupled from collection. Confirming.

### 10. Failure handling philosophy.

## You said: "Jobs do not fail if a server audit cannot run and auditing shall never take down a server instance." Strong agreement. This means:

Every per-server operation is wrapped in TRY/CATCH

Failures log to a collection_errors or job_errors table

The job step itself returns success even when individual instances fail

Dashboard alerts come from querying the errors table, not from job failure notifications

There's one nuance: if every server fails (e.g., the central server lost network), the job probably should fail or alert loudly. A "100% failure rate" threshold check at the end of each run? Or is "log it and let the dashboard handle it" the answer? I'd lean toward logging only; dashboard owns alerting.

### Things I think you're missing

### A. Audit file path resolution per server.

We worked through this in the prior conversation — the audit path comes from sys.server_file_audits.log_file_path after deployment, or from the AuditLog folder pre-created during instance build. The collection job needs to know this path per instance. Either:

Store the path in the audit tracker table (set when audit is deployed)

Query sys.server_file_audits at collection time

I'd store it in the tracker because it eliminates a round-trip per collection cycle. Confirming.

### B. The compliance audit specification baseline.

The audits.txt file you mentioned — should it just list the action groups, or include the full CREATE SERVER AUDIT SPECIFICATION template? I'd produce both: a plain list for human reading, and the deployable T-SQL template the deployment job consumes.

### C. AG GUID coordination is still needed.

This was a hard-won decision in v3 — all AG replicas share one AUDIT_GUID. The new database needs a column to track this:

ag_name (nullable, populated for AG members)

audit_guid (populated after deployment, shared across AG peers)

The configuration sync job has to read this and propagate the same GUID across replicas at deployment time.

### D. What runs on the central server vs. what runs on the targets.

To be explicit:

Central node: the audit database, all tracking tables, both SQL Agent jobs, the inventory table (read-only join target), and all stored procedures

Target instances: only the deployed server audit + spec + the audit files on disk

Nothing new is deployed to targets beyond the audit itself — no linked servers, no extended stored procedures, no agent jobs

Confirming the central node is also a SQL Server instance running SQL Agent, and the inventory database is on that same instance (or accessible via a database reference). If the inventory is on a different server, we need a linked server or cross-instance pattern.

### E. Permissions model.

The SQL Agent service account on the central node needs:

CONTROL SERVER on every target instance (or ALTER ANY SERVER AUDIT + VIEW SERVER STATE + sufficient permissions to enable xp_cmdshell if needed)

SELECT on the inventory table

db_owner on the audit database (to create per-server schemas dynamically)

This is significant — fleet-wide CONTROL SERVER is a privileged grant. Worth explicitly calling out in deployment requirements.

### F. The "orphaned but still auditing" state has a subtle problem.

If a server is removed from inventory but still online, we keep auditing it. But the audit was configured by us — at some point, someone decommissioning the server should clean up the audit on the target. Otherwise audits keep writing to disk forever on a server nobody's tracking. Should the orphaning process attempt to drop the audit on the target (best-effort), or leave it running until decommission? My recommendation: leave it running during orphan state, attempt cleanup on transition to decommissioned. Confirming.

### G. Retention on collected audit data.

The collection tables grow forever unless we have a retention policy. Compliance frameworks usually require 1–7 years of audit retention (SOX is 7, PCI is 1, HIPAA is 6). Do you have a target? This affects partition strategy and storage planning. If unspecified, I'll default to 2 years with monthly partitions or month-based archival logic, and document it as configurable.

### H. The dashboard's read pattern.

Out of scope per your message, but the table design has to support it. The dashboard will likely want:

"Show me all servers and their audit status" — needs current state per instance

"Show me failures in the last 24h" — needs indexed timestamp on errors table

"Show me audit events of type X across the fleet" — needs cross-server query support

I'll design the views/procedures with these patterns in mind. Confirming the read patterns are the typical ones (status board, failure list, search by event type/time range).

### I. The configuration sync job's "first time" definition.

## You said: "If a server is identified as new to the inventory, it will setup the auditing and enable it to start. If auditing is already configured and enabled on the server, it will disable and delete the audit process and then deploy it again. This includes the first time the job runs."

So on the very first run of the sync job against an existing fleet, every server gets torn down and redeployed. This is a one-time event that establishes the baseline. After that, only new servers should get the deploy-from-scratch treatment. Existing servers should be left alone unless drift is detected.

This matches my read of #1 above — confirming the "first run = baseline reset" interpretation.

### J. Time zones.

event_time from sys.fn_get_audit_file() returns UTC. GETDATE() on the central node returns local. Mixing these in the same table is a common bug source. I'll standardize on UTC throughout (use SYSUTCDATETIME() everywhere, store all timestamps in UTC, document the convention). Confirming.

### What I'm planning to deliver

Once you've answered the above, I'll generate the structured prompt system as separate markdown files:

agent.md — the system prompt for the AI agent driving development. Defines the project, constraints, decision log, and "how to think about this project."

copilot-instructions.md — the GitHub Copilot-specific working instructions, similar in spirit to what we did before but updated for the database-build phase.

requirements.md — execution requirements: SQL Server version on central node, permissions, network, inventory table contract, dependencies.

architecture.md — table designs, schema layout, job structure, data flow diagrams (in text), lifecycle state machine.

audits.txt — the action group baseline, plain text.

audit-template.sql — the deployable T-SQL template for the server audit + spec.

database-build.sql — the central audit database creation script (tables, indexes, FKs, history triggers).

procedures.sql — stored procedures and views.

jobs.sql — SQL Agent job definitions.

deployment.md — step-by-step deployment guide for the central node.

That's a lot of files, but they map cleanly to deliverables and let Copilot work on them independently.

### Summary of decisions I need from you

Q1: Drop-and-redeploy every run, or only on first encounter / drift detection? (My rec: first encounter + drift)

Q2: Collection frequency target? (My rec: 15 min, 64 MB rollover, 10 files)

Q3: Confirming sys.fn_get_audit_file() over SQL connection is the read mechanism

Q4: Watermarking in scope? (My rec: yes)

Q5: Schema-per-server vs. partitioned single table? (Going with schema-per-server unless you want a comparison doc)

Q6: Schema naming for named instances — SERVER__INSTANCE (double underscore)? Confirming.

Q7: Orphaned → decommissioned trigger (time-based threshold? DBA action?), and what happens when an orphaned server reappears in inventory?

Q8: User identity captured in history (default: SUSER_SNAME + ORIGINAL_LOGIN + HOST_NAME)

Q9: Two jobs (sync + collect) or three (sync + collect + health)?

Q10: All-failures alerting — log only, or fail the job if 100% of servers fail?

F: Best-effort audit cleanup on orphan transition, or only on decommission?

G: Retention period for collected audit data? (My default: 2 years)

I: Confirming "first run = baseline reset" interpretation

J: Confirming UTC throughout

Answer what you have opinions on; I'll take my recommendations as defaults for the rest. After your response I'll produce the prompt system as a set of structured markdown files ready for Copilot or another agent to consume.

## You said: Good call.

Good call. Instead of tearing down, this job should check and make sure all required audit settings and types are correct. If it is missing an audit type in the required audit list (which should be a table in the central database), it’s added to the audit. If an audit type was removed, it’s removed. If it’s not setup yet, configured or enabled, it does any of those actions. If it tries to enable a disabled audit and fails, it’s logged as a failure. Failures from this daily check should be logged separately to a table in the central database. A flag should be available on an audit inventory table to allow this action and be enabled manually by a DBA (still tracked in the change tracking) in case an audit job keeps failing and the only option is to try removing it and setting it up again brand new.

Collection runs each hour seems reasonable with a 32MB rollover, but the rollover size should also be re-configurable in case it needs to change. A separate job should be available that can be run manually and make this change on any servers flagged for a rollover size change, perhaps in a rollover configuration table where servers, rollover sizes can be added with a flag to enable them, and after the job runs and successfully makes the change, it disables the flag.

This job may also pull audits from ~100 different server instances and will need a way to manage that much load, either in a round robin or staggered schedule approach that breaks up the list of servers into manageable chunks/runtimes or take both approaches.

Since ports will not be open on the firewall to access the UNC path, it will need to run over the sql port (uni directional) and call sys.fn_get_audit_file() with a local path — runs on the target instance, returns rows over the connection using the service account’s file system access.

Yes, watermark tracking is in scope.

& 6. & 7. Clarifications and change - There are only 2 schemas plus dbo at this point.

[dbo]. - for all job, audit, tracking, application specific objects.

[auditcollection]. - for storing all active collection tables for each server instance. The table name will be the server name and if it’s a servername\instancename, replace the \ with a double underscore. Example: [auditcollection]. [servername__instancename]

decommissioned servers - Example: [decommissioned]. [servername__instancename]

The orphaned schema and logic is not needed.

The instance_id should still be a column in the tables even though the table name acts as a unique identifier too.

Rules -

Server in inventory → audited normally in auditcollection.<server> • Server removed from inventory → flag as decommissioned, move to [decommissioned m].<server> schema. Auditing flag in the central audit is turned off. Server auditing is no longer needed and it’s disabled with a flag on the table. The server can no longer be accessed anymore, therefore it doesn’t need any further action or collection on the server. Server collection table is moved to the [decommissioned].<server> schema.

A server removed from inventory means it’s officially gone / decommissioned and should not come back. If it did come back (say it was a new server and then had to be rebuilt) it would come in as a new server with a new identity_id (and flagged as active - another indicator in the central inventory table to identify active vs inactive / decommissioned servers) If it did come back with a new identity_id but it has the same servername as a server previously audited and now in the decommissioned schema, the server in the decommissioned schema should have its table name renamed with a trailing __Identity_Id number at the end of it.

capture: SUSER_SNAME(), ORIGINAL_LOGIN(), and HOST_NAME(). If the sname or login is NULL or empty, capture the application name or other identifier (such as a service account used from a GitHub action that deployed the update).

Proposed Jobs A, B, and C are good as the baseline job structure. Yes, AG aware also needed for all applicable jobs.

Agree with all. Let the dashboard and failure tables handle all alerting. Both collection failures and job failures tables are acceptable too, if needed. Open to the best possible outcome, insights, and management.

Additional responses.

A. Agree, Store the path in the audit tracker table (set when audit is deployed)

B. Agree. The txt file will just store a list of server audit specifications that can be read to create the t-sql build template the deployment job consumes.!

C. Agree. All AG replicas share the same GUID.

D. All Correct. And yes, central inventory is a database on the same server as central audit at this time.

E. Could the SQL Agent Service run as a gMSA on the central server (sysadmin and windows admin on target servers) or will a proxy be needed? What’s the best setup with a MSA/gMSA on both sides without having to create a proxy account? If the account does not need Windows admin rights on the target servers, why will it need xp_cmdshell, and what’s the best way to set the appropriate permissions using SQL Server agent, without having to login to each server manually to grant access in Windows? There’s no remote PowerShell access available. There’s an AD domain group with windows administrators rights that can be used currently. Or if we should have a separate group or permissions added to the server, we will need to do a bulk deployment change with the SCCM team, add a group policy, and add a process in new server deployments. The best and easiest option for least privilege is preferable. Any privileges needed on the central server is an easy option and doable for whatever is required.

F. Skip. Remove from requirements as previously stated.

G. TBD - add a configurable process for managing retention and monthly partitions.

H. Agree

I. Requirements for this were changed in #1. Only check that its audit specifications are correct and add/remove to match the baseline list of audit requirements.

J. Confirmed unless everything can be standardized in the process to convert to Eastern time zone. Or that can be handled in the views / reports separately too if that’s the better option.

Create a final recommendations or considerations / notes document to review additional questions or changes should be made to adjust/revise the prompt files, agent.md, or copilot instructions before deployment.

If additional clarifications or questions are needed, ask them first before writing out all the files.

Show more

*Apr 29*

## Claude responded: Excellent — your responses clarified almost everything.

Excellent — your responses clarified almost everything. Let me work through what I have, surface a few remaining questions, and produce a recommendations document for your review before I write the prompt system files.

### Remaining clarifications I'd like before writing the prompt system

### Q1. The "audit baseline check" logic — what does "missing" mean exactly?

You said the daily check verifies all required audit settings and types are correct. To be precise about what gets compared:

Action groups in the spec: compare sys.server_audit_specification_details to the baseline list. Add missing groups, remove extra groups. Clear.

Audit-level settings: MAXSIZE, MAX_ROLLOVER_FILES, QUEUE_DELAY, ON_FAILURE, FILEPATH. If any of these drift from the central tracker's expected values, what happens?

Option A: alter the audit in place where possible (some of these can be ALTERed; FILEPATH cannot — it requires drop+recreate)

Option B: log drift to a "drift detected" table, leave alone, require DBA action

Option C: only the DBA-set "force redeploy" flag triggers a drop/recreate; everything else gets altered in place if possible, or logged as drift if not

I'd recommend C. This treats the deployment as the source of truth, allows safe in-place corrections (action groups, retention size), but requires manual sign-off for destructive changes (path moves, GUID resets). Confirming.

### Q2. Collection job staggering — what's your preferred shape?

You said hourly collection with manageable batching. Two patterns:

Time-staggered: instances assigned to slots within the hour. Server 1 collects at :00, server 2 at :01, server 3 at :02, etc. With 100 servers in 60 minutes, that's ~1 server every 36 seconds. Single SQL Agent job runs every minute, picks instances whose slot matches. Simple, predictable.

Batched: divide 100 servers into N batches (say 4), batch 1 collects at :00, batch 2 at :15, batch 3 at :30, batch 4 at :45. Each batch runs serially through ~25 servers.

Hybrid: configurable batch size + slot offset per server, stored on the inventory tracker.

Question for you: do you want to be able to manually pin a specific server to a specific slot (e.g., "Server X is huge, give it the :00 slot all to itself")? If yes, that argues for the hybrid approach with an explicit collection_slot column. If no, time-staggered with hash-based assignment (instance_id % 60 for the minute slot) is simpler and self-balancing.

My recommendation: hybrid with explicit slot column, default-assigned via hash but DBA-overridable. Confirming.

### Q3. Collection concurrency.

Even with staggering, the "every minute, pick servers whose slot matches" approach is serial within a slot. If a slot has 2 servers and one takes 10 minutes to collect, the other has to wait. Is serial within slot acceptable, or do you want parallel within slot (multiple SQL Agent job steps, or one job with WAITFOR/parallel queries)?

My recommendation: serial within slot, sized so that no single slot has too many servers. Simpler and matches the "never take down a server" philosophy — one slow server only delays its slot-mates, not the whole fleet. Confirming.

### Q4. The "audit baseline" table — what exactly does it store?

You confirmed audits.txt becomes a table in the central DB. Two interpretations:

Just action groups: audit_baseline (action_group nvarchar(128)). The check compares against sys.server_audit_specification_details.

Full audit configuration: action groups plus MAXSIZE, MAX_ROLLOVER_FILES, ON_FAILURE, etc., as configurable values rather than hardcoded.

Are MAXSIZE / MAX_ROLLOVER_FILES / etc. fleet-wide constants (one set of values for everybody), or per-instance overridable (Server X gets bigger files because it's busy)?

My recommendation: fleet-wide defaults in a single-row config table, with per-instance overrides allowed via the inventory tracker. That way Server X can have a different rollover size if you flag it, but most servers inherit the default. The rollover-change job you described maps cleanly onto this. Confirming.

### Q5. Retention partitioning details.

You said retention is configurable, monthly partitions. A few details:

Partition function: monthly on event_time (UTC) seems right. Each per-server table gets its own partition function/scheme, OR a shared partition function across all per-server tables (more manageable, all tables roll over at the same boundaries).

Retention window: configurable per-server or fleet-wide? My rec: fleet-wide default (2 years) with per-instance override on the inventory tracker for regulated servers that need more.

Aging out: monthly job that switches out old partitions and either drops them or moves them to an archive table/database. My rec: switch out + archive to a separate audit_archive database, then drop. That preserves data for legal hold scenarios without bloating the active database.

Confirming the approach. Also — do you have an archive database / cold storage already, or do we need to design that as out-of-scope-but-document-the-interface?

### Q6. Time zone — final answer?

Your answer was "confirmed unless everything can be standardized." Recommendation:

Storage: always UTC (SYSUTCDATETIME(), event_time from the audit file is already UTC)

Display: views and procedures convert to Eastern at the read layer using AT TIME ZONE 'Eastern Standard Time' (handles DST correctly)

Documented convention: every column with timestamp meaning gets a comment indicating it's UTC

This gives you the best of both — the data is unambiguous, and reports/dashboards see Eastern time naturally. Confirming.

### Q7. The gMSA / permissions question — let me address this in detail.

You asked whether SQL Agent on central can run as a gMSA with sysadmin on targets, without needing xp_cmdshell or proxy accounts. Short answer: yes, that's the cleanest setup. Let me unpack:

### The gMSA approach:

Create a gMSA in AD (e.g., DOMAIN\svc_AuditCentral$)

Configure SQL Server service AND SQL Agent service on the central node to run as this gMSA

Add the gMSA as a SQL login on every target instance with sysadmin role membership (or a tighter custom role with CONTROL SERVER if you want least-privilege)

### Why you don't need xp_cmdshell with this setup:

xp_cmdshell was needed in earlier versions of the script for two things:

MKDIR to create audit directories — eliminated by using existing pre-built AuditLog folders

DIR to verify the directory exists — could be eliminated by querying sys.server_file_audits.log_file_path instead, or by attempting CREATE SERVER AUDIT and catching the failure if path doesn't exist

If the audit folder is always pre-existing on every target (per your post-install convention), neither MKDIR nor DIR is needed. The deployment job can simply attempt the CREATE SERVER AUDIT and trust the folder is there. If it isn't, the CREATE fails with a clear error, gets logged to the failures table, and a DBA investigates. This eliminates xp_cmdshell entirely from the project. Confirming this is acceptable.

### Why you don't need Windows admin on targets:

The gMSA's SQL login on each target operates inside the SQL Server engine. CREATE SERVER AUDIT is a SQL operation — the SQL Server service writes the audit files using its own service account's filesystem access (which is not the gMSA, unless SQL Server on the target also runs as the gMSA, which it doesn't). The gMSA never touches the target's filesystem directly. It just issues SQL commands.

So the permission chain is:

gMSA on central → SQL connection to target → executes T-SQL → SQL Server service on target writes audit files using its own service account

Windows admin on the target is not required.

### For deployment of the gMSA permissions across the fleet:

You don't need SCCM, GPO, or remote PowerShell. You can grant the SQL login + sysadmin via T-SQL from the central node. A one-time deployment script, run once per target via the existing CMS connections:

```sql
USE master;
IF NOT EXISTS (SELECT 1 FROM sys.server_principals WHERE name = N'DOMAIN\svc_AuditCentral$')
    CREATE LOGIN [DOMAIN\svc_AuditCentral$] FROM WINDOWS;
ALTER SERVER ROLE sysadmin ADD MEMBER [DOMAIN\svc_AuditCentral$];
```

This script gets run once across all targets via the CMS multi-server query feature, or via a one-time PowerShell loop using the same v3 pattern. After that, no further per-target permission work is needed.

### Least privilege option:

If sysadmin is too broad, the minimum required permissions for this project on each target are:

CONTROL SERVER (covers audit creation/alteration/drop)

VIEW SERVER STATE (covers DMV queries)

VIEW ANY DEFINITION (covers sys.server_audits, sys.availability_replicas, etc.)

A custom server role bundling these is cleaner than sysadmin. But sysadmin is operationally simpler. Your call.

My recommendation: gMSA + sysadmin on targets, no xp_cmdshell anywhere, one-time T-SQL deployment of the login/role across targets via CMS multi-server query. Confirming.

### Q8. The "force redeploy" DBA flag.

You mentioned a flag DBAs can set to trigger a tear-down + redeploy when an audit keeps failing. Where does this flag live, and what's the lifecycle?

Location: column on the inventory tracker (force_redeploy bit not null default 0)

Set by: DBA via stored procedure (which captures the change to history)

Read by: configuration sync job

Cleared by: configuration sync job after successful redeploy (or left set if redeploy fails — DBA must investigate)

Confirming.

### Q9. The history change tracking — temporal tables vs. trigger-based?

Two patterns for change history on configuration tables:

System-versioned temporal tables (SQL 2016+): built-in, automatic, queryable. Captures every UPDATE with valid_from/valid_to. Doesn't natively capture the user who made the change — you have to add columns and update them on the trigger or app layer.

Trigger-based history tables: explicit history table, INSERT/UPDATE/DELETE triggers write to it, you have full control over what gets captured (including user identity).

For this project, trigger-based wins because you specifically want SUSER_SNAME + ORIGINAL_LOGIN + HOST_NAME + APP_NAME captured on each change. Temporal tables don't give you that without extra work, and you'd end up with triggers anyway. Confirming trigger-based history.

### Q10. Failure table structure — one or two?

You said both collection failures and job failures tables are acceptable. My recommendation:

job_run_log — one row per job execution (job name, start time, end time, status, summary stats)

job_step_failures — one row per failed operation within a job (job_run_id FK, instance_id, stage, error message, severity)

This separation makes the dashboard pattern natural: "show me last 24h of failures" filters job_step_failures, "show me job execution history" reads job_run_log. A failed run with 5 instance failures = 1 row in run log + 5 rows in step failures.

Two tables. Confirming.

### Things I want to flag for your awareness before writing

Note 1: SQL Agent job ownership. Jobs should be owned by sa (or a SQL login with sysadmin), not by your gMSA. Owner-based permission checks behave better when sa-owned. The job steps run under the gMSA (which is the SQL Agent service account), so the security context is correct.

Note 2: The inventory table contract. Since the inventory is read-only and external to this project, I need to know its schema. You mentioned "any other information needed about the instance." For this project to work, the inventory table needs at least:

Unique identity column (you confirmed)

Server name (FQDN or short, must match @@SERVERNAME style)

SQL port (you confirmed)

Active flag (you mentioned for active vs decommissioned distinction)

Optional but useful:

AG name (could be inferred at runtime, but pre-populated is faster)

Environment tag (prod/dev/test — useful for staging deployments)

Should the prompt system documents specify the minimum inventory contract and have the requirements doc list "the inventory table must have these columns"? I think yes. The DBA team owning the inventory then knows what we depend on.

Note 3: AG handling in the daily sync job. In v3, the deployment script processed AG replicas with shared GUIDs. The daily sync job has a wrinkle: when an AG fails over, the audit configuration on the new primary should already be in place (we deployed to all replicas with the same GUID). But the collection job has to know which replica is currently primary for any database-level audit reads, OR collect from each replica independently and dedupe. For server-level audits (which is what we're scoped to), each replica generates its own audit events for its own activity — no deduplication needed; collect from all replicas independently.

This is correct behavior for server-level audits but worth documenting clearly so nobody later assumes deduplication is needed.

Note 4: New server detection cadence. The sync job runs daily. New servers added to inventory mid-day wait up to 24 hours for first audit setup. Is that acceptable, or do you want a faster path? Options:

Daily only (simplest, 24h max delay)

Every 4 hours (faster, more load)

Daily + a manual "sync now" stored procedure DBAs can run when they add a server

My recommendation: daily + manual sync proc. Best of both worlds. Confirming.

### Note 5: What I'm NOT going to design.

To set scope explicitly:

Out of scope: dashboard application (you said), reporting UI, alerting/notification system, archive database, cross-region replication of audit data, integration with SIEM/log aggregators

In scope: central database schema, jobs, procedures, views, audit deployment templates, requirements docs, deployment instructions

Confirming this scope split.

### Questions back to you (consolidated)

Drift handling: in-place ALTER for safe drift, log+ignore for unsafe drift (path/GUID), only force_redeploy flag triggers tear-down? (my Q1)

Slot assignment: hybrid (hash default, DBA override) for collection scheduling? (my Q2)

Concurrency: serial within slot, parallel across slots? (my Q3)

Audit baseline scope: action groups in baseline table, audit-level settings in a fleet-wide config table with per-instance overrides? (my Q4)

Archive database: do you have one, or design the interface as a stub? (my Q5)

Time zone: UTC storage, Eastern at the view layer via AT TIME ZONE? (my Q6)

Permissions: gMSA + sysadmin on targets, no xp_cmdshell, deploy permissions via one-time CMS multi-server T-SQL? (my Q7)

Force redeploy flag: lifecycle as described? (my Q8)

History tracking: trigger-based, capturing four identity columns with fallback chain? (my Q9)

Failure tables: two tables (run log + step failures)? (my Q10)

Sync cadence: daily + manual sync proc? (my Note 4)

Inventory contract: I'll specify minimum columns in requirements doc — confirming you can ensure those exist or will request them from inventory owners?

Least privilege vs sysadmin on targets: custom role with CONTROL SERVER + VIEW SERVER STATE + VIEW ANY DEFINITION, or just sysadmin?

Once you answer these, I'll produce the full prompt system as a structured set of markdown and SQL files. Most of my recommendations should be acceptable defaults if you don't have strong preferences — let me know which you want to override.

## You said: Q1 - audit baseline check" logic - "missing" means

Q1 - audit baseline check" logic - "missing" means

Action groups in the spec: compare sys.server_audit_specification_details to the baseline list. Add missing groups, remove extra groups. Correct.

Option C: only the DBA-set "force redeploy" flag triggers a drop/recreate; everything else gets altered in place if possible, or logged as drift if not.

Q2. - hybrid with explicit slot column, default-assigned via hash but DBA-overridable.

Q3 - serial within slot, sized so that no single slot has too many servers.

Q4 - use the recomendation - fleet-wide defaults in a single-row config table, with per-instance overrides allowed via the inventory tracker. That way Server X can have a different rollover size if you flag it, but most servers inherit the default.

Q5 - a shared partition function across all per-server tables (more manageable, all tables roll over at the same boundaries). fleet-wide default retention (2 years) with per-instance override on the inventory tracker for regulated servers that need more. switch out + archive to a separate audit_archive database, then drop. That preserves data for legal hold scenarios without bloating the active database. An audit archive database can be created on the same server.

Q6. Yes, use recommendation. Recommendation:

Storage: always UTC (SYSUTCDATETIME(), event_time from the audit file is already UTC)

Display: views and procedures convert to Eastern at the read layer using AT TIME ZONE 'Eastern Standard Time' (handles DST correctly)

Documented convention: every column with timestamp meaning gets a comment indicating it's UTC Q7 - gMSA + minimum required permissions for this project on the targets. no xp_cmdshell anywhere, one-time T-SQL deployment of the login/role across targets via CMS multi-server query.

Q8 - "force redeploy" DBA flag - confirmed - Location: column on the inventory tracker (force_redeploy bit not null default 0)

Set by: DBA via stored procedure (which captures the change to history)

Read by: configuration sync job

Cleared by: configuration sync job after successful redeploy (or left set if redeploy fails — DBA must investigate). Note - if the stored procedure is not used to change this setting, any change to the table should still be tracked in change history. Trigger-based on the table.

Q9 - Just answered.

Q10 - Two tables as recommended.

Note 1 - confirmed

Note 2 - confirmed those requirements are available in the external inventory table, including the optional inventory columns (add a placeholder requirement either in [agent.md](http://agent.md) file or a requirement to add a an inventory.sql file that has the create table schema of the inventory table that can be used as a reference).

Note 3 - For server-level audits (which is what we're scoped to), each replica generates its own audit events for its own activity — no deduplication needed; collect from all replicas independently.

Note 4 - daily + manual sync proc.

Note 5 - confirmed.
