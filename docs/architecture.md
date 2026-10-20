# Architecture & Design Decisions

## Enrollment strategy per platform

| Scenario | Method | Why |
|---|---|---|
| Corporate Windows, new purchase | Windows Autopilot, user-driven, Entra ID joined | Hardware hash pre-registered (by reseller or IT) → the device recognizes the tenant at first power-on; user signs in and provisioning runs. No imaging pipeline to maintain. |
| Corporate Windows, no hash available | Manual enrollment fallback (see enrollment runbook) | Repair board swaps change the hash, some legacy machines predate registration — Company Portal enrollment still gets them managed. |
| BYOD (any platform) | User enrolls via Company Portal | Light profile set only; the device is never subject to Wipe, so offboarding uses Retire (see decommission procedure). |
| Apple corporate | Automated Device Enrollment via Apple Business Manager (design note only) | Same group/compliance pattern as Windows; this lab's artifacts are Windows-focused. |
| Android corporate | Fully managed enrollment (design note only) | Corporate-owned devices get the full policy set, same as corporate Windows. |

Autopilot is the default for anything the company owns. Manual enrollment is
the fallback for exceptions and the standard path for BYOD.

## Dynamic device groups

Policies are never assigned to "All devices". Entra ID dynamic groups map a
device to the right policy set automatically at enrollment, so a device can
never be "forgotten" into an unmanaged state:

```text
# Corporate Windows (any enrollment path)
(device.deviceOwnership -eq "Company") and (device.deviceOSType -eq "Windows")

# Autopilot devices (using the Autopilot group tag in devicePhysicalIds)
(device.devicePhysicalIds -any (i: i -contains "[OrderID]:LAB-CORP"))

# BYOD Windows
(device.deviceOwnership -eq "Personal") and (device.deviceOSType -eq "Windows")
```

Group tags (`[OrderID]:...`) are placeholders — replace with your naming
convention. One more group stays manual: the **pilot group** of test devices
and IT staff. Every new policy assignment lands on the pilot group first and
only goes broad after validation.

## Compliance before access (Conditional Access)

The compliance policy (`policies/compliance-policy-windows.example.json`)
evaluates OS version, BitLocker, Secure Boot, firewall, Defender, and
password rules. Its scheduled actions notify the user at first failure and
block after a 72-hour grace period — those values are template defaults.

The key design point: **a compliance policy by itself blocks nothing**. Intune
writes the evaluation result to the device object in Entra ID, and a
Conditional Access policy with the "Require device to be marked as compliant"
grant control is what actually denies access to corporate apps. The compliance
policy is the measuring stick; Conditional Access is the enforcement.

Scope the CA policy to corporate apps (M365, VPN, internal tools) so the user
can still reach the remediation path — Company Portal, which shows exactly
which setting failed. A device that drifts out of compliance gets a fixable
prompt, not a dead laptop.

## Update rings

Windows Update for Business rings assigned per group:

| Ring | Audience | Quality update deferral | Feature update deferral |
|---|---|---|---|
| Pilot | IT + test devices | 0 days | 0 days |
| Broad | Most users | 7 days | 30 days |
| Slow | Shared / critical machines | 14 days | 90 days |

Deadlines keep laggards moving (e.g. quality deadline 5 days, grace 2 days).
Zero-days get an expedite policy outside the rings — the CVE/KB review decides
that, not the ring schedule.

## Why X over Y

- **Autopilot over golden-image deployment (MDT-style):** no image
  capture/patching treadmill; the OEM factory image plus Intune profiles
  converge the device. A replacement laptop can ship straight to the user's
  home office. Trade-off: less control over the base image — mitigated with a
  clean OEM image and debloat/configuration profiles.
- **Dynamic groups over manual assignment:** a technician forgetting to add a
  device to a group produces an unmanaged device with no policies. Dynamic
  rules can't forget; membership follows device attributes.
- **Compliance + Conditional Access over network-location trust:** access
  follows device health, not office geography — a patched laptop at a café
  passes, an unencrypted one inside the building doesn't.
- **Intune profiles over Group Policy** for cloud-managed devices: GPO needs
  domain line-of-sight; Intune profiles apply over any internet connection.
  (Hybrid estates can co-manage with Configuration Manager; this lab assumes
  cloud-native.)
- **Retire over Wipe for BYOD:** Retire removes only company data and the
  management profile. Wipe factory-resets the phone — legally and ethically
  off the table for a personally-owned device.

## Hardening notes

- BitLocker is **required** in the compliance policy (measured) **and**
  **enforced** in the configuration profile (pushed). Policy pushes it;
  compliance proves it.
- Recovery keys escrow to Entra ID automatically when encryption is
  enforced; the helpdesk verifies escrow before approving any wipe (see the
  decommission procedure).
- Defender real-time + behavior monitoring + cloud protection are on via
  OMA-URI; pair with tamper protection so users can't disable AV locally.
- Removable storage: write access denied (config profile) — the practical
  control against casual USB data exfiltration.
- Admin hygiene: policy work under Intune Administrator (or a scoped custom
  role) with scope tags per site/team; changes always go pilot → broad, never
  straight to production groups.
