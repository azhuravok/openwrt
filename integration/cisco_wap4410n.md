# Cisco WAP4410N – how this unit was brought up on OpenWrt

Personal notes for the `wap4410n/local` branch: what was done to the
device, how to build, how to flash. Device support itself lives in
`target/linux/ath79/dts/ar9132_cisco_wap4410n.dts` and
`target/linux/ath79/image/generic.mk`.

## Credit – this was not possible without prior work

Running OpenWrt on this device depends on knowing how to defeat the
Sercomm check in the stock bootloader. That knowledge comes from the 2015
OpenWrt ar71xx port of the WAP4410N. Without it this port would not exist:

- Caleb James DeLisle, "Support Cisco/Linksys WAP-4410N", openwrt-devel,
  April 2015 – https://desiato.infradead.org/pipermail/openwrt-devel/2015-April/006714.html
- Ryan A Young, "[PATCH v3] ar71xx: Add support for Cisco-Linksys WAP4410N",
  openwrt-devel, July 2015 – https://lists.openwrt.org/pipermail/openwrt-devel/2015-July/008105.html

From that patch:

```
/*
 * An instruction in the bootloader which checks that 0x7dfff8 == "eRcOmM" and
 * bricks if it's not.
 * If this instruction is overwritten with a zero, you get 64k of extra space.
 * 2406 0006 1440 000a 8fbc 0020 <-- unpatched
 * 2406 0006 0000 0000 8fbc 0020 <-- patched
 */
#define SERCOMM_CHECK_INSN_OFFSET 0x19e08
```

## Hardware

- SoC Atheros AR9132 rev 2 (MIPS 24Kc), 32 MiB RAM
- Flash: 8 MiB MXIC 29LV640, parallel CFI at 0xbf000000 (not SPI)
- Ethernet: 1 port, AR8021 PHY over RGMII, no switch
- Wi-Fi: AR9100 WMAC + AR2133, 2.4 GHz 3x3
- Serial: 115200 8N1; U-Boot 1.1.4 "AP83 (ar9100) 0.0.11", prompt `ar7100>`,
  interrupt within bootdelay=4

## Flash map

| mtd | name | range | written when flashing OpenWrt? |
|---|---|---|---|
| 0 | u-boot | 0x000000–0x03FFFF | no (patched once, see below) |
| 1 | u-boot-env | 0x040000–0x04FFFF | only by setenv/saveenv |
| 2 | firmware (kernel + squashfs + rootfs_data) | 0x050000–0x7DFFFF | **yes** |
| 6 | nvram | 0x7E0000–0x7EFFFF | no |
| 7 | art (Wi-Fi calibration) | 0x7F0000–0x7FFFFF | no – irreplaceable, back it up |

Stock layout for reference: u-boot 256k, u-boot-env 64k, rootfs 6464k @0x50000,
uImage 1280k @0x6a0000, nvram 64k, calibration 64k.

## One-time device preparation (done 2026-05)

1. Dump all flash partitions over serial/TFTP and keep them
   (stock U-Boot md5 `75c52b0c281d6912a80bd67625028622`).
2. Patch U-Boot: zero the 4 bytes at offset `0x19e08` of the u-boot
   partition (`14 40 00 0a` → `00 00 00 00`). Patched md5
   `51e3fd0c8ad148916ffc6ddd43dbd4b5`.
   **Never restore the stock U-Boot while OpenWrt is installed** – the
   OpenWrt firmware area overwrites 0x7dfff8 and the stock check would
   refuse to boot.
3. U-Boot environment: `setenv bootcmd 'bootm 0xbf050000'`, `saveenv`.

## Build

```
./scripts/feeds update -a && ./scripts/feeds install -a
cp integration/cisco_wap4410n.config .config
make defconfig
make -j$(nproc)
# -> bin/targets/ath79/generic/openwrt-ath79-generic-cisco_wap4410n-squashfs-sysupgrade.bin
```

## Settings: what goes where

| Layer | Where | Content |
|---|---|---|
| device support (`wap4410n/device`, upstream candidate) | DTS, `image/generic.mk`, `board.d/02_network` | hardware only: flash map, MAC location, LEDs, button, LAN on eth0 |
| personal build (`wap4410n/local`, this branch) | `integration/cisco_wap4410n.config`, `config-5.15` trims | package selection, low-memory kernel (SLOB) |
| personal defaults, in git | `files/etc/uci-defaults/99-local` | LAN by DHCP, no DHCP server (AP on an existing LAN) |
| private, **not in git** | `files/etc/shadow`, `files/etc/uci-defaults/98-local-private` (Wi-Fi SSID/key/country), `files/etc/dropbear/authorized_keys` | secrets; `/files` is in `.gitignore`, keep a copy elsewhere |

`uci-defaults` scripts run once on first boot. So after a factory reset
(reset button held > 5 s, or `firstboot -y && reboot`) the AP comes back
on DHCP with the same Wi-Fi and SSH access – no serial needed
(tested 2026-11-22 with `firstboot -y`).

Permissions under `files/` are copied into the image as they are on
disk, and git does not track them (only the exec bit). Keep directories
`755`, `files/etc/dropbear` `700`, `authorized_keys` `600` – with a
group-writable `/etc/dropbear` dropbear refuses all keys.

## Flashing

### A. From U-Boot over TFTP (first install / recovery) – wipes settings

```
ar7100> setenv ipaddr 192.168.1.10; setenv serverip 192.168.1.11
ar7100> erase 0xbf050000 +0x790000
ar7100> tftp 0x80060000 sysupgrade.bin
ar7100> cp.b 0x80060000 0xbf050000 $filesize
ar7100> reset
```

Comes up with the image defaults (static 192.168.1.254).

### B. From a running OpenWrt, streaming (keeps settings) – used 2026-11-22

`sysupgrade` does not work on this 32 MB board: it runs out of memory
after "Commencing upgrade" and hangs (nothing written). Stream the image
straight into flash instead, so it never sits in RAM:

```
# on the AP
sysupgrade -b /tmp/sysupgrade.tgz     # name MUST be sysupgrade.tgz
wifi down; /etc/init.d/dnsmasq stop; sync; echo 3 > /proc/sys/vm/drop_caches
# from the build host
ssh root@AP "mtd -j /tmp/sysupgrade.tgz write - firmware" < sysupgrade.bin
# verify (optional): compare up to the deadc0de marker, then reboot with
# shell builtins only (the running rootfs has just been overwritten):
ssh root@AP 'echo 1 > /proc/sys/kernel/sysrq; echo b > /proc/sysrq-trigger'
```

Takes ~2 min. On 2026-11-22 the archive was named `cfg.tgz` and settings
were *not* restored (first boot only looks for `sysupgrade.tgz`); they
were put back with `sysupgrade -r`. Keep serial attached; method A is
the fallback.

## Notes / TODO

- MAC address: read from the Sercomm node-info block (flash `0x3ff70`,
  MAC at `0x3ffa0`), same for eth0 and Wi-Fi, as in the 2015 port.
- SLOB (in `config-5.15`, affects all ath79 in this tree): needed here –
  with SLUB the AP was OOM-rebooting in May. Kept in this branch because
  OpenWrt has no per-device kernel options. SLOB is removed in Linux 6.4+,
  so on 24.10 (6.6) / main (6.12) use `CONFIG_SLUB_TINY` instead.
- Memory is tight (~24 MB usable). Unused `kmod-ppp`, `kmod-pppoe`,
  `kmod-switch-rtl8366rb` removed from the seed (2026-11-22).
  Candidate if still short: `zram-swap`.
- Planned: Wi-Fi performance testing (iperf3 is in the image). A similar
  old TP-Link gained throughput after tuning – try here too.
