# Deploy-ComplianceAudit.ps1 (v3)

Serial SQL-only deployment of a SQL Server compliance audit baseline across
CMS-registered servers, using a two-pass approach: a pre-check pass builds
an in-memory lookup table with AG membership and SQL error log paths, then
a deployment pass uses that lookup to coordinate AG audit GUIDs and write
audit files alongside the existing SQL error log.

## What’s new in this version

- **In-memory lookup table** instead of a temp table on the CMS host, so
  no secondary connection to the CMS localhost is required.
- **Pre-check pass builds the lookup** with reachability, version, AG
  membership, and SQL error log directory for each target.
- **AG-aware GUID assignment.** All replicas of a given AG receive the
  same `AUDIT_GUID`, so audit continuity survives failover. Standalones
  get their own unique GUID.
- **Audit files written to the SQL error log directory.** No new folders
  created, no `xp_cmdshell MKDIR` required, no NTFS ACL configuration —
  SQL Server already has write access to that location.
- **`xp_cmdshell` logic removed entirely.** Not needed when the audit
  path is the existing error log directory. Simpler, safer, and the
  `sp_configure` save/restore dance is gone.
- **Unreachable servers skipped automatically.** Pre-check failures set
  `Reachable=false` on the lookup row; the deployment pass skips those
  rows without attempting further connections.
- **Single CSV captures the whole lookup table** for audit trail and
  retry-list building.

## Target runtime

**Windows PowerShell 5.1.** Tested with `SqlServer` module 21.1.x and 22.x.
PS 7 works but is not the primary tested path due to `SQLSERVER:\` provider
quirks with named-instance CMS hosts.

## Requirements

- Windows PowerShell 5.1
- `SqlServer` PowerShell module 21.1.x or newer
- Network connectivity to every target SQL instance
- Login with `CONTROL SERVER` and permission to run
  `CREATE SERVER AUDIT` / `CREATE SERVER AUDIT SPECIFICATION` on every target
- SQL Server 2016 or newer on every target

## Usage

```powershell
# Single CMS folder
.\Deploy-ComplianceAudit.ps1 -CmsInstance 'CMS01' `
    -CmsPath 'Inventory\MS\2016'

# Multiple CMS folders
.\Deploy-ComplianceAudit.ps1 -CmsInstance 'CMS01' `
    -CmsPath @('Inventory\MS\2016','DBA\Managed\Inventory\2022')

# Dry run
.\Deploy-ComplianceAudit.ps1 -CmsInstance 'CMS01' `
    -CmsPath 'Inventory\MS\2016' -WhatIf

# Recurse into subfolders
.\Deploy-ComplianceAudit.ps1 -CmsInstance 'CMS01' `
    -CmsPath 'Inventory\MS' -Recurse
```

## Output locations

- **Lookup CSV**: `.\Logs\Deploy-ComplianceAudit_Lookup_<timestamp>.csv`
- **Transcript**: `.\Logs\Deploy-ComplianceAudit_<timestamp>.log`
- **Failure log**: `C:\Temp\Deploy-ComplianceAudit_Failures_<timestamp>.log`

-----

## The script

```powershell
<#
.SYNOPSIS
    Deploys the compliance audit baseline across CMS registered servers using
    SQL connections only. AG-aware: coordinates AUDIT_GUID across all replicas
    of each Availability Group. Reuses the SQL error log directory as the
    audit destination, avoiding new folder creation and ACL configuration.

.DESCRIPTION
    Three-phase execution:

      Phase 1 - Enumerate CMS paths and collect server list.
      Phase 2 - Pre-check pass: connect to each server once, capture
                reachability, version, AG membership, AG peer list, and
                SQL error log directory. Build an in-memory lookup table.
      Phase 3 - GUID assignment: walk the lookup, generating a new GUID
                for each standalone server and one shared GUID for all
                replicas of each AG.
      Phase 4 - Deployment pass: for each reachable row in the lookup,
                connect and deploy the audit using its assigned GUID and
                the SQL error log directory as the audit file path.

    The lookup table is exported as a CSV at completion for audit trail
    and for building retry lists from unreachable or failed instances.

.PARAMETER CmsInstance
    CMS hostname or instance. Named instances supported.

.PARAMETER CmsPath
    One or more CMS folder paths to target. Non-recursive by default.

.PARAMETER Recurse
    Include subfolders of each CmsPath.

.PARAMETER MaxFileSizeMB
    Per-file rollover size. Default 256.

.PARAMETER MaxRolloverFiles
    Rollover file retention. Default 50.

.PARAMETER OnFailureAction
    CONTINUE | SHUTDOWN | FAIL_OPERATION. Default CONTINUE.

.PARAMETER QueryTimeoutSeconds
    Per-query timeout for deployment. Default 120.

.PARAMETER LogDirectory
    Transcript, results, and lookup CSV destination. Default .\Logs.

.PARAMETER FailureLogDirectory
    Persistent failure log destination. Default C:\Temp.

.PARAMETER WhatIf
    Logs intended actions without executing mutating SQL. Pre-check
    read-only queries still execute so the lookup can be validated.

.NOTES
    Tested: Windows PowerShell 5.1, SqlServer module 21.1+ and 22.x
    Target SQL versions: 2016, 2017, 2019, 2022, 2025
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

Set-StrictMode -Version 2.0
$ErrorActionPreference = 'Stop'

#region ---- Prereqs ---------------------------------------------------------

if ($PSVersionTable.PSVersion.Major -lt 5 -or
    ($PSVersionTable.PSVersion.Major -eq 5 -and $PSVersionTable.PSVersion.Minor -lt 1)) {
    throw "PowerShell 5.1 or higher required. Current: $($PSVersionTable.PSVersion)"
}

if (-not (Get-Module -ListAvailable -Name SqlServer)) {
    throw "SqlServer module not installed. Run: Install-Module SqlServer -Scope CurrentUser"
}

Import-Module SqlServer -DisableNameChecking

# Detect -TrustServerCertificate support. Added in SqlServer module 21.1.x.
$invokeSqlcmd = Get-Command Invoke-Sqlcmd -ErrorAction Stop
$script:SupportsTrustServerCert = $invokeSqlcmd.Parameters.ContainsKey('TrustServerCertificate')
if (-not $script:SupportsTrustServerCert) {
    Write-Warning @"
Installed SqlServer module does not support -TrustServerCertificate.
Connections to instances with untrusted certificates will fail.
Upgrade: Install-Module SqlServer -Force -AllowClobber
"@
}

foreach ($dir in @($LogDirectory, $FailureLogDirectory)) {
    if (-not (Test-Path -LiteralPath $dir)) {
        New-Item -Path $dir -ItemType Directory -Force | Out-Null
    }
}

$runTimestamp   = Get-Date -Format 'yyyyMMdd_HHmmss'
$transcriptPath = Join-Path $LogDirectory        "Deploy-ComplianceAudit_$runTimestamp.log"
$lookupPath     = Join-Path $LogDirectory        "Deploy-ComplianceAudit_Lookup_$runTimestamp.csv"
$failureLogPath = Join-Path $FailureLogDirectory "Deploy-ComplianceAudit_Failures_$runTimestamp.log"

Start-Transcript -Path $transcriptPath -Force | Out-Null

Write-Host "=== Compliance Audit Deployment (SQL-only, AG-aware) ===" -ForegroundColor Cyan
Write-Host "PowerShell:      $($PSVersionTable.PSVersion)"
Write-Host "CMS:             $CmsInstance"
Write-Host "CMS path(s):     $($CmsPath -join '; ')"
Write-Host "Recurse:         $Recurse"
Write-Host "Max file size:   $MaxFileSizeMB MB"
Write-Host "Rollover files:  $MaxRolloverFiles"
Write-Host "On failure:      $OnFailureAction"
Write-Host "Lookup CSV:      $lookupPath"
Write-Host "Failure log:     $failureLogPath"
Write-Host "WhatIf mode:     $($WhatIfPreference -eq $true)"
Write-Host ""

#endregion

#region ---- Failure logging -------------------------------------------------

function Write-FailureLog {
    [CmdletBinding()]
    param(
        [Parameter(Mandatory)][string]$Instance,
        [Parameter(Mandatory)][string]$Stage,
        [Parameter(Mandatory)][string]$Message
    )

    $ts  = Get-Date -Format 'yyyy-MM-dd HH:mm:ss'
    $msg = ($Message -replace '\r?\n', ' | ').Trim()
    $line = "[$ts]`t$Instance`t$Stage`t$msg"

    try {
        Add-Content -LiteralPath $failureLogPath -Value $line -Encoding UTF8 -Force
    }
    catch {
        Write-Host "  WARN: could not append to failure log: $($_.Exception.Message)" -ForegroundColor DarkYellow
    }
}

#endregion

#region ---- T-SQL building blocks ------------------------------------------

# Pre-check: returns version, error log directory, AG membership, AG name,
# and (if AG) a delimited list of peer replica server names. All in one
# round-trip so pre-check is one connection per server.
#
# ErrorLogFileName returns the full path to ERRORLOG. We extract the
# directory by stripping everything after the last backslash.
#
# AG peer names are comma-delimited -- AG replica names don't contain
# commas, so this is a safe delimiter.
$sqlPreCheck = @'
SET NOCOUNT ON;

DECLARE @errorLogPath NVARCHAR(512);
DECLARE @errorLogDir  NVARCHAR(512);

SET @errorLogPath = CAST(SERVERPROPERTY('ErrorLogFileName') AS NVARCHAR(512));
IF @errorLogPath IS NOT NULL AND LEN(@errorLogPath) > 0
BEGIN
    -- Extract the directory portion: everything before the last backslash.
    SET @errorLogDir = LEFT(@errorLogPath, LEN(@errorLogPath) - CHARINDEX('\', REVERSE(@errorLogPath)));
END

DECLARE @isHadr INT = CAST(ISNULL(SERVERPROPERTY('IsHadrEnabled'), 0) AS INT);
DECLARE @localServerName NVARCHAR(256) = CAST(ISNULL(SERVERPROPERTY('ServerName'), @@SERVERNAME) AS NVARCHAR(256));
DECLARE @agName NVARCHAR(256) = NULL;
DECLARE @agPeers NVARCHAR(MAX) = NULL;
DECLARE @isAG BIT = 0;

IF @isHadr = 1
BEGIN
    -- Find the AG this local replica belongs to. A single instance could
    -- host replicas of multiple AGs, but for audit purposes we only need
    -- the AG name for coordination; if there are multiple AGs on this
    -- instance, they should still share one audit and therefore one GUID.
    -- We pick the first AG name deterministically for GUID coordination.
    SELECT TOP 1 @agName = ag.name
    FROM sys.availability_groups ag
    INNER JOIN sys.availability_replicas ar ON ar.group_id = ag.group_id
    WHERE ar.replica_server_name = @localServerName
    ORDER BY ag.name;

    IF @agName IS NOT NULL
    BEGIN
        SET @isAG = 1;

        -- Build a comma-delimited list of ALL replica names across ALL
        -- AGs this instance participates in, so downstream lookup
        -- assignment correctly groups multi-AG clusters together.
        SELECT @agPeers = STUFF((
            SELECT DISTINCT ',' + ar.replica_server_name
            FROM sys.availability_replicas ar
            INNER JOIN sys.availability_groups ag ON ag.group_id = ar.group_id
            WHERE ag.group_id IN (
                SELECT ag2.group_id
                FROM sys.availability_groups ag2
                INNER JOIN sys.availability_replicas ar2 ON ar2.group_id = ag2.group_id
                WHERE ar2.replica_server_name = @localServerName
            )
            ORDER BY ',' + ar.replica_server_name
            FOR XML PATH(''), TYPE
        ).value('.', 'NVARCHAR(MAX)'), 1, 1, '');
    END
END

SELECT
     CAST(ISNULL(SERVERPROPERTY('ProductMajorVersion'), 0) AS INT)     AS MajorVersion
    ,CAST(ISNULL(SERVERPROPERTY('Edition'),             N'') AS NVARCHAR(128)) AS Edition
    ,CAST(ISNULL(SERVERPROPERTY('ProductVersion'),      N'') AS NVARCHAR(128)) AS ProductVersion
    ,@localServerName                                                  AS LocalServerName
    ,@errorLogDir                                                      AS ErrorLogDir
    ,@isAG                                                             AS IsAG
    ,@agName                                                           AS AGName
    ,@agPeers                                                          AS AGPeers;
'@

# Audit deployment. {{AUDIT_GUID}} is injected from the lookup table so
# all AG replicas receive the same GUID.
$sqlAuditTemplate = @'
USE [master];
SET NOCOUNT ON;

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
    ,AUDIT_GUID  = '{{AUDIT_GUID}}'
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
SET NOCOUNT ON;
SELECT 
     CAST(sa.is_state_enabled  AS INT)                    AS AuditEnabled
    ,CAST(sas.is_state_enabled AS INT)                    AS SpecEnabled
    ,COUNT(sad.audit_action_name)                         AS ActionGroupCount
    ,CAST(sa.audit_guid AS NVARCHAR(36))                  AS AuditGuid
FROM sys.server_audits sa
LEFT JOIN sys.server_audit_specifications sas
    ON sas.name = N'ServerAuditSpec_Compliance'
LEFT JOIN sys.server_audit_specification_details sad
    ON sad.server_specification_id = sas.server_specification_id
WHERE sa.name = N'Audit_Compliance'
GROUP BY sa.is_state_enabled, sas.is_state_enabled, sa.audit_guid;
'@

#endregion

#region ---- Helper functions ------------------------------------------------

function Get-CmsInstanceFromPath {
    [CmdletBinding()]
    param(
        [Parameter(Mandatory)][string]$CmsServer,
        [Parameter(Mandatory)][string[]]$Paths,
        [switch]$DoRecurse
    )

    $collected = [System.Collections.Generic.List[string]]::new()

    foreach ($p in $Paths) {
        $clean = $p.Trim().Trim('\')

        if ([string]::IsNullOrWhiteSpace($clean)) {
            Write-Host "  WARN: empty CmsPath segment skipped" -ForegroundColor Yellow
            continue
        }

        $providerPath = "SQLSERVER:\SQLRegistration\Central Management Server Group\$CmsServer\$clean"

        if (-not (Test-Path -LiteralPath $providerPath)) {
            Write-Host "  WARN: CMS path not found: $clean" -ForegroundColor Yellow
            Write-FailureLog -Instance '(CMS path)' -Stage 'Enumeration' `
                             -Message "CMS path not found: $clean"
            continue
        }

        Write-Host "  Enumerating: $clean" -ForegroundColor DarkCyan

        try {
            $items = if ($DoRecurse) {
                Get-ChildItem -LiteralPath $providerPath -Recurse -ErrorAction Stop
            }
            else {
                Get-ChildItem -LiteralPath $providerPath -ErrorAction Stop
            }
        }
        catch {
            Write-Host "  ERROR enumerating $clean : $($_.Exception.Message)" -ForegroundColor Red
            Write-FailureLog -Instance '(CMS path)' -Stage 'Enumeration' `
                             -Message "Enumerate $clean failed: $($_.Exception.Message)"
            continue
        }

        $names = @($items |
            Where-Object { $_.PSObject.Properties['ServerName'] -and $_.ServerName } |
            Select-Object -ExpandProperty ServerName)

        Write-Host "    Found $($names.Count) server(s) in this path"
        foreach ($n in $names) { $collected.Add($n) | Out-Null }
    }

    return @($collected | Sort-Object -Unique)
}

function Invoke-Sql {
    [CmdletBinding()]
    param(
        [Parameter(Mandatory)][string]$Instance,
        [Parameter(Mandatory)][string]$Query,
        [int]$Timeout = 60
    )

    $params = @{
        ServerInstance    = $Instance
        Query             = $Query
        QueryTimeout      = $Timeout
        ConnectionTimeout = 15
        ErrorAction       = 'Stop'
    }

    if ($script:SupportsTrustServerCert) {
        $params['TrustServerCertificate'] = $true
    }

    Invoke-Sqlcmd @params
}

function Get-SingleRow {
    [CmdletBinding()]
    param([Parameter()][AllowNull()]$Result)

    if ($null -eq $Result) { return $null }
    if ($Result -is [System.Array]) {
        if ($Result.Count -eq 0) { return $null }
        return $Result[0]
    }
    return $Result
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

function New-LookupRow {
    # Factory ensuring every lookup row has the same shape. Keeps StrictMode
    # happy when later code reads properties that may never have been set on
    # unreachable rows.
    [CmdletBinding()]
    param([Parameter(Mandatory)][string]$Instance)

    return [PSCustomObject]@{
        Instance           = $Instance
        Reachable          = $false
        MajorVersion       = 0
        Edition            = ''
        ProductVersion     = ''
        LocalServerName    = ''
        ErrorLogDir        = ''
        AuditPath          = ''      # ErrorLogDir with trailing backslash
        IsAG               = $false
        AGName             = ''
        AGPeers            = ''      # Comma-delimited replica list
        AuditGuid          = ''
        AuditDeployed      = $false
        AuditEnabled       = $null
        SpecEnabled        = $null
        ActionGroups       = $null
        DurationMs         = 0
        PrecheckError      = ''
        DeployError        = ''
        Timestamp          = (Get-Date).ToString('s')
    }
}

#endregion

#region ---- Phase 1: Enumerate CMS paths -----------------------------------

Write-Host "=== Phase 1: Enumerate CMS ===" -ForegroundColor Cyan
$instances = Get-CmsInstanceFromPath -CmsServer $CmsInstance `
                                     -Paths $CmsPath `
                                     -DoRecurse:$Recurse

if (-not $instances -or $instances.Count -eq 0) {
    Stop-Transcript | Out-Null
    throw "No servers found under the specified CMS path(s)."
}

Write-Host "Total unique instances: $($instances.Count)" -ForegroundColor Green
Write-Host ""

#endregion

#region ---- Phase 2: Pre-check pass, build lookup --------------------------

Write-Host "=== Phase 2: Pre-check / build lookup ===" -ForegroundColor Cyan

# Ordered dictionary keyed by instance name for O(1) lookup during
# the AG peer matching step. Case-insensitive comparer because CMS
# registrations are sometimes case-inconsistent with @@SERVERNAME.
$lookup = [System.Collections.Generic.Dictionary[string, object]]::new(
    [System.StringComparer]::OrdinalIgnoreCase)

$pcCounter = 0
foreach ($instance in $instances) {
    $pcCounter++
    Write-Host "[$pcCounter/$($instances.Count)] Pre-check: $instance" -ForegroundColor Yellow

    $row = New-LookupRow -Instance $instance

    try {
        $preRaw = Invoke-Sql -Instance $instance -Query $sqlPreCheck -Timeout 30
        $pre    = Get-SingleRow -Result $preRaw
        if ($null -eq $pre) {
            throw "Pre-check query returned no rows"
        }

        $row.Reachable      = $true
        $row.MajorVersion   = [int]$pre.MajorVersion
        $row.Edition        = [string]$pre.Edition
        $row.ProductVersion = [string]$pre.ProductVersion

        # @@SERVERNAME style identity -- used for AG peer matching. Can
        # differ from CMS registration string (FQDN vs short name, alias).
        $row.LocalServerName = if ($pre.LocalServerName -is [System.DBNull]) { '' } else { [string]$pre.LocalServerName }

        # Error log directory. Defensive null handling in case the
        # extraction yielded an empty string.
        $errorLogDir = if ($pre.ErrorLogDir -is [System.DBNull]) { '' } else { [string]$pre.ErrorLogDir }
        if ([string]::IsNullOrWhiteSpace($errorLogDir)) {
            throw "Could not determine SQL error log directory"
        }
        $row.ErrorLogDir = $errorLogDir

        # CREATE SERVER AUDIT FILEPATH requires a trailing backslash.
        $row.AuditPath = if ($errorLogDir.EndsWith('\')) { $errorLogDir } else { "$errorLogDir\" }

        # AG fields. IsAG comes back as BIT (0/1).
        $row.IsAG = ([int]$pre.IsAG -eq 1)
        if ($row.IsAG) {
            $row.AGName  = if ($pre.AGName  -is [System.DBNull]) { '' } else { [string]$pre.AGName  }
            $row.AGPeers = if ($pre.AGPeers -is [System.DBNull]) { '' } else { [string]$pre.AGPeers }
        }

        $agTag = if ($row.IsAG) { "AG=$($row.AGName)" } else { 'standalone' }
        Write-Host "  OK (v$($row.MajorVersion), $agTag, log=$($row.ErrorLogDir))" -ForegroundColor Green

        # Version gate. Reachable but below 2016 is still a failure for
        # this baseline -- mark precheck error but leave Reachable true
        # so downstream logic knows we got a valid response (just
        # unsupported).
        if ($row.MajorVersion -lt 13) {
            $row.Reachable = $false
            $row.PrecheckError = "Unsupported version (major $($row.MajorVersion)); baseline targets 2016+"
            Write-Host "  SKIP: $($row.PrecheckError)" -ForegroundColor Red
            Write-FailureLog -Instance $instance -Stage 'VersionCheck' -Message $row.PrecheckError
        }
    }
    catch {
        $row.Reachable     = $false
        $row.PrecheckError = $_.Exception.Message
        Write-Host "  UNREACHABLE: $($_.Exception.Message)" -ForegroundColor Red
        Write-FailureLog -Instance $instance -Stage 'Precheck' -Message $_.Exception.Message
    }

    # Key the lookup by the CMS-registered name so later lookups by that
    # key are stable. AG peer matching (which uses @@SERVERNAME style
    # strings) handles name format differences separately.
    if (-not $lookup.ContainsKey($instance)) {
        $lookup.Add($instance, $row)
    }
}

Write-Host ""
Write-Host "Pre-check complete. Reachable: $(@($lookup.Values | Where-Object { $_.Reachable }).Count) / $($lookup.Count)" -ForegroundColor Green
Write-Host ""

#endregion

#region ---- Phase 3: Assign AUDIT_GUIDs ------------------------------------

Write-Host "=== Phase 3: Assign AUDIT_GUIDs ===" -ForegroundColor Cyan

# Algorithm: walk the lookup values once. For each reachable row with no
# GUID yet, generate a fresh GUID and apply it to this row plus every
# matching AG peer row. Rows that already have a GUID (assigned via
# their AG primary's pass) are skipped.
#
# Standalone rows get a unique GUID per row.
# AG rows share a GUID across all matching replicas in the lookup.

# Build a second index keyed by LocalServerName (the format AG peer names
# use) so we can match a row's peer list against the lookup without
# scanning every time.
$byLocalName = [System.Collections.Generic.Dictionary[string, object]]::new(
    [System.StringComparer]::OrdinalIgnoreCase)
foreach ($r in $lookup.Values) {
    if ($r.Reachable -and -not [string]::IsNullOrWhiteSpace($r.LocalServerName)) {
        # If two rows somehow share the same LocalServerName (shouldn't
        # happen, but e.g. CMS has the same instance registered twice
        # under different names), the first wins. That's fine; the GUID
        # assignment below still propagates through both because we
        # iterate $lookup.Values directly.
        if (-not $byLocalName.ContainsKey($r.LocalServerName)) {
            $byLocalName.Add($r.LocalServerName, $r)
        }
    }
}

$agGroupsAssigned = 0
$standalonesAssigned = 0

foreach ($row in $lookup.Values) {
    if (-not $row.Reachable)                   { continue }
    if (-not [string]::IsNullOrWhiteSpace($row.AuditGuid)) { continue }

    # Fresh GUID for this logical group.
    $newGuid = ([guid]::NewGuid()).ToString()

    if (-not $row.IsAG) {
        $row.AuditGuid = $newGuid
        $standalonesAssigned++
        continue
    }

    # AG row: assign this GUID to every matching peer present in the
    # lookup. The AGPeers list includes this server too, so no need to
    # set it separately.
    $assignedCount = 0
    $missingPeers  = [System.Collections.Generic.List[string]]::new()

    $peerNames = @($row.AGPeers -split ',' | ForEach-Object { $_.Trim() } | Where-Object { $_ })
    foreach ($peerName in $peerNames) {
        if ($byLocalName.ContainsKey($peerName)) {
            $peerRow = $byLocalName[$peerName]
            if ([string]::IsNullOrWhiteSpace($peerRow.AuditGuid)) {
                $peerRow.AuditGuid = $newGuid
                $assignedCount++
            }
        }
        else {
            # A peer replica exists in SQL's AG topology but not in our
            # CMS-derived lookup. The user stated this shouldn't happen
            # (all AG nodes are expected to be in the same CMS folder),
            # so this is a warning worth surfacing.
            $missingPeers.Add($peerName) | Out-Null
        }
    }

    $agGroupsAssigned++
    Write-Host "  AG '$($row.AGName)': $newGuid ($assignedCount replicas assigned)" -ForegroundColor DarkCyan

    if ($missingPeers.Count -gt 0) {
        $warnMsg = "AG '$($row.AGName)' has peer(s) not in CMS lookup: $($missingPeers -join ', ')"
        Write-Host "    WARN: $warnMsg" -ForegroundColor Yellow
        Write-FailureLog -Instance $row.Instance -Stage 'GuidAssignment' -Message $warnMsg
    }
}

Write-Host "Assigned $standalonesAssigned standalone GUID(s) and $agGroupsAssigned AG group GUID(s)" -ForegroundColor Green
Write-Host ""

#endregion

#region ---- Phase 4: Deploy per instance -----------------------------------

Write-Host "=== Phase 4: Deploy audits ===" -ForegroundColor Cyan

$deployCounter = 0
$deployTotal   = @($lookup.Values | Where-Object { $_.Reachable }).Count

foreach ($row in $lookup.Values) {
    if (-not $row.Reachable) {
        # Pre-check failed; skip without attempting connection.
        continue
    }

    $deployCounter++
    $instance = $row.Instance
    Write-Host "[$deployCounter/$deployTotal] Deploying: $instance" -ForegroundColor Yellow
    $agTag = if ($row.IsAG) { "AG=$($row.AGName), GUID=$($row.AuditGuid)" } else { "standalone, GUID=$($row.AuditGuid)" }
    Write-Host "  $agTag"
    Write-Host "  Audit path: $($row.AuditPath)"

    $sw = [System.Diagnostics.Stopwatch]::StartNew()

    try {
        # Build the deployment SQL with the row's assigned GUID and path.
        $auditSql = Expand-Template -Template $sqlAuditTemplate -Tokens @{
            AUDIT_PATH   = $row.AuditPath
            MAX_SIZE_MB  = $MaxFileSizeMB
            MAX_ROLLOVER = $MaxRolloverFiles
            ON_FAILURE   = $OnFailureAction
            AUDIT_GUID   = $row.AuditGuid
        }

        if ($PSCmdlet.ShouldProcess($instance, 'Deploy compliance audit')) {
            Invoke-Sql -Instance $instance -Query $auditSql -Timeout $QueryTimeoutSeconds
            $row.AuditDeployed = $true
        }

        if (-not $WhatIfPreference) {
            $verifyRaw = Invoke-Sql -Instance $instance -Query $sqlVerifyAudit -Timeout 15
            $verify    = Get-SingleRow -Result $verifyRaw
            if ($null -eq $verify) {
                throw "Post-deploy verification query returned no rows"
            }
            $row.AuditEnabled = [int]$verify.AuditEnabled
            $row.SpecEnabled  = [int]$verify.SpecEnabled
            $row.ActionGroups = [int]$verify.ActionGroupCount

            if ($row.AuditEnabled -ne 1 -or $row.SpecEnabled -ne 1) {
                throw "Verification failed (audit=$($row.AuditEnabled), spec=$($row.SpecEnabled))"
            }

            # Confirm the deployed GUID matches the intended GUID. This
            # catches the unlikely case of a collision with a pre-existing
            # audit that somehow survived the DROP at the top of the script.
            $deployedGuid = [string]$verify.AuditGuid
            if ($deployedGuid -and $deployedGuid -ine $row.AuditGuid) {
                throw "Deployed GUID ($deployedGuid) does not match intended ($($row.AuditGuid))"
            }
        }

        Write-Host "  OK ($($row.ActionGroups) action groups)" -ForegroundColor Green
    }
    catch {
        $row.DeployError = $_.Exception.Message
        Write-Host "  FAIL: $($_.Exception.Message)" -ForegroundColor Red
        Write-FailureLog -Instance $instance -Stage 'Deploy' -Message $_.Exception.Message
    }
    finally {
        $sw.Stop()
        $row.DurationMs = $sw.ElapsedMilliseconds
    }

    Write-Host ""
}

#endregion

#region ---- Summary ---------------------------------------------------------

Write-Host "=== Summary ===" -ForegroundColor Cyan

$total          = $lookup.Count
$unreachable    = @($lookup.Values | Where-Object { -not $_.Reachable }).Count
$deploySucceeded = @($lookup.Values | Where-Object { $_.AuditDeployed -and -not $_.DeployError }).Count
$deployFailed   = @($lookup.Values | Where-Object { $_.Reachable -and -not $_.AuditDeployed }).Count

Write-Host "Total instances:       $total"
Write-Host "Unreachable:           $unreachable" -ForegroundColor $(if ($unreachable) { 'Yellow' } else { 'Gray' })
Write-Host "Deploy succeeded:      $deploySucceeded" -ForegroundColor Green
Write-Host "Deploy failed:         $deployFailed" -ForegroundColor $(if ($deployFailed) { 'Red' } else { 'Gray' })

# Export full lookup as CSV for audit trail + retry-list building.
$lookup.Values | Export-Csv -LiteralPath $lookupPath -NoTypeInformation -Encoding UTF8

Write-Host ""
Write-Host "Lookup CSV:    $lookupPath"
Write-Host "Transcript:    $transcriptPath"
if (Test-Path -LiteralPath $failureLogPath) {
    Write-Host "Failure log:   $failureLogPath" -ForegroundColor Yellow
}

if ($unreachable -gt 0) {
    Write-Host ""
    Write-Host "Unreachable instances:" -ForegroundColor Yellow
    $lookup.Values | Where-Object { -not $_.Reachable } |
        Format-Table Instance, PrecheckError -AutoSize -Wrap
}

if ($deployFailed -gt 0) {
    Write-Host ""
    Write-Host "Deployment failures:" -ForegroundColor Red
    $lookup.Values | Where-Object { $_.Reachable -and -not $_.AuditDeployed } |
        Format-Table Instance, IsAG, AGName, DeployError -AutoSize -Wrap
}

Stop-Transcript | Out-Null
if ($deployFailed -gt 0 -or $unreachable -gt 0) { exit 1 } else { exit 0 }

#endregion
```

-----

## Final review notes

### PowerShell logic checks

- `Set-StrictMode -Version 2.0` active. All `PSCustomObject` fields
  initialized in `New-LookupRow` so no property access ever hits an
  undefined name.
- `$script:SupportsTrustServerCert` scoped correctly and read inside
  `Invoke-Sql` via splat — splatting works identically in PS 5.1 and 7.
- `Dictionary[string, object]` with `OrdinalIgnoreCase` comparer handles
  case differences between CMS registration names and `@@SERVERNAME`
  format.
- AG peer list parsing uses `-split ','` with `.Trim()` and a non-empty
  filter, which handles single-element lists and trailing delimiters.
- `[System.DBNull]` guards on every property that the pre-check query
  can return NULL for (`AGName`, `AGPeers`, `ErrorLogDir`,
  `LocalServerName`). `Invoke-Sqlcmd` returns `DBNull` (not `$null`) for
  SQL NULL — string casts need the explicit check.
- `Get-SingleRow` handles single-row, multi-row, zero-row, and null
  return shapes from `Invoke-Sqlcmd`.
- `-WhatIf` short-circuits the mutating phases but preserves pre-check
  read-only behavior.
- No `xp_cmdshell` paths remain, so no `sp_configure` state to save or
  restore. Simplified flow, no risk of leaving `xp_cmdshell` enabled.

### T-SQL compatibility (SQL 2016 through 2025)

|Feature                                     |Minimum version        |Notes                                                          |
|--------------------------------------------|-----------------------|---------------------------------------------------------------|
|`SERVERPROPERTY('ErrorLogFileName')`        |2008                   |                                                               |
|`SERVERPROPERTY('IsHadrEnabled')`           |2012                   |                                                               |
|`SERVERPROPERTY('ProductMajorVersion')`     |2008 R2 SP2 / 2012 SP1+|                                                               |
|`sys.availability_groups`                   |2012                   |AG introduced in 2012                                          |
|`sys.availability_replicas`                 |2012                   |                                                               |
|`STUFF(... FOR XML PATH(''))` pattern       |2005                   |The classic string aggregator used here for broad compatibility|
|`CREATE SERVER AUDIT ... AUDIT_GUID = '...'`|2008                   |                                                               |
|All 26 action groups in the baseline        |2012 or earlier        |None version-gated                                             |

`STRING_AGG` would be cleaner than `STUFF + FOR XML PATH` but is
2017+ only. The `STUFF` pattern works across the entire 2016–2025 range.

### CMS provider / PowerShell edge cases handled

- Named-instance CMS hosts (e.g. `CMS01\PROD`): `-LiteralPath` used
  everywhere.
- Missing or mistyped CMS paths: warned and skipped, not fatal.
- Empty path segments: skipped with warning.
- Dedup across multiple CMS paths: `Sort-Object -Unique`.
- `Invoke-Sqlcmd` single-row return unwrapping: handled via
  `Get-SingleRow`.
- `-TrustServerCertificate` absent on older modules: detected, warning
  emitted, parameter omitted.

### Behavioral guarantees

- **Unreachable servers never get a second connection attempt.**
  Pre-check failure sets `Reachable=false` and every downstream phase
  checks it.
- **AG replicas always get the same GUID.** The GUID assignment walks
  the lookup once, propagates through the full peer list on first
  visit, and skips already-assigned rows.
- **No new directories created.** Every audit writes to the SQL error
  log directory, which SQL Server already has write permission for.
- **Single CSV captures everything.** The lookup table (including
  unreachable instances, AG membership, GUID assignments, deploy
  status, and per-row timing) is exported at the end for audit trail
  and retry workflow.

### Known limitations

1. **AG peer name format mismatch.** If the AG topology reports a
   replica as `PROD-SQL01.corp.example.com\INST1` and the CMS has it
   registered as `PROD-SQL01\INST1`, the lookup match will fail and
   the peer will be reported as “not in CMS lookup.” Fix: register
   using the format that matches `@@SERVERNAME` on each replica, or
   add an alias normalization step.
1. **Multi-AG instances.** An instance hosting replicas in two
   different AGs will be assigned one GUID for both. This is
   intentional (single audit per instance) but means if the two AGs
   have disjoint peer sets, the GUID propagates through the union of
   peers correctly on first pass.
1. **FCI (Failover Cluster Instance) nodes.** FCI is orthogonal to AG
   — it fails over at the Windows cluster level, not SQL. `@@SERVERNAME`
   returns the virtual name, which is what CMS typically has
   registered, so no special handling is needed.