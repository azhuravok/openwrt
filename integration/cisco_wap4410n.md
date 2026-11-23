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

They also run once after *every* flash, after the kept settings
(`sysupgrade.tgz`) were restored. So both scripts only replace factory
defaults (`99-local`: LAN still static 192.168.1.1; `98-local-private`:
SSID still `OpenWrt`) and leave a restored or hand-made config alone –
otherwise each flash would reset e.g. the Wi-Fi channel to the image value.

Permissions under `files/` are copied into the image as they are on
disk, and git does not track them (only the exec bit). Keep directories
`755`, `files/etc/dropbear` `700`, `authorized_keys` and `etc/shadow` `600` – with a
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

Settings are wiped; the image defaults apply (see "What you get after a reset" below).

### B. From a running OpenWrt, streaming (keeps settings)

`sysupgrade` does not work on this 32 MB board: it runs out of memory
after "Commencing upgrade" and hangs (nothing written). Stream the image
straight into flash instead, so it never sits in RAM. Simplest: let the AP
pull it from any wired host with a web server (no SSH stream that a Wi-Fi
hop could break):

```
# on a wired host, in a directory with only the image
python3 -m http.server 8088
# on the AP
sysupgrade -b /tmp/sysupgrade.tgz        # name MUST be sysupgrade.tgz
wget -qO- http://<host>:8088/sysupgrade.bin | md5sum   # compare with the build
wifi down
wget -qO- http://<host>:8088/sysupgrade.bin | mtd -j /tmp/sysupgrade.tgz write - firmware
echo 1 > /proc/sys/kernel/sysrq; echo b > /proc/sysrq-trigger
```
Alternative, push from the build host:
`ssh root@AP "mtd -j /tmp/sysupgrade.tgz write - firmware" < sysupgrade.bin`.

- **Do not drop caches** (`echo 3 > /proc/sys/vm/drop_caches`) before
  writing: the running rootfs gets overwritten, and with an empty cache
  even the shell for the reboot command can no longer be loaded
  (2026-11-22: had to power-cycle; harmless, the image was complete).
  Keep the reboot in the same shell session, or power-cycle.
- Takes ~2 min. Settings come back from `sysupgrade.tgz`; the
  `uci-defaults` only touch factory defaults (see above), so e.g. the
  channel is kept.
- New SSH host keys appear only after a factory reset, not after a flash.

## Use case: add the AP to your network

The AP only *extends* an existing network: your router keeps doing DHCP,
DNS and internet; the WAP4410N bridges Wi-Fi to its Ethernet port.

```
 [router 192.168.10.1, DHCP] ──cable── [WAP4410N, DHCP client] ))) Wi-Fi clients
```

### 1. What you get after a reset

Reset = hold the reset button > 5 s, or `firstboot -y && reboot`.
`uci-defaults` scripts run again on the next boot.

| image | LAN | Wi-Fi | how to log in |
|---|---|---|---|
| **public build** (`wap4410n/local`, no private files) | DHCP client (`99-local`), no DHCP server | **off** (OpenWrt default) | find the AP's address in your router's DHCP list (hostname `OpenWrt`), `ssh root@<ip>` – no password yet |
| **my build** (+ private overlay) | DHCP client | `OpenWrt-e`, my key, country/channel from `98-local-private` | `ssh root@<ip>` with my key – nothing to do, it rejoins by itself |
| **stock OpenWrt image** / U-Boot TFTP flash without `files/` | static `192.168.1.1`, DHCP server **on** | off | cable directly to a PC, `ssh root@192.168.1.1` |

> Never plug an AP with its DHCP server on into your LAN – two DHCP servers
> fight. Do step 2 on a direct cable first.

### 2. Join the LAN (only if it is still static 192.168.1.1)

```
uci batch <<'EOF'
set network.lan.proto='dhcp'
delete network.lan.ipaddr
delete network.lan.netmask
delete network.lan.ip6assign
EOF
uci commit network
/etc/init.d/dnsmasq disable; /etc/init.d/dnsmasq stop
reload_config          # the SSH session drops – the AP now asks your router for an address
```
Move the cable to your LAN, find the new address in the router's DHCP list.

### 3. Set a root password (public build / stock image)

```
passwd
```
(or put your public key in `/etc/dropbear/authorized_keys`)

Once key login works, password logins can be switched off (removes the whole
class of password attacks; keep the serial console as fallback):
```
uci set dropbear.@dropbear[0].PasswordAuth=off
uci set dropbear.@dropbear[0].RootPasswordAuth=off
uci commit dropbear; /etc/init.d/dropbear restart
```

### 4. Your Wi-Fi: name, password, country

```
uci batch <<'EOF'
set wireless.radio0.country='UA'
set wireless.radio0.channel='1'
set wireless.radio0.htmode='HT20'
set wireless.radio0.txpower='15'
set wireless.radio0.disabled='0'
set wireless.default_radio0.network='lan'
set wireless.default_radio0.mode='ap'
set wireless.default_radio0.ssid='<your SSID>'
set wireless.default_radio0.encryption='psk2'
set wireless.default_radio0.key='<your password, 8-63 chars>'
EOF
uci commit wireless
wifi reload
```

- **Same SSID + password as your router** → clients roam between router and AP.
  **Different SSID** (e.g. `OpenWrt-e`) → you choose which one a device uses
  (better for testing).
- **Channel**: pick one of 1 / 6 / 11 that your other APs do *not* use
  (`iw dev phy0-ap0 scan ap-force | grep -E "freq|signal|SSID"`). Avoid channels
  12–13 – many phones don't see them. Channel = (MHz − 2407) / 5.
- **HT20** is the safe default. HT40 doubles the speed for 2-stream clients but
  overlaps neighbours and needs `noscan '1'` – see the performance notes.

### 5. Low-memory settings (this board has 32 MB)

Already part of this image (`files/etc/rc.local`, no firewall package); for other builds:
```
# limit the Wi-Fi queue to 1 MB (default 4 MB OOM-kills hostapd under load)
sed -i '/^exit 0/i echo fq_memory_limit 1048576 > /sys/kernel/debug/ieee80211/phy0/aqm' /etc/rc.local
echo fq_memory_limit 1048576 > /sys/kernel/debug/ieee80211/phy0/aqm
# firewall: not needed on a bridged AP
/etc/init.d/firewall disable; /etc/init.d/firewall stop
```

Why no firewall on this AP: it only bridges Wi-Fi to the LAN (layer 2) –
the firewall filters routed traffic, and without `br_netfilter` (not in this
build) it never sees Wi-Fi↔LAN frames. The AP has no WAN, no routing, no NAT,
and the LAN zone accepts everything anyway, so nothing is lost – but nftables,
conntrack and ~15 modules cost ~1 MB of the 32 MB RAM. This is the usual
OpenWrt "dumb AP" setup. You *would* need it for a guest Wi-Fi isolated from
the LAN, or if the AP routes – then keep/reinstall `firewall4`.
(A guest Wi-Fi is also possible without it: second SSID bridged to a VLAN,
e.g. `eth0.20`, with the main router doing DHCP and isolation.)

### 6. Check

```
ip -4 addr show br-lan               # address from your router
iw dev phy0-ap0 info                 # ssid, channel, width, txpower
iw dev phy0-ap0 station dump | grep -E "Station|signal|bitrate"   # clients
free; logread | grep -i oom          # memory
```

### 7. Make it survive a reset

Settings made with `uci` are lost on reset. To keep them, put the same `uci`
lines into the image (`files/etc/uci-defaults/98-local-private` for anything
with a password, `99-local` for the rest), rebuild, flash – then a reset brings
the AP straight back into your network (that is how "my build" in step 1 works).

## Recovery without serial console (tested 2026-11-22)

The UART is not needed for normal operation or recovery:

| situation | what to do |
|---|---|
| config broken, but the AP boots | **factory reset**: hold reset > 5 s while running (or `firstboot -y && reboot`). Comes back on DHCP with the image's Wi-Fi/SSH settings (my build). Tested |
| where is the AP? | router's DHCP list: hostname `OpenWrt`, MAC `1c:df:0f:0c:99:06`. Best: a DHCP reservation for that MAC on the router, then the address never changes |
| cannot log in / network config unusable | **failsafe**: power-cycle; when the power LED blinks fast (~10–15 s after power-on) press reset briefly. The AP is then `192.168.1.1` on the Ethernet port (PC: 192.168.1.2/24, direct cable); settings are not mounted. Log in with `ssh root@192.168.1.1` – my build: SSH key or the image's root password (both are in the image); stock OpenWrt: no password. Then e.g. `firstboot -y; reboot -f`, or `mount_root` to edit the config. Tested |
| firmware broken (does not boot) | only then serial: U-Boot TFTP (method A) |

Reset button while running (`/etc/rc.button/reset`): < 1 s = reboot;
held ≥ 5 s = factory reset – the power LED starts blinking fast at 5 s,
then release; 1–4 s = nothing.

## Configuring with `uci`

OpenWrt keeps its settings in `/etc/config/*` and edits them with
`uci`. Two ways to change something:

| Goal | How | Survives factory reset? |
|---|---|---|
| change this running AP now | `uci set …; uci commit …; reload_config` over SSH | no – lost on `firstboot` / reset button |
| make it part of my image | add the same `uci` lines to `files/etc/uci-defaults/99-local` (or `98-local-private` for secrets), rebuild, flash | yes – runs on every first boot |

Useful: `uci show network` (read), `uci changes` (pending), `uci revert network` (undo uncommitted),
`uci export network` (whole file).

### Current config (2026-11-22, secrets removed)

```
# network: one bridge, LAN by DHCP from the home router
config device               option name 'br-lan'; option type 'bridge'; list ports 'eth0'
config interface 'lan'      option device 'br-lan'; option proto 'dhcp'

# wireless
config wifi-device 'radio0' option band '2g'; option channel '1'; option htmode 'HT20'
                            option country 'UA'; option txpower '15'
config wifi-iface 'default_radio0'
                            option network 'lan'; option mode 'ap'; option ssid 'OpenWrt-e'
                            option encryption 'psk2'; option key '<98-local-private>'

# system: hostname OpenWrt, timezone UTC, NTP *.openwrt.pool.ntp.org
# dhcp: dnsmasq installed but disabled by 99-local (the router serves DHCP)
# firewall: disabled (bridged AP); /etc/rc.local: Wi-Fi queue limit, see below
```

### Examples

Static IP instead of DHCP:
```
uci batch <<'EOF'
set network.lan.proto='static'
set network.lan.ipaddr='192.168.10.5'
set network.lan.netmask='255.255.255.0'
set network.lan.gateway='192.168.10.1'
add_list network.lan.dns='192.168.10.1'
EOF
uci commit network
```
(back to DHCP: `uci set network.lan.proto='dhcp'; uci delete network.lan.ipaddr; uci delete network.lan.netmask; uci delete network.lan.gateway; uci delete network.lan.dns; uci commit network`)

Static route:
```
uci add network route
uci set network.@route[-1].interface='lan'
uci set network.@route[-1].target='192.168.32.0/24'
uci set network.@route[-1].gateway='192.168.10.1'
uci commit network
```

Hostname, timezone, own NTP server:
```
uci set system.@system[0].hostname='wap4410n'
uci set system.@system[0].zonename='Europe/Kyiv'
uci set system.@system[0].timezone='EET-2EEST,M3.5.0/3,M10.5.0/4'
uci delete system.ntp.server
uci add_list system.ntp.server='192.168.10.1'
uci commit system
```

Wi-Fi channel / power (for performance tests):
```
uci set wireless.radio0.channel='6'
uci set wireless.radio0.htmode='HT40'
uci set wireless.radio0.txpower='18'
uci commit wireless; wifi reload
```

Apply after `uci commit`: `reload_config` (or `/etc/init.d/network restart`,
`wifi reload`). Over SSH, changing the LAN address drops the session –
keep serial attached or reconnect to the new address.

To bake any of these into the image, put the lines (without the shell
prompt) into `files/etc/uci-defaults/99-local` before `exit 0`, keep
`uci commit <config>` at the end, rebuild and flash.

## Wi-Fi performance and memory (tested 2026-11-22)

iperf3 server wired behind the AP (100 Mb/s), clients ~2 m away.

| client / setting | down | up |
|---|---|---|
| phone, 1 stream (MCS 7, 65 Mb/s link), any channel | ~25 | ~25 |
| laptop 2x2, channel 11 shared with a strong AP | ~47 | ~44 |
| laptop 2x2, channel 1 (clean), HT20 | **~55** | **~59** |
| laptop 2x2, channel 1, HT40 forced (`noscan`) | 77 | 88 |

- HT40 reaches the 100 Mb/s Ethernet limit and nearly saturates the CPU
  (softirq ~70%). Not kept: on 2.4 GHz it overlaps neighbours; hostapd
  refuses it unless `noscan '1'`.
- txpower 15 → 20 dBm: no gain at short range.
- What helped most: antenna position and a channel with no strong AP on it
  (1 / 6 / 11 plan with the other APs).
- **Memory:** under wired→Wi-Fi load the default mac80211 queue (4 MB)
  OOM-killed hostapd (May and 2026-11-22). Fix: limit it in `/etc/rc.local`
  (`echo fq_memory_limit 1048576 > /sys/kernel/debug/ieee80211/phy0/aqm`) –
  no throughput loss. Disabling the firewall (not used by a bridged AP)
  gave ~1 MB; after that `wifi reload` takes 5 s instead of minutes/OOM.
- Roaming: `wpad-basic` already has 802.11r/k/v – no extra package.
- After a channel change, clients may silently move to another saved
  network and stay there – check before measuring.

## Notes / TODO

- MAC address: read from the Sercomm node-info block (flash `0x3ff70`,
  MAC at `0x3ffa0`), same for eth0 and Wi-Fi, as in the 2015 port.
- SLOB (in `config-5.15`, affects all ath79 in this tree): needed here –
  with SLUB the AP was OOM-rebooting in May. Kept in this branch because
  OpenWrt has no per-device kernel options. SLOB is removed in Linux 6.4+,
  so on 24.10 (6.6) / main (6.12) use `CONFIG_SLUB_TINY` instead.
- **25.12 does not fit (tried 2026-11-23, v25.12.6, kernel 6.12, SLUB_TINY):**
  kernel 3.99 MB gzip > `KERNEL_SIZE` 3584k (23.05: 3.26 MB); kernel code
  +1.2 MB, which stays in RAM – the ~9 MB free of 23.05 would shrink.
  So this board stays on 23.05 (EOL: keep SSH LAN-only, no password logins,
  no port forwards). No upstream PR: new devices go only to main, which is
  bigger still. The converted DTS (`nvmem-layout` cells, needed for 6.x)
  is kept locally on vaio, branches `wap4410n/device-25.12`, `wap4410n/local-25.12`.
- Traffic limits (guest Wi-Fi, "only to host X"): do not filter on the AP.
  Put those clients on a second SSID tagged into a VLAN on `eth0`, and filter
  that VLAN on the router.
- Memory is tight (~24 MB usable). Unused `kmod-ppp`, `kmod-pppoe`,
  `kmod-switch-rtl8366rb` removed from the seed (2026-11-22).
  Candidate if still short: `zram-swap`.
- Image (2026-11-22): `files/etc/rc.local` sets the Wi-Fi queue limit;
  no `firewall4`/`nftables`; `hostapd-basic-wolfssl` instead of
  `wpad-basic-wolfssl` (no wpa_supplicant – AP mode only, 802.11r/k/v kept).
- Later: Wi-Fi client/bridge mode – needs `wpad-basic-wolfssl` back in the seed (rebuild + flash, ~1 MB RAM) and WDS (4-address) on the remote AP or `relayd`.
