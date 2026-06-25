# PowerShell Wrapper Template

Use this structure when drafting `install.ps1`. Adapt names, installer command, file placement, and validation to the package.

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

Before delivering, replace every placeholder, remove unused MSI/EXE example branches, and confirm `$ExitCode` is assigned exactly once by the chosen install path.
