# Build 73 — Device Install Confirmation

**Date:** 2026-04-29  
**From:** CC  
**Build:** 73 (Phase 0c-21 + voice button fix)

---

## Install Results

### iPad Pro 12.9" M2 — `8B58895C-4976-5DF7-AAA2-FD9EAEBB59F2` ✓

```
07:22:07  Acquired tunnel connection to device.
07:22:07  Enabling developer disk image services.
07:22:08  Acquired usage assertion.
App installed:
  bundleID: com.tramel.homecontrol.wallpanel
  installationURL: file:///private/var/containers/Bundle/Application/BA4917FD-5350-4EA8-96C2-ABED254A2BB7/WallPanel.app/
  databaseSequenceNumber: 1636
```

### iPad Air 13" M2 — `25423385-294F-50EA-8498-5690FECE21BF` ✓

```
07:22:12  Acquired tunnel connection to device.
07:22:12  Enabling developer disk image services.
07:22:13  Acquired usage assertion.
App installed:
  bundleID: com.tramel.homecontrol.wallpanel
  installationURL: file:///private/var/containers/Bundle/Application/DFA669A0-94FB-4C62-BC89-6AA53CE20697/WallPanel.app/
  databaseSequenceNumber: 2200
```

Both iPads were showing `tunnelState: disconnected` (sleeping) at install time. devicectl acquired tunnels successfully in both cases. Build 73 is live on both devices.

---

## UDID Correction

`deploy.sh` had an incorrect iPad Air UDID (`00008112-001669910E06601E` — shows as "not-found" in devicectl). Corrected to `25423385-294F-50EA-8498-5690FECE21BF` per DC's prompt. deploy.sh also updated to warn-but-continue (instead of hard exit) when tunnelState is "disconnected", since devicectl tunnels through reliably.

---

## Standing Workflow Change — Acknowledged

The mandatory phase completion checklist is now written to:  
`home-control/PHASE_COMPLETION_CHECKLIST.md`

Summary of the standing change:
- Install on both iPads is now a hard gate — completion report is not done until both installs confirm
- Install failures are reported honestly (not hidden)
- Relay posts use correct header format: metadata in `X-Kind` / `X-Title` / `X-From-Instance` headers, raw markdown body, `--data-binary @file.md`

From phase 0c-22 onward, every completion report will include explicit per-device install confirmation.
