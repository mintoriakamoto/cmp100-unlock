# Register notes (GV100 BAR0)

## Register map

| offset | meaning | stock | unlocked | how it is written |
|---|---|---|---|---|
| 0x409664 | Tensor gate (PGRAPH) | 0x999 | 0x888 | signed ACR ucode payload, via hook |
| 0x409650 | PLM of the above | 0x8f | 0x8f | read-only for us |
| 0x88610 | PCIe policy | 0 | 1 | signed ACR ucode payload |
| 0x8872c | PCIe vector | 0 | 6 | signed ACR ucode payload |
| 0x88084 | link capability, bits 3:0 = max gen | 1 | 2 | becomes 2 after policy+vector and an NVIDIA rebind |
| 0x880a8 | link target, bits 3:0 | 0 | 2 | ditto |
| 0x88088 | link status, bits 19:16 = current gen | 1 | 2 | result of training |
| 0x8c040 | XP config: bits 19:18 rate field, bit 0 CHANGE_SPEED | field=2 | field=1 | CPU-writable via devmem when unbound |

## What the CPU can write

Writes to 0x88084/0x880a8/0x88078/0x8860c from the CPU do not stick (tried; they read
back unchanged). Only 0x8c040 is CPU-writable, and only after the signed writes have
been applied on this boot.

Root port LnkCtl2 (CAP_EXP+0x30) must target >= 5 GT/s or the endpoint never trains
to Gen2 no matter what it advertises. Some BIOSes leave it at Gen1 for x1 slots; the
script raises it and it resets on reboot.

## Hook and nouveau parameters

Hook parameters that work on 1df4 (do not change): `armed=1 replace_on_return=2
load_test_size=0x5d00`, inserted before the nouveau bind. Nouveau must be loaded
`modeset=2 noaccel=1 runpm=0`: with acceleration on, unbind runs the TTM eviction
through a copy engine that was already torn down and Oopses if any DRM client held a BO.

The hook's kprobes are global; `bus=` only selects which BAR0 it reads. Never let two
cards probe nouveau at once (the script disables `drivers_autoprobe` for the run).
