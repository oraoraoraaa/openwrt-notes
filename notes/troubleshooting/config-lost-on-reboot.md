# Config Lost on Reboot (tmpfs Overlay)

**Device:** CMCC RAX3000M (eMMC)  
**Platform:** MediaTek Filogic (`mt7981b-cmcc-rax3000m-emmc`)  
**Firmware observed:** OpenWrt 25.12.5 (`mediatek-filogic` squashfs sysupgrade ITB)  
**Date diagnosed:** 2026-07-24  
**Status:** Fixed after manual format of `/dev/fitrw` as f2fs

---

## 1. Symptom

After reboot, the router behaves like a freshly flashed device:

- All previous settings and configuration files are gone
- Custom LAN IP (e.g. `192.168.39.1`) reverts to stock `192.168.1.1`
- Packages installed after flash disappear
- `/etc/config/*` changes do not survive reboot
- Device looks like factory defaults every boot

This is **not** a normal factory-reset loop and **not** caused by a bad LAN UCI change by itself.

---

## 2. Root Cause

Settings were never written to flash. The writable root overlay lived only in **RAM (tmpfs)**.

### Boot / storage model on this device

1. Firmware boots from squashfs (`/dev/fit0` / `/rom`)
2. Persistent writable overlay should live on residual eMMC space: **`/dev/fitrw`**
3. `mount_root` expects that residual partition to be formatted as **f2fs** (preferred) or **ext4**
4. If the partition is still unformatted (OpenWrt magic `0xdeadc0de`), first-boot setup should format it
5. This image had **no userspace formatter** (`mkfs.f2fs` / `mkfs.ext4` missing)
6. Kernel had f2fs/ext4 support, but without `mkf2fs` / `e2fsprogs`, format never happened
7. `mount_root` fell back to a temporary RAM overlay
8. All config lived in tmpfs → reboot restored stock state

### Diagnostic evidence (broken state)

| Check | Broken result |
|---|---|
| Root mount | `overlayfs:/tmp/root` on **tmpfs** |
| Persistent overlay device | `/dev/fitrw` present but unused as real overlay |
| Magic on `/dev/fitrw` | `de ad c0 de` = OpenWrt “not formatted yet” marker |
| Boot log | `mount_root: overlay filesystem in /dev/fitrw has not been formatted yet` |
| Boot log | `mount_root: jffs2 not ready yet, using temporary tmpfs overlay` |
| Later error | `failed - mount -t jffs2 /dev/fitrw /rom/overlay: Invalid argument` |
| Format tools | `mkfs.f2fs` / `mkfs.ext4` **missing** from image |

### Why this is common on filogic eMMC / FIT residual builds

- Residual space after the FIT image is exposed as `/dev/fitrw`
- `fstools` prefers f2fs for that residual area
- Some custom / slim images omit `mkf2fs` (and sometimes `e2fsprogs`)
- Without a formatter package, first boot cannot create `rootfs_data`
- Result: perpetual “fresh flash / RAM-only config” mode

---

## 3. Fix

SSH in while the router is still reachable at stock IP:

```sh
ssh root@192.168.1.1
```

### Step 1 — Install a formatter (if missing)

Preferred:

```sh
apk update
apk add mkf2fs
```

Fallback:

```sh
apk update
apk add e2fsprogs
```

Confirm:

```sh
ls -l /usr/sbin/mkfs.f2fs /usr/sbin/mkfs.ext4 2>/dev/null
```

### Step 2 — Format the persistent overlay partition

Preferred (f2fs):

```sh
/usr/sbin/mkfs.f2fs -q -f -l rootfs_data /dev/fitrw
```

Fallback (ext4):

```sh
mkfs.ext4 -q -F -L rootfs_data /dev/fitrw
```

> **Warning:** this wipes whatever is on `/dev/fitrw`. On a broken RAM-overlay system there is usually nothing durable there to keep.

### Step 3 — Reboot so `mount_root` attaches the real overlay

```sh
reboot
```

### Step 4 — Re-apply settings (they can now persist)

Example: restore custom LAN IP:

```sh
ssh root@192.168.1.1

# classic single-value style
uci set network.lan.ipaddr='192.168.39.1'
uci commit network
/etc/init.d/network restart
```

Or newer list-style if your image uses CIDR lists:

```sh
uci delete network.lan.ipaddr
uci add_list network.lan.ipaddr='192.168.39.1/24'
uci commit network
/etc/init.d/network restart
```

Reconnect at the new IP and reboot once more to confirm it sticks.

---

## 4. Verification

### Healthy after fix (observed good state)

```sh
mount | grep 'on / '
df -h
dmesg | grep -i overlay
```

#### Expected mount

```text
overlayfs:/overlay on / type overlay (rw,noatime,lowerdir=/,upperdir=/overlay/upper,workdir=/overlay/work,...)
```

Key points:

- Path is **`overlayfs:/overlay`**
- **Not** `overlayfs:/tmp/root`
- Upper dir is `/overlay/upper`

#### Expected df

```text
/dev/root     ~5M     100%  /rom
tmpfs         ~RAM          /tmp
/dev/fitrw    ~2.0G   ~5%   /overlay
overlayfs:/overlay ~2.0G    /
```

Key points:

- `/overlay` is on **`/dev/fitrw`**
- Size is real flash residual space (here ~2G)
- Root overlay reports that same size

#### Expected dmesg

```text
mount_root: overlay filesystem has not been fully initialized yet
mount_root: switching to f2fs overlay
overlayfs: null uuid detected in lower fs '/', falling back to xino=off,index=off,nfs_export=off.
```

Interpretation:

| Message | Meaning |
|---|---|
| `switching to f2fs overlay` | **Success** — real flash overlay is in use |
| `has not been fully initialized yet` | Normal on first boot after format |
| `null uuid detected in lower fs '/'` | Harmless squashfs lower-layer warning |

### Final persistence test

Change something small, reboot, confirm it survives:

```sh
uci set system.@system[0].hostname='DaveRouter'
uci commit system
reboot
```

After reboot:

```sh
uci get system.@system[0].hostname
mount | grep 'on / '
```

Pass criteria:

1. Hostname still `DaveRouter`
2. `/` still `overlayfs:/overlay`
3. `/dev/fitrw` still mounted on `/overlay`

---

## 5. Prevention

### A. Bake the formatter into the image (best)

Include one of these in the image package set:

- `mkf2fs` (preferred for residual eMMC / FIT residual)
- and/or `e2fsprogs` (ext4 fallback)

Then first boot can auto-format `/dev/fitrw` instead of falling back to tmpfs.

For custom builds / image builders, ensure `fstools` has the matching mkfs helper available at first boot.

### B. Keep the formatter installed after recovery

Once overlay is healthy:

```sh
apk add mkf2fs
```

This helps if overlay is ever wiped / recreated later.

### C. Verify overlay immediately after every flash / sysupgrade

Right after first boot of a new image:

```sh
mount | grep 'on / '
df -h | sed -n '1,10p'
dmesg | grep -iE 'overlay|fitrw|mount_root'
```

If you see `overlayfs:/tmp/root` or “using temporary tmpfs overlay”, **do not configure the router yet**. Fix overlay first, or all work will vanish on reboot.

### D. Avoid false “factory reset” actions while debugging

Do **not** run these unless you intentionally want a wipe:

- `firstboot`
- `jffs2reset -y`
- factory-reset button / LuCI reset

Also note:

- “Keep settings” during flash is useless while overlay is broken (there is nothing durable to keep)
- Installing packages before overlay is fixed only stores them in RAM

### E. Optional hardening checklist after recovery

```sh
# 1) overlay healthy
mount | grep 'on / '
df -h | grep -E 'fitrw|overlay'

# 2) keep formatter available
apk add mkf2fs

# 3) take a backup once config is restored
sysupgrade -b /tmp/backup-OpenWrt-$(date +%F).tar.gz
# then scp it off-router
```

---

## 6. Quick Decision Tree

```text
Config lost after reboot?
│
├─ mount | grep 'on / ' shows overlayfs:/tmp/root
│    └─ RAM overlay. Check /dev/fitrw + dmesg mount_root
│         ├─ magic deadc0de / “has not been formatted yet”
│         │    └─ install mkf2fs (or e2fsprogs) → format /dev/fitrw → reboot
│         └─ format tools missing from image
│              └─ same fix; also bake mkf2fs into next image
│
└─ mount shows overlayfs:/overlay and /dev/fitrw is mounted
     └─ overlay is healthy; look elsewhere
          (wrong config, failsafe, dual-boot partition, external reset, etc.)
```

---

## 7. Commands Cheat Sheet

### Broken-state diagnosis

```sh
mount | grep 'on / '
df -h
dmesg | grep -iE 'overlay|fitrw|mount_root|jffs2|f2fs'
hexdump -C -n 16 /dev/fitrw | head
which mkfs.f2fs mkfs.ext4 2>/dev/null
ls -l /usr/sbin/mkfs.f2fs /usr/sbin/mkfs.ext4 2>/dev/null
```

### Recovery

```sh
apk update
apk add mkf2fs
/usr/sbin/mkfs.f2fs -q -f -l rootfs_data /dev/fitrw
reboot
```

### Healthy-state verification

```sh
mount | grep 'on / '
# expect: overlayfs:/overlay ... upperdir=/overlay/upper

df -h
# expect: /dev/fitrw mounted on /overlay with multi-hundred-MB or GB size

dmesg | grep -i overlay
# expect: switching to f2fs overlay
```

---

## 8. Related Paths / Devices (this platform)

| Path / device | Role |
|---|---|
| `/dev/fit0` | FIT / squashfs root image (`/rom`) |
| `/dev/fitrw` | Residual eMMC space for writable overlay (`rootfs_data`) |
| `/rom` | Read-only base firmware |
| `/overlay` | Writable upper layer (must be on flash, not tmpfs) |
| `/` | `overlayfs` combining `/rom` + `/overlay` |
| `/tmp/root` | Temporary RAM overlay used when real overlay is unavailable |

---

## 9. Bottom Line

This was **not** a random OpenWrt settings bug and not caused by the LAN IP change itself.

The eMMC residual partition **`/dev/fitrw` was never formatted**, so the router ran every boot with a **tmpfs overlay**. All configuration lived in RAM and disappeared on reboot.

**Fix:** install `mkf2fs` → format `/dev/fitrw` as f2fs → reboot → verify `overlayfs:/overlay` on `/dev/fitrw` → reconfigure.

**Prevent:** include `mkf2fs` in the image, and always verify the overlay mount right after flash before investing time in configuration.
