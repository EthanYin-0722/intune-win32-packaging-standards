# Wrapper Standards

Apply these standards to every Intune `install.ps1` wrapper.

## Source Layout

Default to a flat Intune content root named `/source`. Assume all payload files are directly beside `install.ps1` unless the user provides or asks for subfolders:

```text
source/
  install.ps1
  <installer.msi|setup.exe|patch.msp>
  <transform.mst>
  <response-file.iss|xml|ini>
  <license/config files>
  <vendor deployment notes, optional>
```

If the user supplies a nested source tree, preserve it. The wrapper must resolve files from `$PSScriptRoot`; never rely on current working directory.

## Deliverable Folder Layout

When producing a complete package handoff, keep the master package folder easy for another EUC admin to scan. Separate the content root, upload artifact, build tooling, and admin notes:

```text
<PackageName>/
  README.md
  OnePager.md
  Output-Intune-Packages/
    <AppName>.intunewin
  Source/
    install.ps1
    uninstall.ps1
    detect.ps1
    <installer/config/wheels/files>
  Tools/
    <packaging tools>
```

Use `Source` as the folder passed to `IntuneWinAppUtil`, not the master folder. Use `Output-Intune-Packages` only for generated `.intunewin` files. Keep reusable or downloaded build tools in `Tools`. Include a root `README.md` with the upload file, install/uninstall commands, detection settings, prerequisites, included versions, log paths, and rebuild command.

When the package has a handoff audience beyond the packager, include a concise root `OnePager.md` with the Intune upload settings, dependency notes, validation status, and primary troubleshooting log. If a color-coded log preview is useful, create a separate static HTML example such as `Log-Color-Example.html` in the master folder. Do not add color escape codes, HTML, or viewer-only formatting to the wrapper script or actual `.log` files; keep production logs plain text and apply color only in the separate viewer/example.

## Log Layout

Default log root:

```text
C:\ProgramData\Microsoft\IntuneManagementExtension\Logs\MQ\<Vendor>
```

Default files:

- `<AppName>-<Version>-install-wrapper.log`: concise wrapper events.
- `<AppName>-<Version>-install-transcript.log`: optional PowerShell transcript for deeper troubleshooting.
- `<AppName>-<Version>-install-msi.log`, `<AppName>-<Version>-install-msp.log`, or `<AppName>-<Version>-install-vendor.log`: vendor/installer verbose log.

The path shape is:

```text
C:\ProgramData\Microsoft\IntuneManagementExtension\Logs\MQ\<Vendor>\<log>
```

Use sanitized path segments for `<Vendor>`, `<AppName>`, and `<Version>`.

Wrapper log line format:

```text
yyyy-MM-ddTHH:mm:ss.fffzzz | LEVEL | PHASE | EVENT | Message | key=value; key=value
```

Each install or uninstall run must also write plain-text deployment boundary lines before the first structured event and after the final result. This makes repeated runs in the same rolling log easy to separate without adding color codes or viewer-only formatting:

```text
----------------------------------------------------------------------------------------
INSTALL START: timestamp=<timestamp>; vendor=<Vendor>; app=<AppName>; version=<Version>; runId=<guid>; context=<user-or-SYSTEM>
----------------------------------------------------------------------------------------
yyyy-MM-ddTHH:mm:ss.fffzzz | INFO | START | BEGIN | Starting install | vendor=<Vendor>; app=<AppName>; version=<Version>; context=<user-or-SYSTEM>
...
yyyy-MM-ddTHH:mm:ss.fffzzz | INFO | COMPLETE | SUCCESS | Install completed | exitCode=0
----------------------------------------------------------------------------------------
INSTALL FINISH: timestamp=<timestamp>; vendor=<Vendor>; app=<AppName>; version=<Version>; runId=<same-guid>; result=SUCCESS; exitCode=0
----------------------------------------------------------------------------------------
```

Use `UNINSTALL START` and `UNINSTALL FINISH` for uninstall wrappers. The finish boundary must be written for success, retry, failure, and exception paths. Keep the boundary plain ASCII so Intune logs, CMTrace/OneTrace, text editors, and scripts can all read it.

Example:

```text
2026-06-25T14:32:18.456+10:00 | INFO | START | BEGIN | Starting install | vendor=Contoso; app=Example App; version=5.2.0; context=SYSTEM
2026-06-25T14:32:18.790+10:00 | INFO | PRECHECK | PASS | Source files found | installer=setup.exe
2026-06-25T14:32:19.103+10:00 | INFO | INSTALL | EXEC | Running installer | type=EXE; log=Example App-5.2.0-install-vendor.log
2026-06-25T14:45:02.221+10:00 | INFO | INSTALL | EXIT | Installer completed | exitCode=0
2026-06-25T14:45:04.013+10:00 | INFO | VALIDATE | PASS | Application detected | method=fileVersion
2026-06-25T14:45:04.500+10:00 | INFO | COMPLETE | SUCCESS | Install completed | exitCode=0
```

Use levels:

- `INFO`: expected progress and final result.
- `WARN`: fallback, optional item missing, retryable issue, reboot required.
- `ERROR`: fatal issue that returns nonzero.

Use phases:

- `START`: metadata, context, log path.
- `PRECHECK`: source files, prerequisites, license/config presence.
- `INSTALL`: installer command summary and exit code.
- `CONFIG`: license/config copy or post-install configuration.
- `VALIDATE`: detection/vendor verification.
- `FALLBACK`: alternate path after known failure.
- `COMPLETE`: final outcome and exit code.

Log decision points and outcomes only. Do not duplicate verbose MSI logs in the wrapper log.

## Script Structure

Use this order:

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

Use segment banners for major script regions so generated wrappers are easy to scan. Use this exact shape, uppercase titles, and only for major regions:

```powershell
# ---------------------------------------------------------------------------
# INSTALL SCRIPT
# ---------------------------------------------------------------------------
```

Start generated `install.ps1` scripts with:

```powershell
# ---------------------------------------------------------------------------
# INSTALL SCRIPT
# Package: <AppName> <Version>
# Vendor:  <Vendor>
# Purpose: Intune Win32 wrapper install script
# ---------------------------------------------------------------------------
```

Then use banners such as `RUNTIME SETTINGS`, `PACKAGE METADATA`, `LOGGING`, `HELPER FUNCTIONS`, `SOURCE VALIDATION`, `PRECHECK`, `INSTALL`, `CONFIGURATION`, `VALIDATION`, `COMPLETION`, and `ERROR HANDLING`. Avoid inline comments for obvious PowerShell statements.

When all package information is known, build the wrapper directly using `references/build-when-ready.md`; do not continue with generic intake questions.

## Installer Patterns

MSI:

```text
msiexec.exe /i "<msi>" /qn /norestart /L*v "<AppName>-<Version>-install-msi.log>"
```

MSP:

```text
msiexec.exe /p "<msp>" /qn /norestart /L*v "<AppName>-<Version>-install-msp.log>"
```

EXE/bootstrapper:

- Use vendor-confirmed silent/wait/log switches only.
- If the EXE starts child processes and exits early, use vendor wait switch or wrapper wait logic for the real process.
- If EXE logging cannot target the team log root, capture wrapper events and document vendor log location.
- If wrapper wait logic is needed, wait for a concrete install signal such as target file, registry key, service, or process completion with a bounded timeout. Log it under `FALLBACK`; do not wait indefinitely.

## Validation

Prefer validation in this order:

1. MSI ProductCode in uninstall registry.
2. File exists with minimum file/product version.
3. Uninstall registry DisplayName and DisplayVersion.
4. Vendor CLI or qualification tool.
5. Service/process/marker only when vendor docs justify it.

Never use `Win32_Product`.

For Intune detection details and custom `detect.ps1` behavior, read `detection-validation.md`. Wrapper validation and Intune detection should agree on the final installed state. If they disagree, treat it as a packaging issue and report evidence before proposing a fix.

## Uninstall Wrappers

When a package needs `uninstall.ps1`, use the same logging discipline as install wrappers and read `uninstall-validation.md`.

Prefer vendor-documented silent uninstall or MSI ProductCode uninstall. Use `QuietUninstallString` only when verified silent and noninteractive. Never use `Win32_Product`.

Uninstall wrappers should use phases:

```text
START, PRECHECK, STOP, UNINSTALL, CLEANUP, VALIDATE, FALLBACK, COMPLETE
```

Validation after uninstall must run the intended detection method and expect not installed. Return nonzero if uninstall reports success but detection still finds the target app/version.

## Registry Enumeration Under Strict Mode

When `Set-StrictMode -Version Latest` is enabled, never assume every uninstall registry entry has properties such as `DisplayName`, `DisplayVersion`, `Publisher`, `UninstallString`, or `QuietUninstallString`. Many registry keys are incomplete, and direct access like `$_.DisplayName` can throw before filtering reaches the target app.

Use strict-mode-safe property reads:

```powershell
$displayNameProperty = $_.PSObject.Properties['DisplayName']
$displayName = if ($null -ne $displayNameProperty) { [string]$displayNameProperty.Value } else { '' }
if ($displayName -like 'Example App*') {
    # process matching entry
}
```

Apply this pattern to install wrappers, uninstall wrappers, and custom detection scripts. This prevents precheck failures where validation never reaches the installer and no MSI/vendor log is created.

## Cleanup Scope

Separate cleanup into pre-install and post-install phases:

- Pre-install cleanup may remove approved leftover application folders only after old versions have been uninstalled.
- Post-install cleanup must not remove the installed application directory or files that detection depends on.
- Post-install cleanup should be limited to approved shortcuts, temporary package artifacts, or config actions that do not break detection.
- Run final validation after post-install cleanup so the wrapper cannot return success after deleting a required executable, registry entry, service, or marker.

If validation shows `VALIDATE | PASS` followed by cleanup warnings or detection failure, suspect post-install cleanup scope first. Fix by splitting app-folder cleanup from shortcut cleanup and moving final validation after the safe post-install cleanup step.

## Return Codes

Default mapping:

- `0`: success.
- `3010`: soft reboot.
- `1641`: hard reboot.
- `1618`: retry.
- Other nonzero: failed.

Return the original installer reboot code when validation passes, so Intune can apply restart behavior. Return `1` when wrapper validation fails even if the installer returned `0`.

Use `Complete-Install` for installer-controlled outcomes and `Fail-Install` for wrapper-controlled failures:

- Installer returns `0`, `3010`, or `1641` and validation passes: return the original code.
- Installer returns `1618`: return `1618` so Intune can retry.
- Installer returns another nonzero code: return the installer code.
- Installer returns success/reboot but validation fails: return `1`.
- Install block does not assign `$ExitCode`: return `1`.
- Precheck/config/wrapper exception fails: return `1` unless a vendor-approved code is more appropriate.

## Security

Do not log:

- Activation keys.
- License file contents.
- Credentials or tokens.
- Full command lines containing secrets.
- Full environment dumps.
- Complete registry exports.

Log redacted values as `key=<redacted>` when the field matters diagnostically.
