# OpenWrt on the EDUP AX3000 (CreaLentem CLT-R30B1)

The **EDUP AX3000** Wi-Fi 6 router sold on AliExpress is a rebrand of the
**CreaLentem CLT-R30B1** (China Telecom hardware, board `CLT-R30B1_0824_V1.1`),
which is shared by several diamond-shaped 5-antenna routers — including the
**EDUP RT2980**, **Dragonglass DXG21**, and **Livinet Li228**.

**This device is officially supported by OpenWrt** (target `mediatek/filogic`,
device `creatlentem_clt-r30b1`). Support landed in August 2025
([commit 2fea2ede](https://github.com/openwrt/openwrt/commit/2fea2ede6340650052cb9688bdea4592df7f551a),
PR [#19534](https://github.com/openwrt/openwrt/pull/19534) by andros-ua), with an
all-in-UBI layout added in June 2026. Any release from the **25.12 series** onward
includes it. **No custom patch or device tree is needed — use the official images.**

## Specifications

| | |
|---|---|
| SoC | MediaTek MT7981B, dual-core ARM Cortex-A53 @ 1.3 GHz |
| RAM | 256 MiB DDR3 |
| Flash | 128 MiB SPI-NAND (ESMT F50L1G41LB) |
| Wi-Fi | MediaTek MT7976CN, dual-band Wi-Fi 6 (2×2 2.4 GHz + 3×… 5 GHz) |
| Ethernet | 3× LAN GbE (MT7531AE switch) + 1× WAN GbE (internal PHY) |
| Buttons | Reset, Mesh/WPS |
| LEDs | Status (green / red) |
| Power | 12 V DC, 1 A (2.1×5.5 mm barrel) |
| Serial | UART pads on PCB, 115200n8 — pin order: RX, TX, GND, [3.3V] |

## Image variants — pick the one matching your flash layout

The device shipped in **two variants** that differ only in the size of the
`ubi` partition. **The layouts are not interchangeable** — flashing the wrong
one can leave the router without a bootable firmware (recoverable only over UART).

| Image | `ubi` partition size | Shipped with |
|---|---|---|
| `creatlentem_clt-r30b1` | 64 MiB (`0x4000000`) | Stock-style firmware, WaveLink/GL.iNet-style WebUI |
| `creatlentem_clt-r30b1-112m` | 112 MiB (`0x7000000`) | ImmortalWrt with LuCI |
| `creatlentem_clt-r30b1-ubi` | all-in-UBI, new bootloader | — (advanced, see below) |

### Identify your layout

If OpenWrt/ImmortalWrt is already running, check the `ubi` partition size:

```console
cat /proc/mtd
```

- `mtd5` (`ubi`) size `0x7000000` → **112M layout**, use the `clt-r30b1-112m` image
- `mtd5` (`ubi`) size `0x4000000` → **64M stock layout**, use the `clt-r30b1` image
- a single large `ubi` partition at offset `0x100000` (size `0x7f00000`) →
  all-in-UBI layout, use the `clt-r30b1-ubi` image

If the router still has its original firmware: the 112M variant ships with
ImmortalWrt/LuCI on it; the 64M variant has the stock WaveLink/GL.iNet-style
WebUI.

## Download

Official images (current stable series: 25.12):

- [Firmware Selector](https://firmware-selector.openwrt.org/?version=25.12.6&target=mediatek%2Ffilogic&id=creatlentem_clt-r30b1-112m)
  (change the version/device in the URL as needed, or search for "CLT-R30B1")
- Direct: <https://downloads.openwrt.org/releases/25.12.6/targets/mediatek/filogic/>
  — files named `openwrt-25.12.6-mediatek-filogic-creatlentem_clt-r30b1-*`

## Installation

### From OpenWrt/ImmortalWrt on the 112M layout (most common for EDUP units)

Use the standard sysupgrade procedure with the **`clt-r30b1-112m`** sysupgrade
image (LuCI: System → Backup/Flash Firmware, or):

```console
sysupgrade -n /tmp/openwrt-*-creatlentem_clt-r30b1-112m-squashfs-sysupgrade.bin
```

The official 112M image declares `clt,r30b1` and `clt,r30b1-112m` as supported
upgrade sources, so it also upgrades cleanly **from custom builds** that use the
`clt,r30b1` board name (e.g. hanwckf's immortalwrt-mt798x and builds derived
from it). All checks should pass — don't proceed if a "not supported" warning
is issued.

### From the 64M stock firmware

**WebUI method:**

1. Prepare the upgrade image with `clt-r30b1-squashfs-sysupgrade.bin` using
   [andros-ua's `make_staged_upgrade_tar.sh`](https://github.com/andros-ua/owrt-misc/tree/main/clt-r30b1),
   or use the pre-built `staged_openwrt_upgrade.bin` from the same repository.
2. Install it via the stock WebUI update page.
3. After reboot, press and hold **reset** to wipe the stock config and regain access.

**SSH method:**

1. SSH in with the stock credentials `dg:ivanlee`.
2. Upload `clt-r30b1-squashfs-sysupgrade.bin`.
3. Run `sysupgrade -n`.

### All-in-UBI variant (advanced)

The `clt-r30b1-ubi` image moves everything (including U-Boot) into one UBI
partition. Flashing it involves writing a new `preloader.bin` and
`bl31-uboot.fip` over the boot partitions — only attempt this over UART or from
a running system after reading the
[upstream commit](https://github.com/openwrt/openwrt/commit/e414ca4fa6cbc8d1e95d59966b614a4129b5371f)
that introduced it, and only if you understand what a preloader flash failure
means for recovery.

## Recovery

- **112M variant (newer U-Boot):** power on while holding the **mesh** button
  for ~3 seconds — a U-Boot flash WebUI appears at <http://192.168.1.1>.
- **Both variants:** connect UART (115200n8) and use the U-Boot menu to flash a
  firmware image or boot an OpenWrt initramfs image.

## Return to stock

Flash a backup of the `ubi` mtdblock (`mtd5`) via the OpenWrt sysupgrade method
(keep a backup before you start).

## History

This repository started as a hand-written port of the CLT-R30B1 to OpenWrt
24.10 (which predates upstream support). That work — including the earlier
build path via [hanwckf/immortalwrt-mt798x](https://github.com/hanwckf/immortalwrt-mt798x)
— is preserved for reference in [`history/`](history/) but is **obsolete**:
it contains known bugs and was never fully build-tested. Use official 25.12+
images instead.

## Credits

- **andros-ua** — the upstream OpenWrt port and device documentation
  ([owrt-misc](https://github.com/andros-ua/owrt-misc))
- **hanwckf** — immortalwrt-mt798x, which supported this device early on
