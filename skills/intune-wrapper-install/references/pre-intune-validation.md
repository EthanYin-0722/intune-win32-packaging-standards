# Pre-Intune Upload Validation

Use this optional workflow after package files are generated and before the `.intunewin` is uploaded to Intune.

## Confirmation

Ask the user whether they want validation before running anything:

```text
Do you want me to run a pre-Intune validation now using an elevated PowerShell RunAs launch?
```

Run validation only after explicit acceptance such as "yes", "go ahead", "run validation", "validate it", or "proceed".

This validation uses `Start-Process -Verb RunAs`, not PsExec/PSTools. It validates elevated administrator install behavior on the local test machine. It does not exactly reproduce Intune LocalSystem context; if a true SYSTEM-context test is required, ask the user to explicitly request that separate mode.

## Validation Setup

Before running:

- Confirm the package source folder and `install.ps1` path.
- Confirm the intended Intune install command.
- Verify the wrapper log paths from the package plan.
- Warn that the validation may install, uninstall, update, or configure software on the local test machine.

## Elevated Install Simulation

Run the same wrapper launcher that Intune will run, but through an elevated PowerShell process. Example shape:

```powershell
$process = Start-Process -FilePath powershell.exe -ArgumentList @(
    '-NoProfile',
    '-ExecutionPolicy', 'Bypass',
    '-File', '<PackageSource>\install.ps1'
) -Verb RunAs -Wait -PassThru
$process.ExitCode
```

Use the package's actual source path and wrapper command. Do not use interactive installer UI. Do not add unplanned switches. Capture the elevated process exit code when available.

## Log Review

After the run, inspect relevant logs:

- Wrapper log under `C:\ProgramData\Microsoft\IntuneManagementExtension\Logs\MQ\<Vendor>\`.
- PowerShell transcript log if the wrapper starts one.
- Vendor/MSI verbose log created by the wrapper.
- `C:\ProgramData\Microsoft\IntuneManagementExtension\Logs\IntuneManagementExtension.log` only when Intune-agent behavior is relevant or the package was also tested through Intune.

Review for:

- Missing source/config/license files.
- Installer command or quoting failures.
- Prerequisite failures.
- Nonzero installer exit codes.
- Reboot codes `3010` or `1641`.
- MSI code `1618` retry conditions.
- Detection/validation mismatch.
- Permission, path, service, process, or cleanup failures.

## Optional Uninstall Validation

After install validation passes, ask the user whether they want to test the uninstall script:

```text
Install validation passed. Do you want me to test the uninstall script now? This will remove the app from the local test machine.
```

Run uninstall validation only after explicit confirmation such as "yes", "test uninstall", "go ahead", "run uninstall", or "proceed". Do not infer uninstall approval from install validation approval.

Use the package's uninstall command through the same elevated RunAs pattern. Example shape:

```powershell
$process = Start-Process -FilePath powershell.exe -ArgumentList @(
    '-NoProfile',
    '-ExecutionPolicy', 'Bypass',
    '-File', '<PackageSource>\uninstall.ps1'
) -Verb RunAs -Wait -PassThru
$process.ExitCode
```

After uninstall validation, inspect uninstall wrapper/transcript/vendor logs and verify the app is no longer detected using the package detection method. Report the result before any reinstall or fix. If the app must be restored for packaging continuity, ask before reinstalling it.

## Result Format

Report validation results before any fix:

1. `Command Run`: elevated PowerShell command, wrapper path, and exit code.
2. `Outcome`: pass, pass with reboot, retry needed, or fail.
3. `Evidence`: concise log lines or summarized events with paths.
4. `Issues`: each issue with likely cause and impact.
5. `Proposed Solution`: exact change or test to perform next.
6. `Uninstall Prompt`: if install validation passed and uninstall was not already requested, ask whether to test uninstall.
7. `Confirmation Ask`: ask for explicit approval before editing scripts, rebuilding packages, uninstalling/reinstalling software, or rerunning destructive actions.

## Fix Gate

If there is an issue, do not fix it immediately. First list the issue and propose a solution. Implement only after explicit user confirmation such as "go ahead with the fix", "apply it", "update the script", "rebuild", or "approved".

After an approved fix, rerun only the needed build or validation steps and summarize the new result.
