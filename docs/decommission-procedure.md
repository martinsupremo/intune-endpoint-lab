# Decommission Procedure — Retire / Wipe (GDPR-aware)

SOP for device end-of-life and offboarding. The asset system of record is
Snipe-IT. Real serial numbers live in Snipe-IT and tickets — never in this repo.

## 1. Decide: Retire, Wipe, or Autopilot Reset

| Action | What it does | Use when |
|---|---|---|
| **Retire** | Removes company apps/data + the management profile; personal data and the OS stay usable | BYOD offboarding; device the user keeps |
| **Wipe** | Factory reset to OOBE; all data removed | Corporate device lost, donated, sold, or disposed of |
| **Autopilot Reset** | Factory reset but keeps Entra join + enrollment for fast re-provisioning | Corporate device reassigned internally to a new user |

## 2. Pre-wipe checks

- [ ] Ticket opened; owner/manager approval recorded
- [ ] Asset verified in Snipe-IT: asset tag, serial, and assigned user match
      the physical device
- [ ] User data handled: OneDrive known-folder sync confirmed complete, or the
      user confirms nothing needs keeping — **a wipe is irreversible**
- [ ] BitLocker recovery key escrow verified in the Entra ID device record —
      this is the last chance to recover anything
- [ ] For resale/donation: note that the Autopilot registration must be
      removed before handover (see step 3), or the buyer's OOBE will pull
      your tenant branding

## 3. Execute

- [ ] Intune → Devices → select the device → **Wipe** (corporate disposal) or
      **Retire** (BYOD). For disposal use a plain Wipe — do **not** keep the
      enrollment state or the user account.
- [ ] Confirm the device checks in and completes the action (status updates in
      the admin center; the device returns to OOBE or goes permanently
      offline)
- [ ] Remove leftover objects, in order:
      1. Intune device record
      2. Entra ID device object
      3. Autopilot device entry (Devices → Windows enrollment → Devices →
         Delete) — this deregisters the hardware hash from the tenant

## 4. Close the record

- [ ] Snipe-IT: status → Retired/Disposed, date, method
      (Wipe / Retire / physical destruction), ticket link
- [ ] Chain-of-custody log entry completed (template below)
- [ ] Physical destruction by a vendor: obtain a **certificate of destruction**
      listing the serial; file it with the ticket

## Chain-of-custody log template

Keep serial numbers in Snipe-IT and on the certificate — this log carries the
asset tag plus a Snipe-IT reference only.

| Date | Asset tag | Serial (see Snipe-IT) | Action | Method | Performed by | Verified by | Certificate # |
|---|---|---|---|---|---|---|---|
| `<date>` | `<tag>` | ref: Snipe-IT asset record | Wipe | Intune remote wipe | `<name>` | `<name>` | n/a |
| `<date>` | `<tag>` | ref: Snipe-IT asset record | Destroyed | Vendor secure destruction | `<vendor>` | `<name>` | `<cert #>` |

## GDPR notes (practical, not legal advice)

- **Erasure (Art. 17):** a verified Intune Wipe plus deregistration is the
  technical execution of erasure for device-resident personal data; the log
  and the certificate are the evidence it happened.
- **Storage limitation (Art. 5(1)(e)):** don't keep decommissioned devices —
  or their data — "just in case". Wipe on schedule, not months later on
  request.
- **Records of processing (Art. 30):** Snipe-IT history plus this log document
  what was done, when, and by whom.
- **Security of processing (Art. 32):** BitLocker at rest means a lost laptop
  awaiting disposal is still encrypted; chain of custody covers the gap
  between collection and destruction.
- **Processors (Art. 28):** disposal vendors are data processors — contract
  plus per-device certificates, not a bulk receipt.

The DPO owns interpretation; this procedure owns the operational steps.
