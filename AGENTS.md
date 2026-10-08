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

| Directory | Purpose |
|---|---|
| `notes/troubleshooting/` | Symptoms, explanations, fixes, and verification |
| `notes/network/` | LAN/WAN, firewall, DNS, or VPN notes when needed |
| `notes/packages/` | Package manager or image/build notes when needed |
| `notes/scripts/` | Necessary helper scripts when needed |
| `backups/` | Selected non-secret snapshots with a clear README |

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

## Documentation rules (required)

- Write for a human reader who understands ordinary networking but may not know OpenWrt internals. Explain unfamiliar terms at first use: for example TFTP, bootloader, overlay, filesystem, RAM, UCI, Fake-IP, and kernel module.
- Lead with the symptom and practical outcome. Separate observed evidence, its interpretation, and unresolved causes. Do not turn a hypothesis into a confirmed root cause.
- Prefer browser downloads, LuCI menus, desktop network settings, and appropriate graphical tools when they offer the needed operation. Provide official download pages and enough navigation to identify the correct device/image.
- Menu labels vary by version/language. Say so where relevant; do not invent a graphical control for an operation that actually requires recovery SSH.
- Use terminal commands when necessary, such as offline filesystem repair, and as optional fallbacks. Label every command block as **computer** or **router**, explain its purpose and expected result, and make examples copyable. Readers should not have to memorize commands.
- Use **Mermaid** for flowcharts, sequence diagrams, storage diagrams, and decision trees. Do not use ASCII art for graphs, diagrams, or folder trees. Tables are appropriate for comparisons; plain log excerpts can remain text blocks.
- Include visuals when they clarify a mechanism or a choice. Do not manufacture measured charts or add diagrams solely for decoration.
- Omit incidental debugging-session details: temporary proxy software, alternate Wi-Fi networks, interface-priority experiments, personal workstation routes, and chat workspace paths. Retain only reusable prerequisites and details needed to reproduce the fault or recovery.
- Device-specific storage mappings, compatible firmware, checksum values, and bootloader sequences are relevant technical details; keep them in note bodies, not filenames.
- Put an explicit data-loss warning immediately before formatting, clean flashing, or reset operations. Distinguish repair from erase/reinstall. Never recommend fsck or formatting on a mounted live overlay.
- Separate historical incident facts from future fallback instructions. State what was verified and what was not tested, especially for backup restoration or reflashing.
- Preserve intentional edits/removals in the latest repository. Pull before adapting existing notes. Do not reintroduce removed sensitive exports or unnecessary narrative.
- Update the root README and related notes when adding/changing a main guide. Keep snapshot descriptions accurate: package names/metadata are not a settings backup.
- Link authoritative documentation near procedures that depend on it. Use actual evidence for captured device facts. Keep credentials, raw logs with secrets, firmware, and private backups outside Git.

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
5. Commit locally; push only with human authorization. An explicit request to commit and push in the current task already supplies that authorization; do not ask again.

## Out of scope

- Full OpenWrt source trees
- Binary firmware images
- Large sysupgrade/backup tarballs
- Credential material

If those are needed for a session, keep them outside this repo (for example under `~/Downloads`) and only document commands/paths here.
