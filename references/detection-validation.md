# Detection Validation

Use this reference when creating, reviewing, or testing Intune Win32 detection rules or custom `detect.ps1` scripts.

## Detection Selection

Prefer detection methods in this order unless vendor evidence or package behavior justifies another choice:

1. MSI ProductCode when the installed product code is stable for the deployed version.
2. File path with exact or minimum file/product version.
3. Uninstall registry DisplayName plus DisplayVersion and Publisher when ProductCode is unavailable.
4. Vendor CLI or qualification tool when it is documented and noninteractive.
5. Service, process, scheduled task, marker file, or registry marker only when it represents installed state.

Never use `Win32_Product`.

## Custom Detection Script Contract

For Intune custom detection scripts:

- Return exit code `0` only when the target app/version is installed and healthy enough for the package requirement.
- Return nonzero when the target app/version is not installed, partially installed, the wrong version, or the detection signal is missing.
- Write concise output that explains the detected state, but do not print secrets, full registry exports, or long environment dumps.
- Use strict-mode-safe optional property reads for uninstall registry enumeration.
- Check both 64-bit and 32-bit uninstall registry paths when architecture is uncertain.
- Avoid user-profile-only locations unless the package intentionally installs per user.

Strict-mode-safe registry pattern:

```powershell
$nameProperty = $_.PSObject.Properties['DisplayName']
$versionProperty = $_.PSObject.Properties['DisplayVersion']
$displayName = if ($null -ne $nameProperty) { [string]$nameProperty.Value } else { '' }
$displayVersion = if ($null -ne $versionProperty) { [string]$versionProperty.Value } else { '' }
```

## Install Validation

After install validation, run the same detection method intended for Intune against the final installed state:

- Run detection after all post-install cleanup/configuration has finished.
- Confirm detection exits `0` and reports the expected product/version.
- Confirm the detection target is not a temporary installer file, package source file, shortcut-only artifact, or cleanup candidate.
- If wrapper validation passes but detection fails, report the mismatch as a packaging issue before proposing changes.

## Uninstall Validation

After uninstall validation, run the same detection method again:

- Expected result is nonzero/not detected.
- Confirm MSI ProductCode, file target, service, and uninstall registry entries are absent or no longer match the target version.
- Treat a remaining detection hit as uninstall failure or cleanup-scope issue.
- Ask before reinstalling the app after uninstall validation.

## Intune Rule Notes

- For MSI detection, record the ProductCode and whether the rule is version-specific.
- For file detection, record path, operator, version value, and whether file system redirection matters.
- For registry detection, record hive, key path, value name, comparison operator, and 32/64-bit context.
- For custom scripts, keep dependencies inside the package source or use built-in Windows/PowerShell capabilities.

## Failure Handling

If detection fails during validation:

1. Report the exact detection command, exit code, and concise output.
2. Compare wrapper validation evidence with Intune detection evidence.
3. Identify whether the issue is install failure, wrong detection target, cleanup removing the target, architecture mismatch, or strict-mode script failure.
4. Propose a fix.
5. Implement only after explicit user confirmation.
