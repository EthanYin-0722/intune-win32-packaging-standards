---
name: intune-wrapper-install
description: Create and review Microsoft Intune Win32 PowerShell wrapper install scripts for enterprise Windows software. Use when Codex needs to package MSI, EXE, MSP, bootstrapper, or script-based installers with standardized flat-source handling by default, logging, prechecks, license/config placement, installer execution, fallback logic, validation, return-code handling, and official-documentation research. This skill is wrapper-first and should not stop at a bare install command.
---

# Intune Wrapper Install

## Workflow

Act as an Intune wrapper-script packaging advisor. Always design around `install.ps1`; the Intune install command is only the wrapper launcher.

1. Read `references/official-doc-research.md` before recommending vendor switches or licensing behavior.
2. Read `references/wrapper-standards.md` before writing or reviewing wrapper code.
3. Read `references/powershell-template.md` when drafting the actual `install.ps1`.
4. Read `references/build-when-ready.md` when installer details, silent arguments, source layout, and validation target are already known.
5. Read `references/pre-final-review.md` before generating the final script unless the user explicitly asks to skip confirmation or has already approved the reviewed plan.
6. Read `references/scenario-tests.md` when the request is broad, gives only an app name, lacks silent switches, or needs validation of wrapper-first behavior.
7. Inspect local installer metadata and source structure when the user provides a path.
8. Ask only for missing facts that materially affect the wrapper: source tree, installer filename, version/edition, license/config files, silent args, validation target, and reboot handling.

## Scope

Produce or review:

- `install.ps1` wrapper script.
- Intune install command to launch the wrapper:

```text
%SystemRoot%\Sysnative\WindowsPowerShell\v1.0\powershell.exe -ExecutionPolicy Bypass -File .\install.ps1
```

- Source folder recommendation.
- Log paths and log format.
- Validation and fallback behavior.
- Intune settings: install behavior, timeout, restart behavior, and return codes.

Do not produce only a bare MSI/EXE command unless the user explicitly asks to ignore wrapper standards.

## Intake

For underspecified requests, ask for:

- Product name, version, edition, architecture, and vendor.
- Installer filename and type, or local path.
- Source folder tree, including transforms, response files, license/config files, and subfolders.
- Licensing model: license server, license file, activation key, `.stcodes`, response file, or post-install activation.
- Existing official deployment/silent-install guide.
- Desired validation signal: MSI ProductCode, file path/version, uninstall registry entry, service, or vendor CLI check.
- Expected reboot behavior and known return codes.

## Output Contract

Return a concise wrapper package:

- **Pre-Final Review**: user-readable summary of install command, install logic, source layout, logs, validation, return codes, assumptions, and a simple flow before final script generation.
- **Source Layout**: recommended content root tree.
- **install.ps1**: full wrapper script or exact changes to an existing wrapper, using the standard sections and error handling.
- **Intune Install Command**: wrapper launcher only.
- **Logs**: log root, wrapper log, vendor/MSI log, and format.
- **Validation**: post-install check and detection recommendation.
- **Fallbacks**: alternate command/path for known failure modes.
- **Evidence**: official docs used, or a clear unverified note.
- **Intune Settings**: install behavior, timeout, restart behavior, and return-code mapping.

## Team Defaults

Default source layout is a flat Intune content root. Assume all package files sit directly in `/source` unless the user provides or asks for subfolders:

```text
source/
  install.ps1
  <installer.msi|installer.exe|patch.msp>
  <license/config/response/transform files>
```

If the user supplies a different layout, preserve it and resolve every file relative to `$PSScriptRoot`.

Default log root:

```text
C:\ProgramData\Microsoft\IntuneManagementExtension\Logs\MQ\<Vendor>
```

Default log files:

```text
C:\ProgramData\Microsoft\IntuneManagementExtension\Logs\MQ\<Vendor>\<AppName>-<Version>-install-wrapper.log
C:\ProgramData\Microsoft\IntuneManagementExtension\Logs\MQ\<Vendor>\<AppName>-<Version>-install-transcript.log
C:\ProgramData\Microsoft\IntuneManagementExtension\Logs\MQ\<Vendor>\<AppName>-<Version>-install-vendor.log
```

Wrapper log format:

```text
yyyy-MM-ddTHH:mm:ss.fffzzz | LEVEL | PHASE | EVENT | Message | key=value; key=value
```

Standard wrapper phases:

```text
START, PRECHECK, INSTALL, CONFIG, VALIDATE, FALLBACK, COMPLETE
```

Default return-code mapping:

```text
0 success; 3010 soft reboot; 1641 hard reboot; 1618 retry; other nonzero failed
```

## Guardrails

- Do not guess EXE silent switches as final. Use official docs or mark the command unverified and include lab validation.
- Do not use interactive installers for Intune production.
- Do not log secrets, activation keys, license contents, full environment dumps, or credentials.
- Do not hide missing license/config/response-file requirements.
- Do not return success until installer execution and wrapper validation agree.
- Do not use `Win32_Product` for validation or detection.
- Do not generate the final `install.ps1` until the user has reviewed the core plan, unless they explicitly ask to skip the review or have already approved it.
- When information is complete, build the wrapper directly; do not keep asking intake questions.
