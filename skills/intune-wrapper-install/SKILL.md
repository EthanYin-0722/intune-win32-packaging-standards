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
6. Read `references/pre-intune-validation.md` when packaging is complete and the user wants an optional pre-Intune upload validation run.
7. Read `references/review-skills.md` when the user says "review-skills", asks to summarize failures into the skill, or asks for the skill to self-update after a packaging/validation lesson.
8. Read `references/scenario-tests.md` when the request is broad, gives only an app name, lacks silent switches, or needs validation of wrapper-first behavior.
9. Inspect local installer metadata and source structure when the user provides a path.
10. Ask only for missing facts that materially affect the wrapper: source tree, installer filename, version/edition, license/config files, silent args, validation target, and reboot handling.

## Hard Confirmation Gate

Do not implement package changes, generate or overwrite `install.ps1`, create package folders, copy installers, run packaging tools, or perform cleanup/build actions until the user explicitly confirms the reviewed plan with wording such as "go ahead", "implement", "generate it", "build it", "create it", "proceed", "approved", or similarly clear authorization.

If the user asks to "make", "build", "create", "package", "fix", or "update" a package but has not yet confirmed the final reviewed flow, first provide the Pre-Final Review and ask for explicit confirmation. Treat urgency, a destination folder, or a requested output as implementation intent, not as confirmation to skip the gate.

Skip this gate only when the user explicitly says to skip review/confirmation or the user has already approved the same reviewed plan in the current thread. Narrow read-only inspection, metadata extraction, official-doc research, and non-destructive plan drafting may happen before confirmation.

## Optional Pre-Intune Validation

After package files are generated and the source/output layout is ready, ask whether the user wants a pre-Intune upload validation run. This validation is optional and must be explicitly accepted before running because it installs or uninstalls software on the local test machine.

Use an elevated PowerShell launch with `Start-Process -Verb RunAs` to simulate the privileged install conditions expected by Intune. This validates administrator/elevated install behavior on the local test machine; it is not a true LocalSystem simulation. Do not use PsExec/PSTools for this validation workflow unless the user explicitly requests it.

During validation, run the wrapper install command from an elevated process, capture the process exit code when available, then inspect the wrapper, transcript, vendor/MSI, and Intune Management Extension logs relevant to the package. After install validation passes, ask whether the user wants to test the uninstall script; run uninstall validation only after the user explicitly asks or confirms. If validation exposes an issue, list the evidence and proposed solution first. Do not edit scripts, rebuild packages, rerun destructive cleanup, uninstall software, or implement a fix until the user explicitly confirms the proposed action.

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
- Optional pre-Intune upload validation plan and results when the user opts in.
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
- **Optional Pre-Intune Validation**: ask whether to run an elevated RunAs install validation after packaging is complete; if install validation passes, ask whether to test uninstall; summarize exit code, detection result, logs reviewed, issues found, and proposed fixes.
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
- Do not generate the final `install.ps1`, create package folders, copy source files, run IntuneWinAppUtil, or otherwise implement until the user has explicitly approved the reviewed plan, unless they explicitly ask to skip the review.
- Do not run optional pre-Intune validation unless the user explicitly opts in after packaging is complete.
- Do not run uninstall validation unless the user explicitly asks for it or confirms after being prompted.
- Do not fix validation failures until the user has reviewed the issue list and explicitly approved the proposed solution.
- When the user invokes `review-skills` or asks to add a failure lesson to the skill, update the smallest relevant skill reference with a generalized rule, not package-specific clutter.
- When information is complete, build the wrapper directly; do not keep asking intake questions.
