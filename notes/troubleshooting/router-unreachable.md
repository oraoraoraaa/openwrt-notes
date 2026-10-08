# Router unreachable after reboot

**Device reported by owner:** CMCC RAX3000Me

**Model reported by firmware:** CMCC RAX3000M (shared image profile)

**Observed storage:** eMMC, `SCA64G`, 58.2 GiB; MT7531 LAN switch

**Firmware:** OpenWrt 25.12.5, `r33051-f5dae5ece4`, kernel 6.12.94

**Date:** 2026-10-08 (Asia/Shanghai)

**Status:** Recovered through TFTP RAM boot and offline F2FS repair; no reflash

## 1. Symptom

After reboot, the configured wireless networks disappeared and a direct Ethernet connection to LAN did not provide access. The normal LAN address was `192.168.39.1`. Ethernet carrier was present, but there was no ARP response at that address. Checks of `192.168.1.1` and `192.168.0.1` also failed.

The owner had encountered a similar failure before and remembered recovering through failsafe. During this incident, rapid red LED flashing was observed after an attempted failsafe entry, but no ARP or SSH response was obtained on the tested LAN ports. This does not establish whether failsafe actually started. LED color alone is insufficient evidence.

The computer initially used another Wi-Fi network on `192.168.1.0/24`, plus Mihomo TUN. The other network was changed to `192.168.3.0/24` to avoid overlap with recovery. A temporary source-based routing table kept recovery SSH on Ethernet rather than the proxy or the other router.

## 2. Root cause and evidence

### Confirmed immediate fault

The installed squashfs system was readable, but the persistent F2FS overlay on `/dev/fitrw` could not mount. A read-only mount with recovery disabled produced:

```text
F2FS-fs (fitrw): Inconsistent error blkaddr:13311, sit bitmap:0
F2FS-fs (fitrw): sanity_check_extent_cache: inode (ino=3d2) extent info [12800, 33, 512] is incorrect, run fsck to fix
```

An offline dry-run check confirmed:

```text
Info: fs errors: invalid_blkaddr corrupted_inode
Info: checkpoint state = 46 : crc compacted_summary orphan_inodes sudden-power-off
[ASSERT] (fsck_chk_inode_blk:1042) --> ino: 0x3d2 has wrong ext: [pgofs:33, blk:12800, len:512]
[FSCK] other corrupted bugs [Fail]
```

Allocation bitmap, reachable node counts, inode counts, and free-segment counts otherwise agreed. After repair, all reported checks passed, the overlay mounted, and the original network settings returned on normal boot. This establishes overlay corruption as a recoverable fault and strongly connects it to the observed loss of access.

Normal failed-boot console output was not captured, so the exact startup path that led to no LAN access remains unverified. No evidence established OpenClash as the cause. Installed OpenClash files alone do not establish causation.

### What caused the corruption?

The checkpoint recorded an unclean shutdown. It does **not** prove that the owner unplugged power: a kernel crash, watchdog reset, filesystem bug, or storage/power instability can also leave an unclean checkpoint. No specific originating cause was proven.

The eMMC reported `life_time = 0x01 0x01` and `pre_eol_info = 0x01`, which do not indicate exhausted estimated lifetime or a pre-end-of-life warning. These estimates do not rule out intermittent faults. The recovery boot log showed no MMC I/O error in the captured evidence.

### Difference from the earlier incident

[Config lost on reboot](config-lost-on-reboot.md) describes an **unformatted** residual overlay and RAM-only configuration. This incident had an existing F2FS filesystem containing persistent settings and packages; it needed repair, not initial formatting. Do not apply the earlier `mkfs` procedure to this failure without intentionally discarding the old overlay.

## 3. Recovery manual

Commands below distinguish **computer** and **router**. Device names and partition mappings must be checked on every recovery. The observed device used eMMC; do not substitute these block-device commands on a NAND variant.

### A. Try normal access or failsafe first

Connect directly to LAN. Lack of internet is not proof that local management is unavailable: check addresses, DHCP, ARP, HTTP, and SSH separately.

OpenWrt failsafe normally uses `192.168.1.1`, with DHCP and Wi-Fi disabled. Set the computer to a static address such as `192.168.1.254/24`. Briefly press and release Reset during early OpenWrt boot; the timing window is short. Check SSH to confirm success. A long hold during power-on invokes a different bootloader procedure on this installation.

If failsafe works, collect logs and mount information before changing settings. `mount_root` attaches the installed overlay in failsafe, but may fail when that filesystem is corrupt. Do not run `factoryreset`, `firstboot`, `jffs2reset`, or `mkfs` before preserving evidence and deciding to erase settings.

### B. Obtain the two different images

The shared official profile is `cmcc_rax3000m`, including supported RAX3000Me revisions. Hardware support is revision-dependent; consult the device page before choosing another release. All known RAX3000Me revisions are listed as supported from 25.12.3 onward.

| Image | Purpose |
|---|---|
| `openwrt-25.12.5-mediatek-filogic-cmcc_rax3000m-initramfs-recovery.itb` | TFTP boot of a temporary recovery system |
| `openwrt-25.12.5-mediatek-filogic-cmcc_rax3000m-squashfs-sysupgrade.itb` | Install/reinstall the persistent system with `sysupgrade` |
| `OpenWrt-RAX3000M-Necessities.tar.gz` | File-level configuration backup; not firmware or a complete disk backup |

**Computer:**

```sh
mkdir -p "$HOME/Downloads/openwrt-recovery/tftp"
cd "$HOME/Downloads/openwrt-recovery"
base=https://downloads.openwrt.org/releases/25.12.5/targets/mediatek/filogic
curl -fL --retry 3 -o recovery.itb "$base/openwrt-25.12.5-mediatek-filogic-cmcc_rax3000m-initramfs-recovery.itb"
curl -fL --retry 3 -o sysupgrade.itb "$base/openwrt-25.12.5-mediatek-filogic-cmcc_rax3000m-squashfs-sysupgrade.itb"
printf '%s  %s\n' \
  904c75334ac49ae190cf8e3d6c168e62c801d7966ac8005696cc2830ad8a8c4f recovery.itb \
  cd194d6ed64f1fa60d1315e9eafa156138c4c58b76817fb22370af26c4946b21 sysupgrade.itb \
  | sha256sum -c -
```

Both must report `OK`. These hashes apply only to the official 25.12.5 files. Verify another release against its official listing/checksums. Do not rename the sysupgrade image to the recovery filename.

The installed U-Boot requested the **unversioned** filename:

```sh
cp recovery.itb tftp/openwrt-mediatek-filogic-cmcc_rax3000m-initramfs-recovery.itb
```

### C. Prepare computer networking and TFTP

Record existing NetworkManager settings first. Keep an independent internet connection on a different subnet when needed. NetworkManager autoconnect priority selects profiles; route metrics select competing routes. They are different controls.

**Computer, using the observed Ethernet interface `enp5s0`:**

```sh
nmcli connection add type ethernet ifname enp5s0 con-name OpenWrt-Recovery \
  connection.autoconnect yes connection.autoconnect-priority 100 \
  ipv4.method manual ipv4.addresses 192.168.1.254/24 \
  ipv4.never-default yes ipv4.route-table 139 \
  ipv4.routing-rules 'priority 100 from 192.168.1.254/32 table 139' \
  ipv6.method disabled
nmcli connection up OpenWrt-Recovery
ip route get 192.168.1.1 from 192.168.1.254
```

Expected route: `dev enp5s0 table 139`. Use a different unused table/rule priority if these are already occupied. If the profile already exists, inspect or modify it rather than adding duplicates. This creates a connection profile, not a virtual network card.

Install `dnsmasq` through the computer's package manager if absent. Start a temporary, TFTP-only server in a terminal:

```sh
sudo dnsmasq --conf-file=/dev/null --no-daemon --port=0 \
  --interface=enp5s0 --except-interface=lo --bind-interfaces \
  --enable-tftp=enp5s0 \
  --tftp-root="$HOME/Downloads/openwrt-recovery/tftp" \
  --user="$(id -un)" --log-facility=- \
  --pid-file=/tmp/openwrt-recovery-dnsmasq.pid
```

Leave this terminal open. Administrator privileges are needed for UDP 69 and interface binding. No DHCP or DNS service is configured. If a firewall blocks transfers, allow TFTP only on the wired recovery link rather than disabling the entire firewall.

### D. Enter OpenWrt U-Boot TFTP recovery

This sequence is for the **installed OpenWrt U-Boot**, not an assurance that the factory bootloader behaves identically:

1. Connect the computer directly to a LAN port and start the TFTP server.
2. Power off the router.
3. Hold Reset, then power on while continuing to hold it.
4. Release Reset after approximately 10 seconds.
5. Wait for transfer and boot to complete. Allow about two minutes before judging failure.

The successful server log in this incident was:

```text
dnsmasq-tftp: sent .../openwrt-mediatek-filogic-cmcc_rax3000m-initramfs-recovery.itb to 192.168.1.1
```

Confirm a running RAM recovery system, not just a successful transfer:

```sh
ssh -b 192.168.1.254 root@192.168.1.1
```

Accept a new host key only when you have confirmed that you are reaching the directly connected recovery router. The observed initramfs accepted a blank root password. Normal firmware required the configured password afterward.

**Router:**

```sh
ubus call system board
cat /proc/mounts
cat /proc/mtd
ls -l /dev/fit* /dev/mmcblk*
dmesg
ls -la /sys/fs/pstore
fw_printenv
```

The observed root was `tmpfs / tmpfs`, `rootfs_type` was `initramfs`, `/dev/fit0` mapped the installed production squashfs, and `/dev/fitrw` mapped its residual overlay. Firmware named the board CMCC RAX3000M although the owner identified it as RAX3000Me.

If there is no TFTP request, check the cable, all LAN ports, static address, requested filename, server logs, and installed bootloader. A custom bootloader may request a different filename/IP. If TFTP transfers but Linux does not become reachable, capture serial boot output. The device page documents a 3.3 V UART at 115200 8N1 and `mtk_uartboot` recovery. Do not connect a 5 V serial adapter or casually rewrite BL2/FIP.

### E. Preserve the damaged filesystem

Do this before repair while the installed overlay is **unmounted**. Read-only mounts must disable journal/recovery replay where supported.

**Router:**

```sh
mkdir -p /mnt/production /mnt/saved-overlay
mount -t squashfs -o ro /dev/fit0 /mnt/production
cat /mnt/production/etc/openwrt_release
mount -t f2fs -o ro,norecovery /dev/fitrw /mnt/saved-overlay
dmesg | tail -60
```

Failure of the F2FS mount is diagnostic evidence. If it succeeds, unmount it before running fsck.

**Computer:**

```sh
mkdir -p "$HOME/Downloads/openwrt-recovery/evidence"
cd "$HOME/Downloads/openwrt-recovery/evidence"
ssh -b 192.168.1.254 root@192.168.1.1 'dmesg; logread; fw_printenv' > recovery.txt
# Verify that /dev/fitrw is unmounted before this copy.
ssh -b 192.168.1.254 root@192.168.1.1 \
  'dd if=/dev/fitrw bs=1048576 | gzip -1' > overlay-before-repair.img.gz
gzip -t overlay-before-repair.img.gz
```

The observed partition size was 4,172,600 sectors (2,136,371,200 bytes), about 2 GiB. Check `cat /sys/class/block/fitrw/size` for the current size. A valid gzip stream alone does not prove every sector was copied; check SSH/dd success and the decompressed byte count against sectors × 512. Do not start repair while the copy is still running.

Keep images, settings archives, and credentials outside this Git repository. Raw overlay images can contain Wi-Fi keys, proxy subscriptions, SSH keys, and passwords. In this session they were retained under the chat workspace's `work/router-diagnostics/` directory.

### F. Check and repair offline

Locate the tool; on this recovery system it was `/usr/sbin/fsck.f2fs`. Do not assume it is under `/sbin`. If missing, obtain a matching recovery image with the tool or install the compatible `f2fsck` package into the RAM recovery environment when connectivity is available. Do not format the filesystem to solve a missing-tool problem.

**Router, overlay unmounted:**

```sh
cat /proc/mounts
/usr/sbin/fsck.f2fs --dry-run -f /dev/fitrw
```

`--dry-run` was supported by the observed tool; inspect help on other versions. It may print proposed writes even though dry-run is enabled. Save output. The following command **modifies filesystem metadata** and can discard irreparable data; use it only after the backup:

```sh
/usr/sbin/fsck.f2fs -f -y /dev/fitrw
/usr/sbin/fsck.f2fs --dry-run -f /dev/fitrw
mount -t f2fs -o ro,norecovery /dev/fitrw /mnt/saved-overlay
cat /mnt/saved-overlay/upper/etc/config/network
```

The second check must have no remaining failed checks; mount must succeed. Save recovered settings outside the router before reboot if needed:

```sh
# Computer; this archive has an upper/etc layout, not standard sysupgrade backup layout.
ssh -b 192.168.1.254 root@192.168.1.1 \
  'tar -czf - -C /mnt/saved-overlay upper/etc' > recovered-settings.tar.gz
```

Do not feed that raw-overlay archive directly to `sysupgrade -r`. Inspect/extract it separately if manual recovery becomes necessary.

### G. Boot the repaired installation

**Computer:** add a temporary address for the installed LAN subnet:

```sh
nmcli connection modify OpenWrt-Recovery \
  +ipv4.addresses 192.168.39.2/24 \
  +ipv4.routing-rules 'priority 101 from 192.168.39.2/32 table 139'
nmcli connection up OpenWrt-Recovery
```

**Router:**

```sh
umount /mnt/saved-overlay
umount /mnt/production
sync
reboot
```

Allow normal boot time. Connect to `192.168.39.1` using the original credentials. If normal SSH rejects a blank password, that can mean the saved credentials were successfully restored, not that recovery failed.

## 4. Verification

Observed after repair: ARP resolved at `192.168.39.1`, HTTP returned 200, and all three saved wireless networks appeared in a fresh scan. The owner independently confirmed wireless was back. After computer cleanup, Ethernet obtained `192.168.39.126/24` through DHCP, the default route pointed to `192.168.39.1`, and an HTTPS request to GitHub returned 200. That verifies an operational wired internet path at that time, not every WAN service.

A second pre-reboot fsck passed. Full normal-boot router logs and a later repeated reboot were not captured in this session. Run these checks on the healthy router when available:

```sh
cat /proc/mounts | grep -E 'fitrw|overlay'
df -h
ubus call network.interface.lan status
ubus call network.interface.wan status
wifi status
dmesg | grep -iE 'f2fs|fitrw|mount_root|mmc|error'
logread | tail -100
```

Expected: `/dev/fitrw` mounted on `/overlay`, root using that persistent overlay, saved LAN address retained, and no new F2FS errors. A read-only `/rom` showing 100% usage is normal squashfs behavior.

## 5. Reflash fallback if repair fails

This section is a **future fallback, not an action performed in this incident**. Reflashing replaces the installed system and removes added packages. `-n` discards saved settings. Preserve evidence and backups first. Repeated corruption after a clean install should prompt power/storage/serial investigation rather than endless reflashes.

### A. Boot RAM recovery, verify target and image

Use sections B–D. Verify storage mapping, board, checksums, and free RAM. Unmount all manually mounted production filesystems before upgrading. Do not write the whole eMMC or its factory/calibration/bootloader partitions.

**Computer:** upload the already-verified sysupgrade image. `scp -O` uses the legacy protocol when the recovery image has no SFTP server:

```sh
scp -O -o BindAddress=192.168.1.254 \
  "$HOME/Downloads/openwrt-recovery/sysupgrade.itb" root@192.168.1.1:/tmp/sysupgrade.itb
```

**Router:**

```sh
sha256sum /tmp/sysupgrade.itb
sysupgrade -T /tmp/sysupgrade.itb
```

For the official 25.12.5 image the expected SHA-256 is `cd194d6ed64f1fa60d1315e9eafa156138c4c58b76817fb22370af26c4946b21`. Stop if validation fails. Do not bypass an unknown mismatch with `-F`.

### B. Install cleanly

**Destructive: replaces the production firmware and discards settings and added packages.** This is deliberately a separate step after successful validation:

```sh
# Router; unmount any manual production/overlay mounts first.
sysupgrade -n /tmp/sysupgrade.itb
```

Leave power connected until installation and boot finish. SSH disconnection is expected. The installed system normally starts at `192.168.1.1`, with Wi-Fi disabled until configured. Before adding extensive configuration, confirm the persistent overlay is mounted; the earlier [unformatted overlay note](config-lost-on-reboot.md) explains what to investigate if it falls back to RAM. Never format a currently mounted overlay.

### C. Restore the file-level backup

The archive `~/Downloads/OpenWrt-RAX3000M-Necessities.tar.gz` contains `etc/config/*`, account files, Dropbear host keys, and other settings. It is neither firmware nor a package backup. Inspect its file list and compatibility before restoring; it can reinstate passwords, network addresses, and startup behavior. It cannot repair physical storage.

For a compatible installation, use standard backup restore:

```sh
# Computer
tar -tzf "$HOME/Downloads/OpenWrt-RAX3000M-Necessities.tar.gz"
scp -O -o BindAddress=192.168.1.254 \
  "$HOME/Downloads/OpenWrt-RAX3000M-Necessities.tar.gz" root@192.168.1.1:/tmp/settings.tar.gz
```

```sh
# Router
sysupgrade -r /tmp/settings.tar.gz
sync
reboot
```

This restores files, then reboot applies them. Expect the LAN to change back to `192.168.39.1`; add the temporary `192.168.39.2` address as in section G before reconnecting. Packages such as OpenClash must be reinstalled separately. Across incompatible releases, manually migrate selected network/wireless settings instead of restoring the entire archive. Test basic LAN/WAN/Wi-Fi before enabling additional proxy services.

## 6. Prevention and collecting the next failure

- Use LuCI or `reboot` for normal restarts and allow the shutdown to finish. `sync` flushes pending writes but cannot compensate for a faulty supply or filesystem bug.
- Keep verified recovery and sysupgrade images plus private configuration backups off-router.
- Confirm persistent overlay mounts after every flash; keep formatter/checker availability in mind when choosing a recovery image.
- If corruption recurs, capture the **failing normal boot** through serial and preserve `/sys/fs/pstore` before rebooting again. Compare F2FS/MMC errors, watchdog resets, and power behavior. The current incident does not justify a claim that a specific package caused it.
- Avoid periodic forced fsck on a mounted filesystem. Offline checking belongs in recovery with the overlay unmounted.

## 7. Restore the computer after recovery

Stop the TFTP server with **Ctrl+C in its terminal**. If that terminal was lost, identify the exact temporary process and stop only that PID with sudo; do not kill unrelated dnsmasq services.

```sh
nmcli connection down OpenWrt-Recovery
nmcli connection delete OpenWrt-Recovery
nmcli connection up 'Wired connection 1'
ip -br addr
ip route
ip rule
```

Restore any pre-existing profile values from the records taken before recovery. If the intent is standard NetworkManager wired behavior, the defaults used in this session were:

```sh
nmcli connection modify 'Wired connection 1' \
  connection.autoconnect-priority 0 \
  ipv4.method auto ipv4.never-default no ipv4.route-metric -1 \
  ipv4.route-table 0 ipv4.routing-rules '' \
  ipv6.method auto ipv6.ignore-auto-routes no \
  ipv6.never-default no ipv6.route-metric -1
nmcli connection up 'Wired connection 1'
```

Do not overwrite deliberate pre-existing settings on another computer. No recovery static addresses or table-139 policy rules should remain. Keep genuine physical interfaces. Mihomo's TUN interface and its own policy rules belong to the proxy application; stop TUN through that application if no longer wanted rather than deleting its managed interface manually.

If a temporary diagnostic SSH key was authorized, remove only its marked line from `/etc/dropbear/authorized_keys` before deleting the private/public key files on the computer. Preserve private evidence backups until the owner decides they are no longer needed.

## References

- [Device support and OpenWrt U-Boot recovery](https://openwrt.org/toh/cmcc/rax3000m)
- [Official 25.12.5 images and checksums](https://archive.openwrt.org/releases/25.12.5/targets/mediatek/filogic/)
- [OpenWrt failsafe and factory reset](https://openwrt.org/docs/guide-user/troubleshooting/failsafe_and_factory_reset)
- [Sysupgrade procedure](https://openwrt.org/docs/guide-user/installation/generic.sysupgrade)
- [Sysupgrade options](https://openwrt.org/docs/techref/sysupgrade)
- [Configuration backup and restore](https://openwrt.org/docs/guide-user/troubleshooting/backup_restore)
