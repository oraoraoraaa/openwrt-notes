# AGENTS.md

Guidance for AI agents and humans working in this repository.

## Purpose

This repo stores **personal OpenWrt notes**: debugging write-ups, procedures, and reference material gathered while using OpenWrt routers.

It is a notes repo, not firmware source.

## Naming rules (required)

These rules come from the repo owner and must be followed:

1. **Keep folder and markdown names concise and simple.**
2. **Do not include router model information in naming** (paths, folders, filenames).
3. **Mention the model only inside note bodies when necessary** (device-specific quirks, observed platform, etc.).
4. Prefer topic/symptom-based names over device-based names.

### Good

- `notes/troubleshooting/config-lost-on-reboot.md`
- `notes/network/vlan-bridge.md`
- `notes/packages/apk-basics.md`

### Bad

- `notes/rax3000m-config-lost.md`
- `notes/cmcc-rax3000m/tmpfs-overlay.md`
- `openwrt-rax3000m-config-lost-on-reboot-tmpfs-overlay.md`

## Suggested layout

```text
notes/
  troubleshooting/   # bug/symptom write-ups
  network/           # LAN/WAN/firewall/DNS/VPN (optional, create when needed)
  packages/          # package manager / image / build notes (optional)
  scripts/           # helper scripts if any (optional)
```

Create new top-level topic folders only when a real note needs them. Do not pre-create empty category trees.

## Troubleshooting note structure

For debug notes, use this order when practical:

1. **Symptom**
2. **Root cause**
3. **Fix**
4. **Verification**
5. **Prevention**
6. Optional: decision tree, command cheat sheet, related paths

Include observed device/firmware details in the note body metadata when useful, for example:

```markdown
**Device:** CMCC RAX3000M (eMMC)
**Platform:** MediaTek Filogic
**Firmware:** OpenWrt 25.12.x
**Status:** Fixed / Open / Workaround
```

## Writing style

- Prefer practical commands that can be copy-pasted on the router
- Separate **broken-state** vs **healthy-state** evidence clearly
- Call out destructive commands explicitly
- Keep titles focused on the problem, not the hardware brand
- Update the root `README.md` notes table when adding a notable note

## Git rules

- Commit with **conventional commit** messages, concise and specific
  - Examples: `docs: add tmpfs overlay reboot fix`, `docs: clarify overlay verification`
- **Do not push to remote** unless a human explicitly asks or confirms
- Do not rewrite published history unless explicitly requested
- Do not commit secrets, private keys, live passwords, or full router backups containing credentials

## When adding a note from a live debug session

1. Choose a short topic filename (no model in the name)
2. Place it under the best-matching folder (`notes/troubleshooting/` for most fixes)
3. Put device-specific details only in the note body
4. Link it from `README.md` if it is a main/reference note
5. Commit locally; wait for human instruction before `git push`

## Out of scope

- Full OpenWrt source trees
- Binary firmware images
- Large sysupgrade/backup tarballs
- Credential material

If those are needed for a session, keep them outside this repo (for example under `~/Downloads`) and only document commands/paths here.
