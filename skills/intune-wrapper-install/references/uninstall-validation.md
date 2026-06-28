# Uninstall Validation

Use this reference when creating, reviewing, or testing Intune uninstall commands or `uninstall.ps1` wrappers.

## Confirmation

Uninstall validation removes software from the local test machine. Ask first unless the user already explicitly requested uninstall testing in the current turn:

```text
Do you want me to test the uninstall script now? This will remove the app from the local test machine.
```

Proceed only after explicit confirmation such as "yes", "test uninstall", "run uninstall", "go ahead", or "proceed".

## Uninstall Command Selection

Prefer uninstall methods in this order:

1. Vendor-documented silent uninstall command.
2. MSI ProductCode uninstall with `msiexec.exe /x {ProductCode} /qn /norestart /L*v <log>`.
3. QuietUninstallString from uninstall registry when verified silent and noninteractive.
4. UninstallString transformed to a silent command only when vendor/MSI behavior is understood and validated.

Never use `Win32_Product`.

## Wrapper Expectations

An uninstall wrapper should:

- Create the same MQ log root before running uninstall.
- Log wrapper, transcript, and vendor/MSI uninstall output separately.
- Stop app processes or services only when needed and safe for the product.
- Resolve source-side helper files from `$PSScriptRoot`.
- Use strict-mode-safe registry property reads.
- Return installer reboot codes when uninstall succeeds and detection confirms removal.
- Return nonzero when uninstall exits success but detection still finds the app.

Use uninstall phases:

```text
START, PRECHECK, STOP, UNINSTALL, CLEANUP, VALIDATE, FALLBACK, COMPLETE
```

## Return Codes

Map common MSI uninstall outcomes:

- `0`: uninstall success.
- `3010`: uninstall success with soft reboot.
- `1641`: uninstall success with hard reboot.
- `1618`: retry because another installation is in progress.
- `1605`: product not installed; success only when the detection method also reports not installed.
- `1614`: product uninstalled or product source missing; success only when detection reports not installed.
- Other nonzero: failed unless vendor docs define a safe success code.

## Cleanup Scope

Cleanup after uninstall may remove approved leftovers, but avoid broad deletion until the user has approved scope:

- Approved examples: empty vendor app folder, known app shortcuts, package-created config files.
- Risky examples: shared vendor folders, user data, license files, browser profiles, logs needed for troubleshooting.

Log cleanup decisions and skipped risky paths. Do not hide leftover user data or shared components; report them as retained.

## Validation

After uninstall:

- Run the package detection method and expect nonzero/not detected.
- Check ProductCode, file version target, service, and uninstall registry signals as applicable.
- Review uninstall wrapper, transcript, and MSI/vendor logs.
- If detection still finds the app, report evidence and propose a fix before editing or rerunning destructive actions.
- Do not reinstall the app unless the user explicitly asks.

## Failure Handling

If uninstall validation fails:

1. Report uninstall command, exit code, and relevant log paths.
2. Report detection command, exit code, and concise output.
3. Identify likely cause: wrong ProductCode, interactive uninstall string, running process lock, missing source, reboot pending, cleanup scope, or detection mismatch.
4. Propose the next fix or test.
5. Wait for explicit confirmation before implementing the fix or rerunning uninstall.
