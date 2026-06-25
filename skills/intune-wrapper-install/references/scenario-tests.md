# Scenario Tests

Use these generic scenarios to validate wrapper-first behavior. Do not encode product-specific install knowledge in the skill; use official documentation or user-provided evidence for each real package.

## Broad Product Request

User:

```text
Help me create an Intune install script for <ProductName> <Version>.
```

Expected behavior:

- Use this skill because the user asks for an Intune install script.
- Research official deployment, silent install, licensing, and logging documentation first.
- Do not guess EXE silent switches or license behavior.
- Ask only for missing facts needed to build a production wrapper.

Ask for:

- Installer filename and local path if available.
- Flat `/source` file list, or the actual source tree if subfolders are used.
- Installer type: MSI, MSP, EXE, bootstrapper, or script.
- License/config/response/transform files and their required destination.
- Official or lab-tested silent install arguments.
- Validation target: MSI ProductCode, installed file path/version, uninstall registry, service, or vendor CLI.

Good response shape:

```text
I will build a wrapper-first Intune install.ps1. Please send the installer filename, source file list, silent arguments, and validation target. I will assume a flat /source folder unless your package uses subfolders. I will not guess silent EXE switches; if official docs are unavailable, the wrapper will be lab-validation-only until tested under SYSTEM.
```

## Standard MSI

User:

```text
ExampleApp.msi version 5.2, make Intune wrapper.
```

Expected wrapper:

- Flat source layout: `source\install.ps1` and `source\ExampleApp.msi`.
- Wrapper creates the team log folder before installer execution.
- Runs `msiexec.exe /i` with `/qn /norestart /L*v`.
- Assigns `$ExitCode` from the MSI install branch before validation.
- Validates with MSI ProductCode if supplied; otherwise asks for ProductCode or local MSI path to inspect.
- Returns original `0`, `3010`, or `1641` when validation passes.

## MSP Patch

User:

```text
Deploy update.msp.
```

Expected wrapper:

- Confirm the base app prerequisite.
- Flat source layout: `source\install.ps1` and `source\update.msp`.
- Run `msiexec.exe /p` with verbose log.
- Assign `$ExitCode` from the MSP install branch before validation.
- Validate base app and patched version.
- Recommend an Intune requirement or detection rule for the base product.

## EXE Unknown Switches

User:

```text
I only have setup.exe. Build the wrapper.
```

Expected wrapper behavior:

- Do not finalize `/quiet`, `/S`, or `/silent` unless verified.
- Ask for vendor, product, version, installer source, and source file list.
- Research official deployment docs.
- If docs are missing, provide a discovery/lab plan, not production wrapper code with guessed switches.

## EXE Official Switches

User:

```text
Vendor docs say setup.exe /quiet /norestart /log install.log.
```

Expected wrapper:

- Flat source layout unless the user provides subfolders.
- Create log root first.
- Run EXE with vendor-confirmed args and an absolute log path if supported.
- Assign `$ExitCode` from the EXE install branch before validation.
- If the EXE cannot create directories or needs relative logs, wrapper handles folder prep and documents vendor log location.
- Validate install before returning success.
- Before final script generation, show a pre-final review with install command, simple text flow, logs, validation, return codes, evidence, and risks.

## Information Complete

User:

```text
Vendor is Contoso, app is Example App 5.2 x64. Source has install.ps1 and setup.exe. Official docs say setup.exe /quiet /norestart /log <path>. Validate by C:\Program Files\Contoso\Example\App.exe version 5.2.0. Install under SYSTEM.
```

Expected behavior:

- Do not ask broad intake questions.
- Show the pre-final review first, not the final script.
- Include a readable install command summary with secrets redacted if present.
- Include a concise text flow for precheck, install, config, validation, fallback, and completion.
- Ask the user to confirm or correct the plan.
- Generate the final `install.ps1` only after confirmation, unless the user explicitly asked to skip confirmation.

## License Or Config Copy

User:

```text
The installer also needs a license file and config file.
```

Expected wrapper:

- Ask for exact license/config destination and whether any values are secret.
- Resolve required files from the flat `/source` root unless the user provides subfolders.
- Copy or place files during `CONFIG`.
- Log copy summary only, not file contents or secrets.
- Validate that required config exists and the app is installed.

## Bootstrapper Or Child Process

User:

```text
The setup EXE exits quickly but installation continues in another process.
```

Expected wrapper:

- Prefer a vendor-supported wait switch when documented.
- If no wait switch exists, include a documented lab-tested wait strategy for the real child process, installed file, MSI product code, or registry signal.
- Use a bounded timeout; do not wait indefinitely.
- Log fallback behavior under `FALLBACK`.
- Validate before returning success.

## Complex Upgrade Or Cleanup

User:

```text
Before installing the new version, remove old versions and clean leftover files.
```

Expected wrapper behavior:

- Use the pre-final review gate before final script generation.
- Upgrade the review flow from simple text to Mermaid because the package has multiple decision branches.
- Show detection of target version, old-version uninstall, cleanup, install, config, validation, and return-code behavior.
- Confirm destructive cleanup scope with the user before writing final script.
- Do not delete files, registry keys, services, scheduled tasks, or user data unless the user has approved the exact scope.

## Custom Source Tree

User:

```text
My package uses Files\setup.exe and Config\app.ini.
```

Expected wrapper:

- Preserve the user-provided tree.
- Resolve `Files\setup.exe` and `Config\app.ini` from `$PSScriptRoot`.
- Do not rewrite the user's layout back to flat source.

## Coverage Checklist

The wrapper skill should cover:

- MSI, MSP, EXE, bootstrapper, script, and response-file installs.
- Flat `/source` layout by default, with custom subfolder support when provided.
- Unified MQ log folder and timestamped wrapper log format.
- License/config copy without leaking secrets.
- Official-doc-first switch validation.
- Prechecks and post-install validation.
- Fallback for EXE child process, missing logs, missing config, and unverified switches.
- Return-code mapping for Intune.
- Intune wrapper launcher command.
