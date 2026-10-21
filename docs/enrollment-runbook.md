# Enrollment Runbook — Windows (Autopilot + manual fallback)

SOP for enrolling a corporate Windows device into Intune. Placeholders
throughout (`<serial>`, `<user>`); group names follow `docs/architecture.md`.

## Pre-enrollment checklist

- [ ] Intune license assigned to the user (or device license in place)
- [ ] Asset record created in Snipe-IT — asset tag and serial recorded
      **there**, never in this repo
- [ ] Autopilot enrollment profile `Autopilot-Corporate` exists and is
      assigned to the Autopilot device group
- [ ] Compliance and configuration policies assigned to the pilot / corporate
      groups
- [ ] Enrollment restrictions allow Windows and corporate devices
- [ ] User account active in Entra ID; MFA registered
- [ ] Device has network access for OOBE (corporate Wi-Fi or wired)

## Windows Autopilot flow

1. **Get the hardware hash.** Preferred: the vendor/reseller uploads it at
   purchase. Otherwise, on the device from an admin PowerShell, run the
   `Get-WindowsAutopilotInfo` script (PSGallery) to export the hardware-hash
   CSV.
2. **Import.** Intune admin center → Devices → Windows → Windows enrollment →
   Devices → Import. Upload the CSV (device serial, Windows product ID,
   hardware hash, optional group tag e.g. `CORP`).
3. **Wait for sync**, then confirm the device appears with profile status
   **Assigned**. Until a profile is assigned, the device will not recognize
   your tenant at OOBE.
4. **Group membership.** The Autopilot dynamic group (rule on the group tag in
   `devicePhysicalIds`) picks the device up automatically. The Autopilot
   profile itself is assigned to that same group.
5. **Hand the device to the user** (or ship it).
6. **OOBE.** User powers on → connects to a network → the device recognizes
   the tenant → signs in with their Entra ID work account + MFA → the
   Enrollment Status Page tracks *Device preparation → Device setup → Account
   setup* → desktop.
7. **First check-in.** Policies and required apps download after the first
   check-in; give it time, then run the verification steps below.

## Manual enrollment fallback

When no hash is available (board-swap repair, legacy machine) or for BYOD:

- Install Company Portal (Microsoft Store) → sign in with the work account →
  the device enrolls (work account + MDM).
- Or Settings → Accounts → Access work or school → Connect → sign in. This
  joins Entra ID and auto-enrolls via the MDM user scope.
- BYOD devices get the light profile set automatically via the
  `deviceOwnership = Personal` dynamic rule.
- **Confirm ownership classification** (Devices → device properties). A device
  misclassified as Personal lands in the BYOD group and misses corporate
  policies — flip it to Corporate when appropriate.

## Group targeting (verify)

- Intune → Devices → All devices → select the device → **Group membership**.
- Expect the corporate dynamic group to be present. Dynamic membership
  processing takes time — if the device is absent after a few hours, go to
  Entra ID → Groups → group → Dynamic membership rules → *Validate rules* and
  fix the syntax.

## Verification steps

- [ ] Device listed in Intune with the correct name and ownership
- [ ] Compliance state reads **Compliant** (device → Device compliance)
- [ ] Configuration profiles show **Succeeded**, no conflicts
      (device → Device configuration)
- [ ] Required apps report **Installed** (device → Managed apps)
- [ ] BitLocker on; recovery key visible in the Entra ID device record
- [ ] Update ring applied (device shows the expected ring policy)
- [ ] Snipe-IT record updated: assigned to user, status Deployed, Intune
      device name noted

## Common failures

| Symptom | Likely cause | Fix |
|---|---|---|
| Device boots to plain consumer OOBE, no tenant branding | Hash not imported, or imported into the wrong tenant | Find the serial in Autopilot devices; re-import the CSV into the correct tenant |
| Autopilot profile status **Not assigned** | Device isn't in the profile's target group | Check the group tag / dynamic rule; fix the rule, wait for membership processing, sync |
| OOBE error / stuck on the Enrollment Status Page | Network drop, or ESP blocked waiting on a failing required app | Reconnect network and retry; review ESP settings — don't block ESP on non-critical app installs |
| Enrollment fails with `0x80180014` | Enrollment restriction blocks the platform or personal devices | Review Devices → Enrollment restrictions; confirm the device is marked corporate |
| `0x8018000a` — device already enrolled | Stale MDM record from a previous enrollment | Retire/remove the old device record and old work account, then re-enroll |
| Device in Entra ID but not in Intune | MDM auto-enrollment scope doesn't include the user | Entra ID → Mobility (MDM) → Microsoft Intune: set the MDM user scope to cover the user; re-join |
| Compliance stuck at **Not evaluated** | Device hasn't checked in, or no compliance policy assigned to it | Sync from Company Portal (Settings → Sync); confirm policy assignment and group membership |
| BitLocker never turns on | TPM not ready, Home edition, or conflicting encryption present | Check TPM state, require Pro edition or higher, resolve third-party encryption, review profile errors |
| Configuration profile shows **Conflict** | Two profiles set the same setting to different values | Open the device's configuration report, find the conflicting source profiles, keep one value |
