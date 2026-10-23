# Hackintosh EFI — i7-12700F + RX 6800 XT (iMacPro1,1)

OpenCore EFI backup for a desktop Hackintosh running **macOS 15.8.1 (24H32)**.

> **Before you use this:** every SMBIOS identity field in `EFI/OC/config.plist` is a
> placeholder. Generate your own with [GenSMBIOS](https://github.com/corpnewt/GenSMBIOS)
> and never reuse someone else's serial, MLB, or UUID.

| Field | Placeholder |
|---|---|
| `SystemSerialNumber` | `{REPLACE_SERIAL}` |
| `MLB` | `{REPLACE_MLB}` |
| `SystemUUID` | `DEADBEEF-DEAD-BEEF-DEAD-BEEFDEADBEEF` |
| `ROM` | `DE AD BE EF 00 00` (6 bytes) |

The serial and MLB placeholders are deliberately not valid Apple identities. The UUID
keeps a valid UUID shape (`DEADBEEF…`) instead of curly braces, because some tooling
parses that field as a UUID and would choke on a non-hex string. The machine still boots
with these values, but iMessage/FaceTime/App Store will not work until you generate a
real SMBIOS set. No real serial number, MLB, UUID, or MAC address is present in this
repository.

## Hardware

| Component | Model |
|---|---|
| CPU | Intel Core i7-12700F (Alder Lake, 8 P-cores + 4 E-cores, 20 threads, **no iGPU**) |
| GPU | AMD Radeon RX 6800 XT 16 GB (Navi 21, `1002:73BF`) — Yeston / WEIKENG NAVI21EXT D412 |
| RAM | 96 GB DDR4-3200 |
| Ethernet | Realtek RTL8125 (`RTL812xLucy.kext`) |
| Wi-Fi | Intel Wi-Fi via `AirportItlwm.kext` 2.4.0 (native macOS Wi-Fi, no HeliPort) |
| Bluetooth | Intel Bluetooth via `IntelBluetoothFirmware` + `BlueToolFixup` + `IntelBTPatcher` |
| Audio | `AppleALC.kext`, `layout-id = 12` |
| Bootloader | OpenCore REL-109 (1.0.9), SMBIOS `iMacPro1,1` |

## What works

- Metal 3 GPU acceleration (RX 6800 XT)
- Native Wi-Fi (AirportItlwm) and Bluetooth
- Audio
- USB ports (mapped with USBToolBox / UTBMap)
- S3 sleep and wake
- GPU temperature and power monitoring (`RadeonSensor` + `SMCRadeonGPU`)
- Custom GPU fan curve via a SoftPowerPlayTable (see below)

## Notable configuration

**Sleep is S3-only.** `hibernatemode 0`, no `sleepimage`. S3 is stable on this board;
S4/hibernate is intentionally not configured.

**USB wake is disabled.** With USB wake enabled, this machine woke up 1–26 seconds
after entering sleep (`Wake from Normal Sleep ... due to XDCI/HID Activity`) because
the 2.4G receiver, a USB 2.0 hub, and other devices kept signalling on the bus. After
disabling USB wake, sleep stays in S3 until the power button is pressed. Keyboard and
mouse cannot wake the machine — this is a deliberate trade for sleep stability.

**GPU fan curve / power table.** `PP_PhmSoftPowerPlayTable` is injected under
`DeviceProperties` for the GPU PCI path. It disables the factory zero-RPM mode and
lowers the fan start/stop temperatures. The full change record and the offsets used are
in [`EFI/FAN-CURVE-PPT.txt`](EFI/FAN-CURVE-PPT.txt). This table is specific to this
card and VBIOS build (`NAVI21EXT Gaming XT D412`, ATOMBIOS `VER020.001.000.071.000000`);
remove the GPU `DeviceProperties` entry if you use a different card. It is a soft table,
not a VBIOS flash, and is fully reversible.

**boot-args:** `-v debug=0x100 keepsyms=1 agdpmod=pikera ctrsmt=full itlwm_he=1`

## Kexts

| Kext | Version | | Kext | Version |
|---|---|---|---|---|
| AirportItlwm | 2.4.0 | | RadeonSensor | 0.3.3 |
| AppleALC | 1.9.9 | | RestrictEvents | 1.1.7 |
| BlueToolFixup | 2.7.3 | | RTL812xLucy | 1.1.1 |
| CpuTopologyRebuild | 2.0.2 | | SMCProcessor | 1.3.9 |
| IntelBluetoothFirmware | 2.5.0 | | SMCRadeonGPU | 0.3.3 |
| IntelBTPatcher | 2.5.0 | | SMCSuperIO | 1.3.9 |
| Lilu | 1.7.3 | | USBToolBox | 1.2.0 |
| NVMeFix | 1.1.4 | | UTBDefault | 1.0 |
| VirtualSMC | 1.3.9 | | UTBMap | 1.1 |
| WhateverGreen | 1.7.2 | | XHCI-unsupported | 0.9.2 |

`UTBDefault.kext` is present but disabled in `config.plist`.

## Benchmarks

Geekbench 7.1.0 for macOS AVX2, macOS 15.8.1 (24H32):

| Test | Score | Result |
|---|---|---|
| CPU single-core | **2219** | https://browser.geekbench.com/v7/cpu/701753 |
| CPU multi-core | **13134** | https://browser.geekbench.com/v7/cpu/701753 |
| GPU Metal | **202470** | https://browser.geekbench.com/v7/gpu/319027 |
| GPU OpenCL | **110274** | https://browser.geekbench.com/v7/gpu/319186 |

Per-workload scores are in [`benchmarks/geekbench7.md`](benchmarks/geekbench7.md).

## Repository layout

```
EFI/
  BOOT/BOOTx64.efi
  OC/                 OpenCore config, ACPI, drivers, kexts, resources
  FAN-CURVE-PPT.txt   GPU PowerPlay / fan table change record
benchmarks/
  geekbench7.md       Geekbench 7 results (CPU + GPU)
```

## Restoring

1. Mount the EFI system partition of the target disk.
2. Copy the `EFI` folder to the root of that partition (back up the existing one first).
3. Replace the SMBIOS placeholders in `EFI/OC/config.plist` with values generated for
   your own hardware.
4. Remove the GPU `DeviceProperties` entry if your graphics card is not the same model.
5. Adjust `boot-args` (drop `-v` and `debug=0x100` for normal use).

If the machine boots to a black screen after a GPU-related change, keep a bootable USB
with a known-good EFI on hand. This CPU has no iGPU, so there is no fallback display.

## Credits

OpenCore, Lilu, VirtualSMC, WhateverGreen, AppleALC — [Acidanthera](https://github.com/acidanthera).
AirportItlwm — [OpenIntelWireless](https://github.com/OpenIntelWireless/itlwm).
USBToolBox — [USBToolBox](https://github.com/USBToolBox).
SMCRadeonSensors / RadeonSensor — ChefKiss / Aluveitie.
PowerPlay extraction — `upp` (sibradzic) and the write-ups by perez987.

Third-party kexts and drivers keep their own licenses. The configuration and notes in
this repository are provided as-is, with no warranty.
