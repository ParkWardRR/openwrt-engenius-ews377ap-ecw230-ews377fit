# Which firmware for which hardware

One OpenWrt image per **SKU**, not per hardware revision. The three SKUs are the same IPQ8072A / `ap-hk07`
board but each image carries its own device tree (model string, Wi-Fi calibration *variant*) and its own
Wi-Fi board file. **Flash the file named for your SKU.**

| Your AP | Files to use (release `v0.5.1`) | Status |
|---|---|---|
| **EnGenius EWS377AP v3** | `…-engenius_ews377ap-v3-squashfs-factory.ubi` (first install), `…-sysupgrade.bin` (later), `…-initramfs-uImage.itb` (RAM test) | hardware-proven (1 GiB unit, earlier builds); v0.5 per-SKU rebuild not re-run on hardware |
| **EnGenius EWS377-FIT** | `…-engenius_ews377-fit-…` (same three) | **hardware-proven 2026-10-06** on a 512 MiB / u-boot 2.1.0 unit — [report](ews377-fit-hardware-validation.md) |
| **EnGenius ECW230v3** | `…-engenius_ecw230v3-…` (same three) | **untested on hardware** — see [ECW230v3 notes](#ecw230v3-notes) |
| NETGEAR WAX218 | use the official OpenWrt image | same board, [officially supported](wax218-equivalence.md) |

## Hardware revisions we have seen (same image, different procedure)

| | EWS377AP v3 unit | EWS377-FIT unit |
|---|---|---|
| RAM | 1 GiB | 512 MiB |
| u-boot | 2.0.0 — `=>` prompt, 5 s "press a key" countdown | 2.1.0 (2022) — `IPQ807x#`, **boot menu: press `4`** |
| MAC source | ART | u-boot env `ethaddr` (+ `cert` partition); ART has placeholders |
| Backup reads | one 111 MiB `nand read` worked | **chunks ≤ 32 MiB** (a 111 MiB read overwrites u-boot's FDT and resets) |

RAM size and u-boot version do **not** need a different image: the kernel takes memory from the bootloader and
both bootloaders boot FIT config `config@hk07`. This was checked by RAM-booting the EWS377AP v3 initramfs on the
512 MiB FIT unit (it booted, saw 512 MiB, brought up NSS and both radios). Only the **install procedure** differs
— follow the callouts in [install-and-restore.md](install-and-restore.md).

## Do not cross-flash SKU images (unless you know why)

The Wi-Fi board file is matched by the *variant string* in the image's own device tree. Within one image they always
agree; mixing them does not:

- An image whose DTS asks for `EnGenius-EWS377-FIT` but ships the EWS377AP v3 board file boots fine and has **no
  radios** (`failed to fetch board data … variant=EnGenius-EWS377-FIT`). That was a v0.4-era build bug, fixed in v0.5
  by building each SKU on its own.
- The three board files differ only slightly (EWS377-FIT vs v3: 56 bytes), but "slightly" has not been shown to be
  "irrelevant" for RF calibration targets — use your own SKU's file.

## How to tell which unit you have

1. The label / stock web UI model name (EWS377AP v3, ECW230v3, EWS377-FIT).
2. On UART at boot: the u-boot banner (`U-Boot 2016.01-EWS377AP-FIT-uboot_version:V2.1.0 …`), the `DRAM:` line, and
   on `printenv`: `hw_id`, `hw_ver`, `machid`, `ethaddr`. (Observed FIT unit: `hw_id=0101012B`, `machid=8010006`.)
3. `cat /proc/mtd` on stock firmware (see the label warning below).

## ECW230v3 notes

No ECW230v3 unit has been tested. Known differences from the other two SKUs, from the stock firmware analysis
([`sibling-sku-firmware-analysis.md`](https://github.com/ParkWardRR/openwrt-nss-edma/blob/ews377ap-v3/ews377ap-v3-port/sibling-sku-firmware-analysis.md)):

- Stock kernel is 5.4.213 (newer SDK) and its FIT's **default configuration is `config@hk08`**, not `config@hk07`.
  The ECW230v3 image therefore carries **both** `config@hk07` and `config@hk08` (same kernel and DTB) so either
  name boots. Verified by listing the FIT with `mkimage -l`.
- Its stock flash script validates `soc_hw_version` and `machid` before flashing; the EWS377 ones do not.
- The `rootfs` / `rootfs_1` partition **labels are reversed** on observed ECW230v3 units (slot at `0x1000000` is
  labelled `rootfs_1`). OpenWrt mounts the partition *labelled* `rootfs`, so on those units the install target may
  be the other slot. **Check `cat /proc/mtd` on your stock firmware first and do not blindly use `0x1000000`** — if
  you are the first ECW230v3 tester, please report `/proc/mtd`, `printenv` and the u-boot banner in an issue.

If it does not boot, nothing is lost: stop at the u-boot prompt and restore your backup ([guide §6](install-and-restore.md#6-go-back-to-stock-engenius-firmware)).

## ECW336 (Wi-Fi 6E) — same family, and two corrections it forces

The **ECW336** shares this generation's SoC and has been UART-brought-up and
stock-flashed on three units
([ECW336-firmware-tools](https://github.com/ParkWardRR/ECW336-firmware-tools)).
**An OpenWrt port for it now exists and runs on hardware** — see
[what that took](#the-ecw336-port-works--what-it-took-and-what-it-says-about-ours)
below — and two of its findings bear directly on the notes above.

### 1. A FIT's default configuration does **not** identify the board

The ECW230v3 note above reads `config@hk08` as a hardware difference. Treat that
as unsafe: the ECW336's FIT **also** defaults to `config@hk08`, which matches
neither its board nor the DTB it boots. Worse, the DTB it *does* boot sits in
the slot named **`hk07`** and even carries `model = "…IPQ8074/AP-HK07"`, yet
declares `compatible = qcom,ipq8074-ap-hk01-c6, qcom,ipq8074-hk01` — the board
is **AP-HK01-C6**, host plus a Pine (QCN9074) 6 GHz card. Senao writes its own
board content into a generic QSDK slot, so **slot names and `model` strings are
labels, not identities**; only `compatible` is authoritative (it is what
OpenWrt's `board_detect` uses). More tellingly, a **known-`ap-hk07`**
EWS377-FIT 1.1.65
image defaults to `config@hk01.c4`.

So `configurations/default` is a QSDK **build artifact** — whatever the vendor's
build happened to select — not a statement about silicon. Userspace is no better:
the ECW336 prints `Platform Name: ap-hk01-c6` while its kernel reports `AP-HK07`.

**Read `machid` on the unit — but know exactly what it tells you.** For
reference, hk07 is SoC id pair `0x156 0x185`; hk08/hk06 are `0x158 0x188`.

**`machid` is a DTB selector, not a product ID.** Note the `machid=8010006`
already recorded for the observed **EWS377-FIT** above — the ECW336 reports the
*same* value, differing only in `hw_id` (`0101012B` on that FIT unit vs
`01010123` on the ECW336). Senao evidently programs one CDT across its IPQ807x
line, so:

| question | field to use |
|---|---|
| which config **slot** will u-boot select? | `machid` |
| which **board** is in that slot? | the DTB's **`compatible`** — never its `model` or the slot name |
| which product is this? | `hw_id`, plus `product_id` in the image header |
| what peripherals exist? | inspect the unit (PCIe radios, ethernet PHYs) |

So an EWS377-FIT and a tri-band 6E ECW336 select the **same slot name** while
being obviously different products, because each vendor's firmware puts its own
board content in that slot. Never infer hardware from `machid` or a slot name
alone — and note the ECW336's DTB differs from the EWS377's `hk07` one in ~670
of 3347 lines (ethernet `dp4`+`dp5` vs `dp5`+`dp6`, Aquantia 10G PHY
`ieee802.3-c45` refs the EWS377 lacks, `aqr_int`/`dc_det` GPIOs, `wlan_6g_led`),
so the two board DTSs are **not** interchangeable even though the slot is.

### 2. Partition labels: read them per unit, but they are not always reversed

The ECW230v3 warning about reversed `rootfs`/`rootfs_1` labels is worth keeping,
and the ECW336 shows why it must be phrased as "check, don't assume" rather than
"ECW units are reversed". Its `smeminfo` was identical on all three units, in the
**normal** order:

| partition | offset | size |
|---|---|---|
| `rootfs` (FW1) | `0x1000000` | `0x6ec0000` |
| `0:WIFIFW` (FW1) | `0x7ec0000` | `0x900000` |
| `rootfs_1` (FW2) | `0x87c0000` | `0x6ec0000` |
| `0:WIFIFW_1` (FW2) | `0xf680000` | `0x900000` |
| `0:ART` | `0xf80000` | `0x80000` |

Two things to carry into any port work here:

- The **active slot is chosen by the u-boot env var `active_fw`** (`0` = FW1,
  `1` = FW2) — not by a `BOOTCONFIG` partition, which does not exist on these
  units. Resolve partitions **by name from `/proc/mtd`**, never by index; ART
  was `mtd12` on an EWS377-FIT and `mtd11` on an EWS377AP v3.
- Each A/B slot has its **own `WIFIFW` partition**, so a wifi-firmware write has
  to target the inactive one too.

### The ECW336 port works — what it took, and what it says about ours

Brought up on hardware 2026-11-20 in
[openwrt-engenius-nss-edma](https://github.com/ParkWardRR/openwrt-engenius-nss-edma)
(branch `engenius-ipq807x`; our own patch series is split out as
[engenius-ap-openwrt](https://github.com/ParkWardRR/engenius-ap-openwrt)): all
three radios up on the unit's own Senao calibration (`phy0` 5 GHz, `phy1`
2.4 GHz, `phy2` 6 GHz/50 channels), the 5 GbE AQR114C port bridged and passing
traffic, A/B proven from either slot, and **NSS offload active** — firmware
`NSS.FW.12.5-210-HK.R`, `Wi-Fi: NSS offload active (wifili)`, ~879 Mbps on a
200 MB fetch (line rate for the 1 Gbps the port negotiated), plus ECM
acceleration of through traffic confirmed (57 % of frames forwarded without the
host seeing them).

**Read the release status carefully, because it concerns the APs in this repo.**
`v0.4.0-engenius-ipq807x` ships images for all four boards built from one tree,
but **only the ECW336 is hardware-verified**. The EWS377AP v3, EWS377-FIT and
ECW230v3 images in that release have never been booted with the NSS runtime or
kernel 6.18. Treat them as untested builds, not as a release for this hardware.

Four traps from that port that apply to **any** board in that tree, including
ours:

- **Stale out-of-tree kmods corrupt `sk_buff`, and it looks exactly like an NSS
  bug.** OpenWrt rebuilds the kernel without rebuilding already-built kmod
  packages. Selecting `nss-tools` pulls in `kmod-sched-core`, which turns on
  `CONFIG_NET_SCHED`, which inserts `__u16 tc_index` into `struct sk_buff` —
  every later field shifts by 4 (`skb->tail` 184 in the new kernel vs 180 in a
  stale `cfg80211.ko`). A stale kmod does not fail to load, it *corrupts*: the
  ~4 GiB CE rx DMA map and the `wiphy_register` oops with
  `skb->tail = 0xffff0000` were both this. This was the root cause of every
  earlier "NSS breaks ath11k" symptom. After any kernel-config change, delete
  the whole kernel build tree.
- **Set `country` on every `wifi-device`.** `hostapd.uc`'s
  `device_country_code()` loops all radios and keeps the **last** country it
  finds; OpenWrt gives a 6 GHz radio `country '00'`, which then poisons
  `country_code` for *all* phys and hostapd dies on `Invalid country_code '00'`.
  The simultaneous ath11k `failed to process regulatory info -22` warnings are a
  red herring. Also `wifi reload` skips radios whose config did not change — use
  `wifi down; wifi up`.
- **NSS firmware naming.** `qca-nss-drv` requests `qca-nss{0,1}.bin` while
  `nss-firmware` installs `qca-nss{0,1}-retail.bin`, so the probe fails with
  `-110` about 60 s in. `nss-tools` now symlinks both names.
- **Load order matters.** `nss.init` must load `ath11k_ahb`/`ath11k_pci` itself,
  after arming the data plane and writing `nss_offload`. An `AUTOLOAD` on the bus
  drivers lets kmodloader bind the radios in host mode first.

A new board in that tree needs **six** registrations, and three of them fail
*silently* — the image builds and boots with a subsystem simply dead: the DTS +
`Device/` recipe in `image/ipq807x.mk` (with the Senao header's
`-p <product_id>`), `ipq-wifi-<dev>` board data, an `11-ath11k-caldata` case
(missing ⇒ **no radios at all**: `failed to load board data file: -12`), a
`board.d/02_network` case (missing ⇒ the default bridges the DSA master `eth0`
instead of the `lan` port, so `lan` stays down and nothing passes), and a
`platform.sh` sysupgrade case.

### Hardware notes specific to the ECW336

- Third radio is a **QCN9074 on PCIe** (`remoteproc1: QCN9000_PCI0 is
  available`), in addition to the IPQ8074's on-chip radios — so its port needs
  QCN9074 firmware and PCIe wiring that the EWS377AP v3 DTS has no equivalent
  for. Its caldata offset in `0:art` is **`0x26800`** (the internal pair are at
  `0x1000`), and both radios report `board_id 0xff`.
- Stock is OpenWrt **19.07**, kernel **5.4.213** — the newer-SDK generation, same
  as ECW230v3.
- 6 GHz works on stock and is **not** exotic regulatory territory: a 6E client
  associated on channel 5 (6 GHz) / 80 MHz / 802.11ax / WPA3, `Country Code: US`,
  with no AFC grant, because channel 5 is a PSC channel and Low-Power Indoor
  suffices. 6 GHz does require WPA3-SAE or OWE — WPA2-PSK bound to the band
  disables the radio.
- The stock env ships `country=000`; the regulatory domain arrives from the
  controller, so a port must set it explicitly.
