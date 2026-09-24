---
name: Unlock report or problem
about: Worked / did not work on your card
---

**Card**: `lspci -nn | grep 10de` output, `nvidia-smi --query-gpu=name,vbios_version,pci.sub_device_id --format=csv`
**Host**: distro + `uname -r`, NVIDIA driver version, motherboard, which slot / behind a switch?
**Result of** `sudo cmp100-unlock status`:

```
paste here
```

**Run log** (`/var/log/cmp100-unlock/run-*.log`, latest) and `sudo dmesg | grep -iE "nouveau|acr_hook|NVRM|Xid"`:

```
paste here
```

**Measured**: `cmp100-bench` and `cmp100-pcie-bw` output, if the run passed.
