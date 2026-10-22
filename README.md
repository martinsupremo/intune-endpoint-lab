# Intune Endpoint Implementation — Full Device Lifecycle with Microsoft Intune

I run the full device lifecycle at Kime Capital with Intune and Autopilot, from a
new laptop in the box to wiping it when someone leaves. These are the templates and
checklists behind that work, with every company detail swapped for placeholders.

It covers Autopilot enrollment, dynamic group targeting, compliance-gated access,
Win32 app deployment, and GDPR-aware decommissioning.

**What this demonstrates:** Windows Autopilot provisioning, Entra ID dynamic
device groups, compliance policies paired with Conditional Access, OMA-URI
configuration profiles, Win32 app packaging with detection rules, update
rings, and retire/wipe decommissioning with an asset trail in Snipe-IT.

```
          ┌─────────────────────────────────────────────────┐
          │          MICROSOFT INTUNE (cloud)                 │
          │  ┌────────────────┐  ┌────────────────────────┐   │
          │  │  Compliance    │  │  Configuration         │   │
          │  │  policies      │  │  profiles (OMA-URI)    │   │
          │  └────────────────┘  └────────────────────────┘   │
          │  ┌────────────────┐  ┌────────────────────────┐   │
          │  │  App           │  │  Update rings +        │   │
          │  │  deployment    │  │  Conditional Access    │   │
          │  └────────────────┘  └────────────────────────┘   │
          └───────────────────────┬───────────────────────────┘
                                  │ enrollment
              ┌───────────────────┼───────────────────┐
              ▼                   ▼                   ▼
     Autopilot (Windows)   Manual / Company    BYOD (light
       (corporate new)      Portal (fallback)   profiles only)
              │                   │                   │
              └───────────────────┴───────────────────┘
                                  ▼
                  Dynamic device groups (Entra ID)
               grp-win-corporate / grp-win-byod / pilot
                                  │
                                  ▼
              Compliance check ──not compliant──► CA blocks
                  │ compliant       (notify now, block after grace)
                  ▼
           Apps + config delivered, update rings patch
                                  │
                                  ▼
           End of life: Retire (BYOD) / Wipe (corporate)
              → remove Entra ID + Autopilot → Snipe-IT closed
```

## Repo layout

| Path | Purpose |
|---|---|
| `policies/compliance-policy-windows.example.json` | Windows compliance policy template (Graph-shaped, placeholder IDs) |
| `policies/configuration-profile-windows.example.json` | OMA-URI configuration profile template (BitLocker, removable storage, Defender, Wi-Fi placeholder) |
| `docs/architecture.md` | Design decisions: enrollment, groups, compliance + CA, update rings |
| `docs/enrollment-runbook.md` | Enrollment SOP: checklist, Autopilot flow, manual fallback, verification, failure table |
| `docs/app-deployment-checklist.md` | Win32 app deployment: packaging, detection, assignment, monitoring, troubleshooting |
| `docs/decommission-procedure.md` | Retire/wipe SOP with GDPR notes + chain-of-custody log template |
| `.gitignore` | Keeps real policy exports, hardware-hash CSVs, and logs out of version control |

## Quick start

1. Sign in to the Microsoft Intune admin center (`intune.microsoft.com`) with
   an Intune Administrator account in a test tenant with Intune licenses.
2. Create the Entra ID dynamic groups from `docs/architecture.md` (corporate,
   BYOD, pilot).
3. Recreate the policies from the JSON templates: Devices → Windows →
   Compliance policies → Create (Windows 10 and later), using
   `policies/compliance-policy-windows.example.json` as the settings
   reference; Devices → Windows → Configuration profiles → Create → Custom
   (OMA-URI) from `policies/configuration-profile-windows.example.json`.
   (In a test environment you can also POST them via Microsoft Graph to
   `deviceManagement/deviceCompliancePolicies` / `deviceConfigurations`.)
4. Assign both policies to the **pilot group first**, never straight to
   production.
5. Enroll a test device with Autopilot following
   `docs/enrollment-runbook.md`.
6. Verify: device reads Compliant, profiles show Succeeded, then deploy a test
   Win32 app with `docs/app-deployment-checklist.md`.
7. Practice the exit: run the retire/wipe procedure in
   `docs/decommission-procedure.md` on the test device.

## Security notes

- Templates only: placeholder GUIDs (`00000000-...`), no tenant IDs,
  serials, hostnames, SSIDs, or credentials anywhere in this repo.
- Hardware-hash CSVs contain serial numbers — gitignored. Real exported
  policy JSONs — gitignored. Only `*.example.json` templates are committed.
- A compliance policy by itself blocks nothing — Conditional Access enforces
  it. The template's 72-hour block grace is a starting point, not a verdict.
- BitLocker recovery keys escrow to Entra ID when encryption is enforced;
  verify escrow before any wipe, or recovery is impossible.
- Least privilege: do policy/enrollment work with Intune Administrator (or a
  scoped custom role) plus scope tags — never a global admin daily driver.

## Testing

Both JSON templates are valid JSON, and their field names follow the Microsoft
Graph payloads for `windows10CompliancePolicy` and
`windows10CustomConfiguration` (OMA-URI). The runbooks and tables were reviewed
by hand, and the diagram is illustrative.

These exact files haven't been imported into a tenant yet. Treat every value as
a starting point: check it against the settings catalog in the Intune admin
center, and assign it to a pilot group before anything wider.
