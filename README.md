<img width="1920" height="1080" alt="image" src="https://github.com/user-attachments/assets/74fe959d-7039-44c7-9cf2-ce3a1006de46" /># OpenCore (Hackintosh) EFI for GIGABYTE GA-Z77P-D3

<img width="1280" height="720" alt="image" src="https://github.com/user-attachments/assets/d9757da6-503f-4bea-ba26-96743cc0f337" />

<img width="1280" height="720" alt="image" src="https://github.com/user-attachments/assets/56cee336-e572-432b-841d-bac709cdc760" />


macOS 10.9 Mavericks on Ivy Bridge desktop, no dGPU.

## Hardware

|---|---|
| Motherboard | GIGABYTE GA-Z77P-D3 (rev. 1.0/1.1), Z77 chipset |
| CPU | Intel Core i5-3330 (Ivy Bridge, no XCPM, no Turbo binning tricks needed) |
| iGPU | Intel HD Graphics 2500 (GT1, `0x0152`) |
| Audio | Realtek ALC887 |
| LAN | Realtek RTL8111E/F |
| Target OS | macOS 10.9 Mavericks only - not tested/intended for anything newer |

SMBIOS: `iMac13,2`. Bootloader: OpenCore 1.0.7, graphical picker (OpenCanopy).

## What works

- Full boot, install, boot from internal disk
- Audio (ALC887, layout-id 7)
- Ethernet (RTL8111)
- USB 2.0/3.0
- Sleep/wake (S3) - see notes below, needs one BIOS setting most people miss
- Native power management (no SSDT-PLUG needed for boot stability on this CPU/board combo, see below)

## What doesn't

- **No iGPU acceleration.** See "Notes on HD 2500 acceleration" below - this isn't a config gap, it was tried and reverted deliberately.
- No HDMI audio (follows from no iGPU acceleration - the audio device lives behind the same framebuffer driver)
- No native NVRAM writes past what `OpenVariableRuntimeDxe.efi` covers (kept disabled by default; enable only if you see NVRAM-related resets)

## Notes on HD 2500 acceleration

HD 2500 (GT1) has no display-capable framebuffer personality of its own in Apple's `AppleIntelFramebufferCapri.kext` - only 0-connector ("empty", IQSV-only) ones. The historically-documented workaround is spoofing `device-id`/`AAPL,ig-platform-id` to a real HD 4000 (GT2) id, since GT1 and GT2 dies of the same generation share the same display/scanout block - only the EU count differs. This genuinely works on some boards.

On this specific board it doesn't: both of Acidanthera's documented desktop personas for this case (`0x0166000A` and `0x0166000B`, connector layout 2×DP + 1×HDMI with the two phantom DP entries disabled via `framebuffer-con0/1-enable`, `stolenmem`/`fbmem` sized to match) load cleanly and pass POST, but corrupt (visual noise) as soon as the desktop starts compositing. Whether that's this board's specific DDI wiring, this specific monitor's EDID, or something else wasn't root-caused.

**Current config**: no `device-id`/`AAPL,ig-platform-id` injection at all, `WhateverGreen.kext` present but disabled. The real, unspoofed `0x0152` id loads cleanly on its own with no crashes. Desktop is composited via the firmware/GOP framebuffer - no acceleration, but stable. If you're on the same board and want to retry acceleration, both persona configs are worth trying again with a different monitor/cable before assuming it's hopeless.

Recommendation: if you need real 2D/3D performance on this class of board, a cheap dGPU with native 10.9 support (GT 610/630, Radeon HD 5450/6450) is the actual fix - the iGPU then runs headless (`0x01620007`) for QuickSync only, and you get real acceleration from the discrete card.

## Notes on sleep (S3)

Works, but two things commonly break it on Gigabyte boards of this generation if left at BIOS defaults:

- **ErP/EuP Support** must be **disabled** in BIOS. When enabled, the board cuts standby power to USB more aggressively than S3 resume tolerates on this chipset generation - symptom is "sleeps fine, never wakes, needs a hard reset."
- `darkwake=0` boot-arg. Without it, some Gigabyte boards wake immediately after entering sleep, or never fully settle into S3. Already set by default in this EFI's `boot-args`.

If wake is triggered spuriously by network traffic, also check **PME Event Wake Up** / **Wake on LAN** in BIOS - RTL8111 can trigger a resume off ordinary broadcast traffic on some switches.

## Notes on SSDT-PLUG (CPU power management)

Both `SSDT-PLUG-CPU0.aml` and `SSDT-PLUG-PR00.aml` are included in `ACPI/` but **disabled by default**. They inject `plugin-type=1` on the CPU ACPI object so `X86PlatformPlugin` picks up real power-management data instead of falling back on the generic path (you'll see `WARNING: IOPlatformPluginUtil: getCPUIDInfo: this is an unknown CPU model 0x3a` in verbose boot either way - harmless on this CPU/board combo, `AppleCpuPmCfgLock` quirk is what actually keeps boot stable here).

Enable **one** of the two, not both, only if you see actual symptoms (Turbo Boost not engaging, thermal/power irregularities under sustained load) - not just to silence the warning above. Which one depends on how the CPU object is named in this board's real DSDT (`_PR.CPU0` vs `_SB.PR00`); the wrong one is inert, not harmful.

## Issues & Tips

- `npci=0x2000` is in `boot-args`. Kept as a standard compatibility flag for this chipset generation; wasn't the fix for any specific bug encountered on this board, but doesn't cost anything either.
- `XhciPortLimit` is enabled - required on macOS builds older than 11.3 for USB personality matching on non-Apple boards, unrelated to the real 15-port kernel limit removed in 11.3+.
- `DisableRtcChecksum` is enabled - without it, this board's BIOS settings/RTC state get reset intermittently across reboots.
- `AppleCpuPmCfgLock` is enabled - this board's CFG-Lock (MSR 0xE2 write protection) can't reliably be disabled from BIOS; this quirk works around it whether or not the BIOS toggle is off. `ControlMsrE2` tool is included in the picker to check the actual MSR state if you want to verify.
- If ethernet doesn't come up: this board shipped across revisions with either a Realtek or an Atheros LAN chip depending on revision/batch. Check `About This Mac -> System Report -> Network` for what's actually detected before assuming the kext is wrong.

## Installation

Standard OpenCore install: copy `EFI/` to the root of your USB installer's EFI partition **and** to the target disk's EFI partition after install (they're independent - updating only one leaves the other stale on next boot from that device). If ethernet doesn't work, install old version of RealtekRTL8111 kext

BIOS: AHCI, CSM/Secure Boot disabled, VT-d disabled, DVMT Pre-Allocated ≥ 64M, XHCI/EHCI Hand-off enabled.

## Credits

- [Acidanthera](https://github.com/acidanthera) - OpenCorePkg, Lilu, VirtualSMC, WhateverGreen, AppleALC
- [Mieze](https://github.com/Mieze) - RealtekRTL8111
- [Dortania](https://github.com/dortania) - OpenCore Install Guide, Ivy Bridge reference config
