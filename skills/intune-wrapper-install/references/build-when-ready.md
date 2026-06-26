# Build When Information Is Complete

Use this guide when the user has already provided installer details, source layout, silent arguments, license/config requirements, and validation target.

## Required Inputs

Do not ask more intake questions if these are known:

- Vendor, app name, version, architecture, and install context.
- Installer relative path under the package root.
- Installer type: MSI, MSP, EXE/bootstrapper, or script.
- Silent arguments and log argument syntax, verified by official docs or lab testing.
- Source layout, including transforms, response files, license/config files, and destination paths.
- Validation method: MSI ProductCode, file path/version, uninstall registry, service, or vendor CLI.
- Expected success/reboot/retry codes.

If one of these is missing but can be inferred safely from a local installer, inspect it. Otherwise ask only for that missing item.

When these inputs are known, use `pre-final-review.md` before writing the final script or creating/building package files unless the user explicitly skipped or already approved that review. The review is a confirmation checkpoint, not a new intake round.

Require explicit implementation confirmation after the review. Accept clear wording such as "go ahead", "implement", "generate it", "build it", "create it", "proceed", or "approved". Do not infer confirmation from a requested destination folder, package name, or broad instruction to make/build/create a package.

## Standard Output

After the user confirms the pre-final review, return the package in this order:

1. `Source Layout`: tree of the Win32 content root.
2. `install.ps1`: full script, not pseudocode.
3. `Intune Install Command`: wrapper launcher.
4. `Log Output`: exact wrapper log and vendor log paths.
5. `Validation`: post-install validation and recommended Intune detection.
6. `Optional Pre-Intune Validation`: ask whether the user wants an elevated RunAs validation before upload.
7. `Return Codes`: Intune mapping and any custom vendor codes.
8. `Assumptions`: only items not proven by artifact inspection or official docs.

## Build Steps

Before writing the final `install.ps1`, copying source files, creating package folders, or running packaging tools, confirm these core decisions with the user in a readable pre-final review:

- Source layout and installer relative path.
- Effective installer command and evidence level, with secrets redacted.
- Install logic flow: precheck, install, config, validation, fallback, completion.
- Log paths and log format.
- Validation signal and expected return-code behavior.
- Assumptions, risks, and anything still unverified.

Use a flat `/source` content root by default:

```text
source/
  install.ps1
  <installer>
  <license/config/response/transform files>
```

Use subfolders only when the user provides them or asks for them.

Create `install.ps1` in these sections:

1. **Constants**: app metadata, paths, success/retry codes, install command arguments.
2. **Logging helpers**: timestamped `LEVEL | PHASE | EVENT` wrapper log.
3. **Path helpers**: resolve all content from `$PSScriptRoot`.
4. **Execution helper**: run installer with `Start-Process -Wait -PassThru`.
5. **Precheck**: verify source files, config/license files, log folder, and obvious prerequisites.
6. **Install**: run vendor command and log only command summary plus exit code.
7. **Config**: copy or write config files without logging secrets.
8. **Validation**: confirm the app is installed after successful installer exit.
9. **Completion**: return original success/reboot code when validation passes.
10. **Error handling**: log fatal errors and return nonzero.

## Error Handling Standard

Use one completion path and one failure path:

- `Complete-Install`: handles installer codes and exits with `0`, `3010`, `1641`, `1618`, or vendor-approved codes.
- `Fail-Install`: logs `ERROR | COMPLETE | FAILED` and exits nonzero, usually `1`.

Rules:

- If the installer returns `1618`, return `1618` before validation so Intune can retry.
- If the installer returns any other non-success code, return that code unless the wrapper failure itself prevents knowing the installer code.
- If the install block does not assign `$ExitCode`, fail with `1` before validation.
- If the installer returns success/reboot but validation fails, return `1`.
- If a required source/license/config file is missing, return `1` before running the installer.
- Always stop the transcript in `finally`.
- Always write a final `COMPLETE` log line for success, retry, failure, and unhandled exception paths.

## Logging Standard

Wrapper log:

```text
C:\ProgramData\Microsoft\IntuneManagementExtension\Logs\MQ\<Vendor>\<AppName>-<Version>-install-wrapper.log
```

Vendor log:

```text
C:\ProgramData\Microsoft\IntuneManagementExtension\Logs\MQ\<Vendor>\<AppName>-<Version>-install-vendor.log
```

Log only:

- `START | BEGIN`: metadata and run context.
- `PRECHECK | PASS/FAIL`: source/config/prereq status.
- `INSTALL | EXEC`: installer type, installer filename, vendor log filename.
- `INSTALL | EXIT`: installer exit code.
- `CONFIG | COPY/PASS/FAIL`: config action summary.
- `VALIDATE | PASS/FAIL`: validation result.
- `COMPLETE | SUCCESS/RETRY/FAILED/EXCEPTION`: final outcome.

Do not log:

- License keys or license file contents.
- Full command lines containing secrets.
- Full environment dumps.
- Repeated progress lines.
- Verbose MSI log content.

## Minimum Acceptance Check

Before handing off, check:

- Pre-final review has been confirmed, or confirmation was explicitly skipped by the user.
- The user explicitly authorized implementation with clear go-ahead wording after the review, unless they explicitly skipped review/confirmation.
- No placeholders remain.
- All referenced source files exist in the proposed source tree.
- Installer arguments are quoted correctly.
- `$ExitCode` is assigned exactly once by the selected install branch.
- Wrapper uses `$PSScriptRoot`.
- Wrapper creates `C:\ProgramData\Microsoft\IntuneManagementExtension\Logs\MQ\<Vendor>` before installer execution.
- Wrapper validation cannot pass accidentally because of a hardcoded `$true`.
- Failure paths call `Fail-Install` or `Complete-Install`, not raw `exit`, except as the final line inside those helpers.
- If the package is complete, user has been asked whether they want optional pre-Intune elevated RunAs validation before upload.
