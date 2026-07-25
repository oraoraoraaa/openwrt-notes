# OpenClash Breaks Remote SSH (Fake-IP + Transparent Proxy)

**Firmware observed:** OpenWrt (aarch64, kernel 6.12.x) with OpenClash (Meta/mihomo core)  
**Date diagnosed:** 2026-07-25  
**Status:** Fixed by sending all destination port 22 traffic `DIRECT`

---

## 1. Symptom

On the OpenClash-enabled LAN/Wi‑Fi:

```text
ssh user@example.com
Connection closed by 198.18.x.x port 22
```

Verbose SSH shows:

```text
Connecting to example.com [198.18.x.x] port 22.
Connection established.
Local version string SSH-2.0-OpenSSH_...
kex_exchange_identification: Connection closed by remote host
```

Same host works on an unproxied / “native” network (direct modem/ISP path).

Typical fake-IP address range: **`198.18.0.0/16`**.

---

## 2. Root Cause

Two OpenClash behaviors stack:

### A. Fake-IP DNS

With:

- `operation_mode` / `en_mode` = `fake-ip`
- `enhanced-mode: fake-ip`
- `fake-ip-range: 198.18.0.1/16`
- DNS redirect enabled (`enable_redirect_dns` / dnsmasq hijack)

LAN clients do **not** receive the real A record for many hostnames. OpenClash answers with a synthetic address in `198.18.0.0/16`.

Evidence:

| Network | `getent hosts example.com` |
|---|---|
| OpenClash LAN | `198.18.x.x` |
| Public DNS / unproxied path | real public IP (e.g. AWS `18.x.x.x`) |

### B. Transparent proxy of SSH

Even when connecting by **real IP**, SSH can still fail on the OpenClash network because TCP/22 is intercepted by the transparent proxy path (nftables/fw4 + clash). The client completes TCP connect, then the handshake dies (`kex_exchange_identification` / empty banner).

So:

- Fake-IP explains the weird `198.18.*` address
- Transparent hijack explains “all remote SSH is broken,” not only one domain

This is **not** an SSH key problem and **not** a server-side OpenSSH misconfig when the unproxied path works.

---

## 3. Fix (permanent, all SSH)

Send **all destination port 22** traffic direct. Do **not** rely on per-domain exceptions.

### On the router

```sh
ssh root@<router-lan-ip>
```

1. Backup and write top custom rules:

```sh
TS=$(date +%Y%m%d-%H%M%S)
cp -a /etc/openclash/custom/openclash_custom_rules.list \
  /etc/openclash/custom/openclash_custom_rules.list.bak-$TS

printf '%s\n' \
  '# OpenClash custom rules (top)' \
  '# Keep SSH out of the transparent proxy path permanently.' \
  'rules:' \
  '  - DST-PORT,22,DIRECT' \
  > /etc/openclash/custom/openclash_custom_rules.list
```

2. Enable custom Clash rules and commit:

```sh
uci set openclash.config.enable_custom_clash_rules='1'
uci commit openclash
```

3. Restart OpenClash:

```sh
/etc/init.d/openclash restart
```

Notes:

- OpenClash loads `/etc/openclash/custom/openclash_custom_rules.list` at the **top** of `rules:` when `enable_custom_clash_rules=1`
- File format is YAML. Either a bare array or a hash with `rules:` works; OpenClash’s Ruby merge accepts both
- BusyBox on many OpenWrt images has **no `base64`**; use `printf` / `cat` heredoc instead
- `openclash_custom_rules_2.list` is the bottom-insert file; SSH bypass belongs in the **top** file

### Optional LuCI path

OpenClash → Rules / Custom rules:

- Enable custom rules
- Add top rule: `DST-PORT,22,DIRECT`
- Apply / restart

---

## 4. Verification

### On router

```sh
# custom rule present
cat /etc/openclash/custom/openclash_custom_rules.list

# enabled
uci get openclash.config.enable_custom_clash_rules   # expect 1

# first generated rule is DST-PORT 22
grep -n 'DST-PORT,22' /etc/openclash/tag.yaml | head
awk '/^rules:/{f=1} f{print; c++; if(c>5) exit}' /etc/openclash/tag.yaml

# runtime API: index 0 should be DstPort 22 -> DIRECT
SECRET=$(uci get openclash.config.dashboard_password)
curl -s -H "Authorization: Bearer $SECRET" http://127.0.0.1:9090/rules | head -c 400
```

Expected runtime rule shape:

```json
{"index":0,"type":"DstPort","payload":"22","proxy":"DIRECT", ...}
```

### On client

```sh
# may still show 198.18.x under fake-ip mode; that alone is OK after the port-22 rule
getent hosts <ssh-host>

ssh -o ConnectTimeout=10 user@<ssh-host> 'echo SSH_OK; hostname'
# also verify by real IP if known
ssh -o ConnectTimeout=10 user@<real-ip> 'echo SSH_OK_IP; hostname'
```

Successful fix: remote command output appears; no `Connection closed by 198.18...` during KEX.

Observed after fix (2026-07-25):

- Runtime rule index `0` = `DstPort 22 → DIRECT`
- Hostname SSH succeeded even while DNS still returned a fake-IP
- Real-IP SSH also succeeded

---

## 5. Prevention / operational notes

- Prefer **port-based** SSH bypass (`DST-PORT,22,DIRECT`) over domain allowlists if “all remote SSH” must work
- Domain/fake-ip-filter exceptions only help name resolution for specific hosts; they do **not** fully replace a port-22 DIRECT rule when transparent proxy is the failure mode
- After subscription updates, confirm custom rules are still enabled and still first in the generated `rules:` list
- Keep a dated backup of `openclash_custom_rules.list` before edits
- Switching entire mode from Fake-IP → Redir-Host is a larger hammer; not required if only SSH is broken

### What not to do

- Do not “fix” this by disabling OpenSSH keys or reinstalling the remote server when unproxied access already works
- Do not put live dashboard passwords, node credentials, or full config dumps into notes/git

---

## 6. Quick decision tree

```text
SSH fails only on OpenClash network?
  ├─ resolves to 198.18.0.0/16  → Fake-IP involved
  ├─ real IP also fails on that network → transparent proxy hijack
  └─ unproxied network works → server is fine; fix OpenClash rules

Want all SSH working permanently?
  └─ enable custom rules + top rule: DST-PORT,22,DIRECT → restart OpenClash
```

---

## 7. Related paths

| Path | Role |
|---|---|
| `/etc/openclash/custom/openclash_custom_rules.list` | Top custom rules (YAML) |
| `/etc/openclash/custom/openclash_custom_rules_2.list` | Bottom custom rules |
| `/etc/config/openclash` | UCI (`enable_custom_clash_rules`, mode, DNS) |
| `/etc/openclash/tag.yaml` | Generated runtime config (name may match active profile) |
| `/etc/openclash/config/*.yaml` | Subscription / profile source |
| `/tmp/openclash.log` | Start/restart log |
| `http://127.0.0.1:9090/rules` | Clash/Meta rules API (needs dashboard secret) |
