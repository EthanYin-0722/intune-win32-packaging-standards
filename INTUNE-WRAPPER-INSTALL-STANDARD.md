# Intune Win32 Wrapper Install Standard

Use this document as the team standard for creating `install.ps1` wrapper scripts for Microsoft Intune Win32 apps. It is tool-agnostic: it can be used by Codex, ChatGPT, another AI assistant, or by an engineer writing the script manually.

## Goal

Every Win32 app package should use a wrapper-first install model:

- Intune runs `install.ps1`.
- `install.ps1` resolves source files from its own package root.
- The wrapper creates consistent logs under the team log path.
- The wrapper validates installation before returning success.
- The wrapper returns Intune-friendly exit codes.

The Intune install command should only launch the wrapper:

```text
%SystemRoot%\Sysnative\WindowsPowerShell\v1.0\powershell.exe -ExecutionPolicy Bypass -File .\install.ps1
```

## Intake Checklist

Collect these details before writing the production wrapper:

- Vendor, app name, version, edition, architecture, and install context.
- Installer filename and type: MSI, MSP, EXE, bootstrapper, or script.
- Source file list or source tree.
- Official or lab-tested silent install arguments.
- License, config, response, transform, or prerequisite files.
- License/config destination path and whether values are sensitive.
- Validation method: MSI ProductCode, file path/version, uninstall registry, service, or vendor CLI.
- Expected reboot behavior and known return codes.
- Official deployment, silent install, licensing, and logging documentation.

If silent EXE switches are unknown, do not guess. Research official docs or mark the command lab-validation-only.

## Source Layout

Default to a flat Intune content root named `source`. Assume all payload files sit directly beside `install.ps1` unless a package owner provides a different layout.

```text
source/
  install.ps1
  <installer.msi|setup.exe|patch.msp>
  <transform.mst>
  <response-file.iss|xml|ini>
  <license/config files>
  <vendor deployment notes, optional>
```

If the package owner supplies a nested tree, preserve it. Always resolve files from `$PSScriptRoot`; never rely on the current working directory.

## Log Standard

Default log root:

```text
C:\ProgramData\Microsoft\IntuneManagementExtension\Logs\MQ\<Vendor>
```

Default files:

```text
C:\ProgramData\Microsoft\IntuneManagementExtension\Logs\MQ\<Vendor>\<AppName>-<Version>-install-wrapper.log
C:\ProgramData\Microsoft\IntuneManagementExtension\Logs\MQ\<Vendor>\<AppName>-<Version>-install-transcript.log
C:\ProgramData\Microsoft\IntuneManagementExtension\Logs\MQ\<Vendor>\<AppName>-<Version>-install-vendor.log
```

For MSI/MSP packages, use `install-msi.log` or `install-msp.log` when clearer.

Wrapper log line format:

```text
yyyy-MM-ddTHH:mm:ss.fffzzz | LEVEL | PHASE | EVENT | Message | key=value; key=value
```

Allowed levels:

- `INFO`: expected progress and final result.
- `WARN`: fallback, optional item missing, retryable issue, reboot required.
- `ERROR`: fatal issue that returns nonzero.

Allowed phases:

- `START`: metadata, context, log path.
- `PRECHECK`: source files, prerequisites, license/config presence.
- `INSTALL`: installer command summary and exit code.
- `CONFIG`: license/config copy or post-install configuration.
- `VALIDATE`: detection or vendor verification.
- `FALLBACK`: alternate path after known failure.
- `COMPLETE`: final outcome and exit code.

Log decision points and outcomes only. Do not duplicate verbose MSI/vendor log content in the wrapper log.

Example:

```text
2026-06-25T14:32:18.456+10:00 | INFO | START | BEGIN | Starting install | vendor=Contoso; app=Example App; version=5.2.0; context=SYSTEM
2026-06-25T14:32:18.790+10:00 | INFO | PRECHECK | PASS | Source files found | installer=setup.exe
2026-06-25T14:32:19.103+10:00 | INFO | INSTALL | EXEC | Running installer | type=EXE; log=Example App-5.2.0-install-vendor.log
2026-06-25T14:45:02.221+10:00 | INFO | INSTALL | EXIT | Installer completed | exitCode=0
2026-06-25T14:45:04.013+10:00 | INFO | VALIDATE | PASS | Application detected | method=fileVersion
2026-06-25T14:45:04.500+10:00 | INFO | COMPLETE | SUCCESS | Install completed | exitCode=0
```

## Official Documentation Rule

Use this source priority before finalizing installer behavior:

1. Vendor enterprise deployment/admin guide for the exact product and version.
2. Vendor support article for silent install, unattended install, licensing, response files, or mass deployment.
3. Microsoft Learn for Intune Win32 behavior, PowerShell execution context, return codes, and detection.
4. Installer technology docs when vendor docs identify MSI, InstallShield, Inno Setup, NSIS, WiX Burn, or another bootstrapper.
5. Community posts only as clues. Label them unverified.

Extract:

- Installer type and whether the EXE is a bootstrapper.
- Silent, no-restart, wait, and log switches.
- Whether the installer creates log directories or needs the wrapper to create them.
- Required transforms, response files, license files, license server values, or config files.
- Dependencies, prerequisite order, and install context.
- Expected success, reboot, retry, and failure exit codes.
- Vendor-recommended validation method.

## Script Structure

Write `install.ps1` in this order:

1. Strict mode and constants.
2. Logging helpers.
3. Path resolution helpers.
4. Process execution helper.
5. Precheck block.
6. Install block.
7. Config block.
8. Validation block.
9. Return-code normalization.
10. Catch/finally with final log line.

## Installer Patterns

MSI:

```text
msiexec.exe /i "<msi>" /qn /norestart /L*v "<msi-log>"
```

MSP:

```text
msiexec.exe /p "<msp>" /qn /norestart /L*v "<msp-log>"
```

EXE/bootstrapper:

- Use vendor-confirmed silent, wait, and log switches only.
- If the EXE starts child processes and exits early, use a vendor wait switch or a lab-tested wrapper wait strategy.
- If EXE logging cannot target the team log root, capture wrapper events and document the vendor log location.
- If wrapper wait logic is needed, wait for a concrete install signal such as target file, registry key, service, or process completion with a bounded timeout. Log it under `FALLBACK`; do not wait indefinitely.

## Validation Standard

Prefer validation in this order:

1. MSI ProductCode in uninstall registry.
2. File exists with minimum file/product version.
3. Uninstall registry DisplayName and DisplayVersion.
4. Vendor CLI or qualification tool.
5. Service/process/marker only when vendor docs justify it.

Never use `Win32_Product`.

## Return Codes

Default Intune mapping:

```text
0    success
3010 soft reboot
1641 hard reboot
1618 retry
other nonzero failed
```

Rules:

- Return the original installer reboot code when validation passes.
- Return `1618` before validation so Intune can retry.
- Return installer nonzero codes unless the wrapper itself failed before knowing the installer result.
- Return `1` if the install block does not assign `$ExitCode`.
- Return `1` when wrapper validation fails, even if the installer returned `0`.
- Return `1` when required source/license/config files are missing.

## Security Rules

Do not log:

- Activation keys.
- License file contents.
- Credentials or tokens.
- Full command lines containing secrets.
- Full environment dumps.
- Complete registry exports.

Use `key=<redacted>` when a sensitive field matters diagnostically.

## Standard PowerShell Template

Replace every placeholder before production use. Remove unused MSI/EXE branches and confirm `$ExitCode` is assigned exactly once by the selected install path.

```powershell
#requires -Version 5.1
Set-StrictMode -Version Latest
$ErrorActionPreference = 'Stop'

$Vendor = '<Vendor>'
$AppName = '<AppName>'
$AppVersion = '<Version>'
$VendorSafe = (($Vendor.Split([IO.Path]::GetInvalidFileNameChars()) -join '_').Trim())
$AppSafe = (($AppName.Split([IO.Path]::GetInvalidFileNameChars()) -join '_').Trim())
$VersionSafe = (($AppVersion.Split([IO.Path]::GetInvalidFileNameChars()) -join '_').Trim())
$LogRoot = Join-Path -Path $env:ProgramData -ChildPath "Microsoft\IntuneManagementExtension\Logs\MQ\$VendorSafe"
$LogPrefix = "$AppSafe-$VersionSafe"
$WrapperLog = Join-Path -Path $LogRoot -ChildPath "$LogPrefix-install-wrapper.log"
$TranscriptLog = Join-Path -Path $LogRoot -ChildPath "$LogPrefix-install-transcript.log"
$VendorLog = Join-Path -Path $LogRoot -ChildPath "$LogPrefix-install-vendor.log"
$SuccessExitCodes = @(0, 3010, 1641)
$RetryExitCodes = @(1618)
$TranscriptStarted = $false

function ConvertTo-LogValue {
    param([AllowNull()][object]$Value)
    if ($null -eq $Value) { return '' }
    return ([string]$Value).Replace(';', ',').Replace("`r", ' ').Replace("`n", ' ')
}

function Write-PackageLog {
    param(
        [ValidateSet('INFO', 'WARN', 'ERROR')][string]$Level,
        [ValidateSet('START', 'PRECHECK', 'INSTALL', 'CONFIG', 'VALIDATE', 'FALLBACK', 'COMPLETE')][string]$Phase,
        [Parameter(Mandatory = $true)][string]$Event,
        [Parameter(Mandatory = $true)][string]$Message,
        [hashtable]$Data = @{}
    )
    $timestamp = Get-Date -Format 'yyyy-MM-ddTHH:mm:ss.fffzzz'
    $pairs = @()
    foreach ($key in ($Data.Keys | Sort-Object)) {
        $pairs += "$key=$(ConvertTo-LogValue -Value $Data[$key])"
    }
    $suffix = if ($pairs.Count -gt 0) { ' | ' + ($pairs -join '; ') } else { '' }
    Add-Content -Path $WrapperLog -Value "$timestamp | $Level | $Phase | $Event | $Message$suffix" -Encoding UTF8
}

function Resolve-PackagePath {
    param([Parameter(Mandatory = $true)][string]$RelativePath)
    $path = Join-Path -Path $PSScriptRoot -ChildPath $RelativePath
    if (-not (Test-Path -LiteralPath $path)) {
        throw "Required package item not found: $path"
    }
    return $path
}

function Invoke-Installer {
    param(
        [Parameter(Mandatory = $true)][string]$FilePath,
        [Parameter(Mandatory = $true)][object]$ArgumentList,
        [string]$Type = 'Unknown'
    )
    Write-PackageLog -Level INFO -Phase INSTALL -Event EXEC -Message 'Running installer' -Data @{
        type = $Type
        file = (Split-Path -Path $FilePath -Leaf)
        log = (Split-Path -Path $VendorLog -Leaf)
    }
    $process = Start-Process -FilePath $FilePath -ArgumentList $ArgumentList -Wait -PassThru -WindowStyle Hidden
    Write-PackageLog -Level INFO -Phase INSTALL -Event EXIT -Message 'Installer completed' -Data @{ exitCode = $process.ExitCode }
    return [int]$process.ExitCode
}

function Wait-ForInstallSignal {
    param(
        [Parameter(Mandatory = $true)][scriptblock]$Condition,
        [int]$TimeoutSeconds = 600,
        [int]$PollSeconds = 5,
        [string]$Description = 'install signal'
    )
    $deadline = (Get-Date).AddSeconds($TimeoutSeconds)
    Write-PackageLog -Level WARN -Phase FALLBACK -Event WAIT -Message 'Waiting for install signal' -Data @{
        signal = $Description
        timeoutSeconds = $TimeoutSeconds
    }
    while ((Get-Date) -lt $deadline) {
        if (& $Condition) {
            Write-PackageLog -Level INFO -Phase FALLBACK -Event PASS -Message 'Install signal observed' -Data @{ signal = $Description }
            return $true
        }
        Start-Sleep -Seconds $PollSeconds
    }
    Write-PackageLog -Level WARN -Phase FALLBACK -Event TIMEOUT -Message 'Timed out waiting for install signal' -Data @{ signal = $Description }
    return $false
}

function Complete-Install {
    param([Parameter(Mandatory = $true)][int]$ExitCode)
    if ($SuccessExitCodes -contains $ExitCode) {
        Write-PackageLog -Level INFO -Phase COMPLETE -Event SUCCESS -Message 'Install completed' -Data @{ exitCode = $ExitCode }
        exit $ExitCode
    }
    if ($RetryExitCodes -contains $ExitCode) {
        Write-PackageLog -Level WARN -Phase COMPLETE -Event RETRY -Message 'Installer requested retry' -Data @{ exitCode = $ExitCode }
        exit $ExitCode
    }
    Write-PackageLog -Level ERROR -Phase COMPLETE -Event FAILED -Message 'Install failed' -Data @{ exitCode = $ExitCode }
    exit $ExitCode
}

function Fail-Install {
    param(
        [Parameter(Mandatory = $true)][string]$Message,
        [int]$ExitCode = 1,
        [hashtable]$Data = @{}
    )
    $Data.exitCode = $ExitCode
    Write-PackageLog -Level ERROR -Phase COMPLETE -Event FAILED -Message $Message -Data $Data
    exit $ExitCode
}

New-Item -Path $LogRoot -ItemType Directory -Force | Out-Null
Write-PackageLog -Level INFO -Phase START -Event BEGIN -Message 'Starting install' -Data @{
    vendor = $Vendor
    app = $AppName
    version = $AppVersion
    context = [System.Security.Principal.WindowsIdentity]::GetCurrent().Name
}

try {
    Start-Transcript -Path $TranscriptLog -Append -Force | Out-Null
    $TranscriptStarted = $true
} catch {
    Write-PackageLog -Level WARN -Phase START -Event TRANSCRIPT -Message 'Could not start transcript' -Data @{ error = $_.Exception.Message }
}

try {
    # PRECHECK
    $InstallerRelativePath = '<installer>'
    $InstallerPath = Resolve-PackagePath -RelativePath $InstallerRelativePath
    Write-PackageLog -Level INFO -Phase PRECHECK -Event PASS -Message 'Source files found' -Data @{ installer = $InstallerRelativePath }

    # INSTALL - REQUIRED: assign $ExitCode from exactly one installer branch.
    $ExitCode = $null

    # MSI example. Uncomment and adapt when the installer is an MSI:
    # $Arguments = @('/i', $InstallerPath, '/qn', '/norestart', '/L*v', $VendorLog)
    # $ExitCode = Invoke-Installer -FilePath (Join-Path $env:SystemRoot 'System32\msiexec.exe') -ArgumentList $Arguments -Type 'MSI'

    # EXE example. Use only vendor-confirmed or lab-tested silent switches:
    # $Arguments = @('/quiet', '/norestart', "/log=`"$VendorLog`"")
    # $ExitCode = Invoke-Installer -FilePath $InstallerPath -ArgumentList $Arguments -Type 'EXE'

    # Bootstrapper fallback example. Use only when the EXE is known to exit before child installation completes:
    # $Waited = Wait-ForInstallSignal -Description 'installed executable' -TimeoutSeconds 900 -Condition {
    #     Test-Path -LiteralPath 'C:\Program Files\Vendor\App\App.exe'
    # }
    # if (-not $Waited) {
    #     Fail-Install -Message 'Timed out waiting for installer completion signal' -ExitCode 1
    # }

    # CONFIG
    # Copy license/config files here when required. Resolve them relative to $PSScriptRoot.
    # Log only summaries, not secrets.

    # VALIDATE
    # Use MSI ProductCode, file version, registry, or vendor validation here.
    if ($null -eq $ExitCode) {
        Fail-Install -Message 'Installer exit code was not captured; update the INSTALL block to assign $ExitCode' -ExitCode 1
    }

    if ($SuccessExitCodes -notcontains $ExitCode) {
        Complete-Install -ExitCode $ExitCode
    }

    $Detected = $false
    if (-not $Detected) {
        Write-PackageLog -Level ERROR -Phase VALIDATE -Event FAIL -Message 'Application was not detected after install'
        Fail-Install -Message 'Application was not detected after install' -ExitCode 1
    }
    Write-PackageLog -Level INFO -Phase VALIDATE -Event PASS -Message 'Application detected'

    Complete-Install -ExitCode $ExitCode
} catch {
    Fail-Install -Message 'Unhandled install exception' -ExitCode 1 -Data @{ error = $_.Exception.Message }
} finally {
    if ($TranscriptStarted) {
        try { Stop-Transcript | Out-Null } catch { }
    }
}
```

## Handoff Output

Every completed package should include:

- Source layout tree.
- Full `install.ps1`.
- Intune install command.
- Log paths.
- Validation method and recommended Intune detection rule.
- Return-code mapping.
- Official docs used, or an explicit unverified/lab-only note.
- Assumptions and known risks.

## Acceptance Check

Before handoff:

- No placeholders remain.
- All referenced source files exist in the proposed source tree.
- Installer arguments are quoted correctly.
- `$ExitCode` is assigned exactly once by the selected install branch.
- Wrapper uses `$PSScriptRoot`.
- Wrapper creates the MQ log folder before installer execution.
- Wrapper validation cannot pass because of a hardcoded `$true`.
- Failure paths call `Fail-Install` or `Complete-Install`, not raw `exit`, except as final lines inside those helpers.
- Script does not log secrets.
- EXE silent switches are official or lab-tested.
