# openwrt-notes

Personal notes from running and debugging OpenWrt.

## Layout

```text
notes/
  troubleshooting/   # symptoms, root causes, fixes, verification
```

## Notes

| Note | Summary |
|---|---|
| [Router unreachable after reboot](notes/troubleshooting/router-unreachable.md) | Corrupt F2FS overlay; TFTP recovery, offline repair, and reflash fallback |
| [Config lost on reboot](notes/troubleshooting/config-lost-on-reboot.md) | tmpfs overlay when residual flash was never formatted |
| [OpenClash breaks remote SSH](notes/troubleshooting/openclash-ssh-broken.md) | Fake-IP + transparent proxy; fix with `DST-PORT,22,DIRECT` |

## Conventions

- Keep folder and file names short and topic-based
- Do **not** put router model names in paths or filenames
- Mention device/platform only inside a note when it matters
- Prefer symptom → cause → fix → verify → prevent structure for troubleshooting notes

See [AGENTS.md](AGENTS.md) for agent/contributor guidance.
