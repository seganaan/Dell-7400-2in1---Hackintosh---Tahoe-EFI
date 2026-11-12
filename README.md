# Dell Latitude 7400 2-in-1 — Hackintosh OpenCore EFI (macOS Tahoe)

An OpenCore EFI built from scratch for the **Dell Latitude 7400 2-in-1** (Intel Core i7, Intel UHD 620 graphics), targeting **macOS Tahoe (26)**.

> ℹ️ Tahoe is the last macOS release that supports Intel Macs, so this is likely the final macOS generation this laptop can run.

> ⚠️ **Bluetooth is not working yet** on this EFI. See [Known issues](#known-issues).

> 🔐 **Use your own SMBIOS.** The serial, MLB, SmUUID and ROM in `config.plist` are placeholders. Generate your own with GenSMBIOS / OCAT before booting. The placeholder ROM `<data>` value is not valid base64, so some plist editors may complain until you replace it.

Looking for Ventura or Sonoma? See the companion repo: [Dell-7400-2in1---Hackintosh---Sonoma-x-Ventura-EFI](https://github.com/seganaan/Dell-7400-2in1---Hackintosh---Sonoma-x-Ventura-EFI).

---

## Repository layout

| Path | Purpose |
|------|---------|
| `BOOT/` | `BOOTx64.efi`, the OpenCore bootstrap loaded by the firmware |
| `OC/` | OpenCore itself: `config.plist`, ACPI SSDTs, Drivers, Kexts, Tools, OpenCore.efi |
| `Dell/logs/` | Debug logs captured while building and verifying the EFI |
| `.DS_Store`, `.gitattributes` | Housekeeping files |

---

## System summary

| Item | Value |
|------|-------|
| Target OS | macOS Tahoe (26) |
| Emulated Mac | `MacBookPro16,2` (Tahoe-supported Intel model) |
| iGPU | Intel UHD 620, spoofed as UHD 630 mobile (`AAPL,ig-platform-id` = `0x3E9B0000`, `device-id` = `0x3E9B`) |
| Internal panel link rate | `dpcd-max-link-rate` = `0x0A` (2.7 Gbps) |
| Boot picker | OpenCore built-in picker, 5 s timeout, auxiliary entries hidden (press Space to show them) |
| SIP | Enabled (`csr-active-config` = `00000000`) |
| Secure Boot Model | Disabled |
| OpenCore profile | Debug-style (`AppleDebug`, `ApplePanic`, `Target = 67`) |

---

## Component status

The status below comes from what is enabled in `config.plist` plus the author's testing.

| Component | Status | How it's handled |
|-----------|:------:|------------------|
| **Graphics acceleration** (UHD 620) | ✅ | WhateverGreen + device-id / platform-id spoof |
| **Internal display + brightness control** | ✅ | `SSDT-PNLF` + WhateverGreen backlight fix (see [Backlight options](#backlight-boot-options)) |
| **Wi-Fi** | ✅ | `itlwm` (connect with the **HeliPort** menu-bar app) |
| **Audio** | ✅ | `AppleALC` (`alcid=77`) and `SimpleHDA` (`shda-lang=en`) are both present in the config |
| **Keyboard** | ✅ | `VoodooPS2Controller` + `VoodooPS2Keyboard` |
| **Trackpad / touch input** | ✅ | `VoodooI2C`, `VoodooI2CHID`, `VoodooGPIO`, `VoodooI2CServices`, `VoodooInput` |
| **PS/2 mouse & trackpad fallback** | ✅ | `VoodooPS2Mouse`, `VoodooPS2Trackpad` |
| **Battery status** | ✅ | `VirtualSMC` + `SMCBatteryManager` |
| **CPU sensors / temperatures** | ✅ | `SMCProcessor` |
| **NVMe SSD** | ✅ | `NVMeFix` |
| **Sleep / wake** | ✅ | `SSDT-GPRW` (+ `GPRW → XPRW` patch), `HibernationFixup`, `SSDT-EC-USBX-LAPTOP` |
| **Embedded controller** | ✅ | `SSDT-EC-USBX-LAPTOP` |
| **CPU power management** | ✅ | `SSDT-PLUG-DRTNIA` (XCPM). `CPUFriend` is **not** included in this EFI |
| **Bluetooth** | ❌ | **Not working, no fix found yet** |

> Not covered by this EFI: USB port map kext, Thunderbolt, fingerprint reader, SD card reader, webcam-specific fixes.

---

## Known issues

### Bluetooth does not work
- No working Bluetooth setup has been found for Tahoe on this machine yet.
- Using **BlueToolFixup** together with the regular Intel Bluetooth stack (**IntelBluetoothFirmware + IntelBTPatcher**) causes a **crash**. Because of that, none of these kexts are included in this EFI.
- `-btlfxallowanyaddr` is still present in `boot-args` but does nothing without BlueToolFixup. It is harmless.
- Workaround: a supported **USB Bluetooth dongle** is the usual fallback, but compatibility depends on its chipset and is untested here.
- If you find a fix, please open an issue or pull request.

### Audio kexts overlap
`AppleALC` and `SimpleHDA` are both enabled. If you hear no sound or get odd audio behaviour, try disabling one of them and test again.

### Wi-Fi uses `itlwm`
`itlwm` does not show up as native Wi-Fi in System Settings. Install and run **HeliPort** to join networks.

---

## Backlight boot options

The `SSDT-PNLF` table provides the backlight device. On top of that, WhateverGreen offers backlight fixes for Coffee/Whiskey Lake laptops, controlled by boot-args in:

`OC/config.plist → NVRAM → Add → 7C436110-AB2A-4BBB-A880-FE41995C9F82 → boot-args`

| Boot-arg | What it does | When to use it |
|----------|--------------|----------------|
| `-igfxblr` | Backlight Registers Fix for KBL / CFL / ICL | **Shipped default in this EFI** |
| `-igfxblt` | Alternative Backlight Registers Fix for mobile Coffee Lake on macOS 13.4+ (WhateverGreen 1.6.5+). Fixes the ~3-minute dark screen | Use this **instead of** `-igfxblr` if the screen stays dark or the backlight comes on late after the Apple logo / boot |
| `-igfxbls` | Backlight Smoother, for smoother brightness transitions | Optional extra, see the WhateverGreen manual |

### Delayed backlight after the Apple logo / boot?
Swap `-igfxblr` for `-igfxblt`:

```diff
- keepsyms=1 debug=0x100 igfxonln=1 -igfxmlr -igfxblr -btlfxallowanyaddr alcid=77 -lilubetaall shda-lang=en
+ keepsyms=1 debug=0x100 igfxonln=1 -igfxmlr -igfxblt -btlfxallowanyaddr alcid=77 -lilubetaall shda-lang=en
```

Tips:
- Use `-igfxblr` **or** `-igfxblt`, not both.
- Make sure WhateverGreen is **1.6.5 or newer** (Tahoe support constants arrived in **1.7.0**).
- After editing, reset NVRAM from the OpenCore picker (`ResetNvramEntry`) so the new boot-args apply.

---

## Boot arguments

```
keepsyms=1 debug=0x100 igfxonln=1 -igfxmlr -igfxblr -btlfxallowanyaddr alcid=77 -lilubetaall shda-lang=en
```

| Argument | Meaning |
|----------|---------|
| `keepsyms=1` | Keep kernel symbols in panic logs |
| `debug=0x100` | Don't auto-reboot on a kernel panic, so you can read it |
| `igfxonln=1` | WhateverGreen: force all displays to report as online |
| `-igfxmlr` | WhateverGreen: DPCD maximum link rate fix (pairs with `dpcd-max-link-rate`) |
| `-igfxblr` | WhateverGreen: backlight registers fix (swap for `-igfxblt` if needed, see above) |
| `-btlfxallowanyaddr` | BlueToolFixup option. No effect while BlueToolFixup isn't installed |
| `alcid=77` | AppleALC audio layout ID 77 |
| `-lilubetaall` | Lets Lilu and its plugins load on macOS versions they don't officially list yet |
| `shda-lang=en` | SimpleHDA language setting |

---

## What each section of `config.plist` does

**ACPI**
- `SSDT-EC-USBX-LAPTOP`: fake embedded controller and USB power properties.
- `SSDT-PLUG-DRTNIA`: native Intel CPU power management (XCPM).
- `SSDT-PNLF`: backlight device for brightness control.
- `SSDT-GPRW` + `GPRW → XPRW` patch: prevents instant wake / USB wake problems.
- `ResetLogoStatus` quirk enabled.

**Booter**
- Coffee/Whiskey Lake laptop quirk set: `AvoidRuntimeDefrag`, `EnableSafeModeSlide`, `ProvideCustomSlide`, `RebuildAppleMemoryMap`, `SetupVirtualMap`, `SyncRuntimePermissions`, `FixupAppleEfiImages`.

**DeviceProperties** (iGPU at `PciRoot(0x0)/Pci(0x2,0x0)`)
- `AAPL,ig-platform-id`, `device-id` spoof and `dpcd-max-link-rate` for the built-in eDP panel.

**Kernel**
- Enabled kexts: Lilu, VirtualSMC, VoodooPS2 (Keyboard, Mouse, Trackpad), WhateverGreen, VoodooI2C stack (VoodooGPIO, VoodooI2CServices, VoodooInput, VoodooI2C, VoodooI2CHID), AppleALC, SMCBatteryManager, SMCProcessor, NVMeFix, HibernationFixup, itlwm, SimpleHDA.
- Quirks: `AppleXcpmCfgLock` (CFG-Lock is not disabled in BIOS, this works around it), `DisableIoMapper`, `DisableLinkeditJettison`, `PanicNoKextDump`, `PowerTimeoutKernelPanic`.
- The `IOSkywalkFamily` block entry and other sample patches exist but are **disabled**.

**Misc**
- Picker visible, `HibernateMode = Auto`, `DmgLoading = Signed`, `Vault = Optional`, `BlacklistAppleUpdate` on, `SecureBootModel = Disabled`.
- Many OpenCore **Tools** are enabled (OpenShell, CleanNvram, CsrUtil, ControlMsrE2, ResetSystem, and more) for debugging and recovery. You can trim these once everything is stable.

**NVRAM**
- Holds the `boot-args`, `run-efi-updater = No`, SIP enabled, and the usual keyboard-language and audio-volume variables.

**PlatformInfo**
- Automatic SMBIOS generation as `MacBookPro16,2`, `UpdateSMBIOSMode = Create`. **Replace the placeholder serial values with your own.**

**UEFI**
- Drivers: `HFSPlus.efi`, `OpenRuntime.efi`, `ResetNvramEntry.efi`.
- `ConnectDrivers`, `ReleaseUsbOwnership`, `RequestBootVarRouting`, `EnableVectorAcceleration` on, keyboard support on.
- One `ReservedMemory` region (4 KB, `RuntimeCode`) is enabled as a black-screen-after-hibernation workaround.

---

## Other troubleshooting

- **Boot loop / panic:** the config is debug-flavoured. Check `Dell/logs/` and the OpenCore log written to the EFI partition.
- **Dark screen or wrong brightness:** see [Backlight boot options](#backlight-boot-options).
- **No Wi-Fi:** make sure `itlwm` is up to date and HeliPort is running.
- **Settings not applying:** reset NVRAM from the picker after any `boot-args` change.

---

## Credits

[Acidanthera](https://github.com/acidanthera) (OpenCore, Lilu, WhateverGreen, VirtualSMC, AppleALC, NVMeFix, HibernationFixup), [OpenIntelWireless](https://github.com/OpenIntelWireless) (itlwm, HeliPort), the VoodooI2C and VoodooPS2 maintainers, and the Dortania OpenCore guide.

## Disclaimer

Provided as-is for the specific hardware above. Back up your own EFI before using it, and always use your own SMBIOS data.
