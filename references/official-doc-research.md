# Official Documentation Research

Research official sources before finalizing wrapper behavior.

## Source Priority

1. Vendor enterprise deployment/admin guide for the exact product and version.
2. Vendor support article for silent install, unattended install, licensing, response files, or mass deployment.
3. Microsoft Learn for Intune Win32 behavior, PowerShell execution context, return codes, and detection.
4. Installer technology docs when vendor docs identify MSI, InstallShield, Inno Setup, NSIS, WiX Burn, or another bootstrapper.
5. Community posts only as clues. Label them unverified.

## Search Pattern

Use searches like:

```text
<vendor> <product> <version> silent install official
<vendor> <product> deployment guide unattended install
<vendor> <product> admin guide command line
<product> MSI properties license server silent install
site:<vendor-domain> <product> silent install
```

For Intune behavior, prefer:

- Microsoft Learn: `https://learn.microsoft.com/en-us/intune/app-management/deployment/add-win32`
- Microsoft Learn IME flow/logs: `https://learn.microsoft.com/en-us/troubleshoot/mem/intune/app-management/develop-deliver-working-win32-app-via-intune`
- Windows Installer error codes: `https://learn.microsoft.com/en-us/windows/win32/msi/error-codes`

## Facts To Extract

- Installer type and whether the EXE is a bootstrapper.
- Silent/no-restart/wait/log switches.
- Whether the installer creates log directories or needs the wrapper to create them.
- Required transforms, response files, license files, license server values, or config files.
- Architecture, install context, dependencies, and prerequisite order.
- Expected success, reboot, retry, and failure exit codes.
- Vendor-recommended validation method.

## If Official Docs Are Missing

State that official silent-install documentation was not found.

Then design a validation-first wrapper that:

- Logs that silent switches are unverified.
- Runs only a lab-confirmed command.
- Validates installation before returning success.
- Includes fallback notes such as extracting embedded MSI content, checking `setup.exe /?`, or requesting vendor deployment documentation.

Never present an unverified EXE switch as production-ready.
