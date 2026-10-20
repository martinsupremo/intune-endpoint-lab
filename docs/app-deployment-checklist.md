# Win32 App Deployment Checklist

Packaging and deploying Windows desktop apps via Intune. Placeholders:
`<AppName>`, `<vendor installer>`.

## 1. Prepare the package

- [ ] Obtain the vendor installer (`.msi` / `.exe`) from the vendor's official
      source; note the exact version
- [ ] Test the **silent install** manually on a clean VM and confirm the
      silent switches (MSI: `msiexec /i "app.msi" /qn /norestart`;
      typical EXE: `/quiet /norestart` or vendor-specific)
- [ ] Test the **silent uninstall** the same way
- [ ] Wrap with the Microsoft Win32 Content Prep Tool (`IntuneWinAppUtil.exe`):
      source folder → setup file → output folder → produces
      `<AppName>.intunewin`
- [ ] Keep the source and wrapped package in the app source library (not in
      this repo if the binaries are licensed)

## 2. Create the app in Intune

- [ ] Apps → Windows → Add → app type **Windows app (Win32)** → upload
      `.intunewin`
- [ ] Name / publisher / description filled in — this is what users see in
      Company Portal if the app is assigned as Available
- [ ] Program: exact **install command** and **uninstall command**
- [ ] Install behavior: System (machine-wide, the usual default) vs User;
      device restart behavior set deliberately (suppress for silent rollouts)
- [ ] Requirements: OS architecture (x64) and minimum OS version
- [ ] Return codes: map success (0) and soft-reboot (3010) as needed

## 3. Detection rules

- [ ] Rule type chosen: MSI product code (for MSI installers), File
      (path + file + version/date check), or Registry (key + value)
- [ ] Detection verified **both ways**: against a machine with the app
      installed AND one without
- [ ] Remember: Intune decides "Installed" by detection. The installer's exit
      code alone is not proof.

## 4. Assignment

- [ ] **Pilot group first, Required** — validate end-to-end on test devices
- [ ] Then broad group: **Required** (auto-install) vs **Available**
      (Company Portal self-service), chosen deliberately
- [ ] Exclusions reviewed — no group is both Required and Excluded
      unintentionally
- [ ] Rollout timing considered (user notifications, restart grace)

## 5. Monitor

- [ ] Apps → `<AppName>` → Device install status reviewed after the rollout
      window
- [ ] Failures sampled per device — common patterns: detection rule mismatch
      (app installed but reporting failed), installer not actually silent
      (hangs until timeout), Intune Management Extension not running
- [ ] Success rate meets the bar before widening the assignment

## 6. Troubleshoot (device side)

- [ ] Intune Management Extension service running
      (`services.msc` → Microsoft Intune Management Extension)
- [ ] Force a check-in: Company Portal → Settings → Sync
- [ ] IME logs: `C:\ProgramData\Microsoft\IntuneManagementExtension\Logs\`
      (`IntuneManagementExtension.log` — search by app ID)
- [ ] Check detection manually: does the file / registry value / product code
      actually exist after install?
- [ ] Reproduce the install command locally as SYSTEM (e.g. via PsExec
      `-s`) — many installers behave differently in system context than as an
      admin user
