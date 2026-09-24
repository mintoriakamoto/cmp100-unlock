# Register notes (GV100 BAR0)

## Register map

| offset | meaning | stock | unlocked | how it is written |
|---|---|---|---|---|
| 0x409664 | Tensor gate (PGRAPH) | 0x999 | 0x888 | signed ACR ucode payload, via hook |
| 0x409650 | PLM of the above | 0x8f | 0x8f | read-only for us |
| 0x8841c | `NV_XVE_PRIV_MISC_1`: PCIe speed clamp, see below | 0xc0346500 | 0x00340500 | CPU-writable via devmem, pre-POST only (after a secondary bus reset, before any driver binds) |
| 0x2157c | fuse `OPT_PCIE_BOOT_GEN23_DISABLE` | 0 | 0 | read-only; script refuses the Gen3 stage if non-zero |
| 0x21580 | fuse `OPT_PCIE_BOOT_GEN3_DISABLE` | 0 | 0 | read-only; ditto |
| 0x88084 | link capability (`LnkCap`), bits 3:0 = max gen | 0x00453c11 | 0x00453c13 after the clamp clear; re-clamped to ..11 once the NVIDIA driver POSTs | result of 0x8841c |
| 0x880a4 | link capabilities 2 (`LnkCap2`), bit n = Gen n supported | 0x2 (2.5 GT/s only) | 0xe (2.5/5/8 GT/s) | result of 0x8841c |
| 0x880a8 | link target (`LnkCtl2`), bits 3:0 | 0 | 3 | `setpci CAP_EXP+30.w` on the endpoint and the upstream port |
| 0x88088 | link status (`LnkSta`), bits 19:16 = current gen | 1 | 3 | result of training |
| 0x88610 | PCIe policy | 0 | 1 | signed ACR ucode payload (Gen2 fallback path) |
| 0x8872c | PCIe vector | 0 | 6 | signed ACR ucode payload (Gen2 fallback path) |
| 0x8c040 | XP config: bits 19:18 rate field, bit 0 CHANGE_SPEED | field=2 | field=1 | CPU-writable via devmem when unbound (Gen2 fallback path) |

## The Gen1 clamp (0x8841c)

Credit: duggasco/CMP100-210 (`tools/pcie_retrain_probe.py`) found this; we reimplemented
the writes in bash.

`NV_XVE_PRIV_MISC_1` holds four CYA (chicken) bits that force the advertised PCIe speed:

| bit | name | stock |
|---|---|---|
| 13 | `CYA_GEN2_SPEED_OVERRIDE_VAL` | 1 |
| 14 | `CYA_GEN2_SPEED_OVERRIDE_EN` | 1 |
| 30 | `CYA_GEN3_SPEED_OVERRIDE_VAL` | 1 |
| 31 | `CYA_GEN3_SPEED_OVERRIDE_EN` | 1 |

Mask `0xc0006000`. With all four set the endpoint advertises Gen1 only (`LnkCap2 = 0x2`)
even though GV100 is PCIe 3.0 silicon. Clearing them (`0xc0346500 -> 0x00340500`) lifts
`LnkCap` to `0x00453c13` and `LnkCap2` to `0xe` immediately, and the link then trains to
8 GT/s when the upstream port retrains.

Two things make it a soft clamp rather than a fuse on `10de:1df4`: the fuse words
`OPT_PCIE_BOOT_GEN23_DISABLE` (0x2157c) and `OPT_PCIE_BOOT_GEN3_DISABLE` (0x21580) read
0 on both test cards, and the write to 0x8841c sticks from the CPU. A card with either
fuse non-zero is genuinely limited; the script dies with `fused` and `--gen2` is the
only option.

Timing is everything. devinit (run by every driver POST) latches the clamp state into the
link capability, so:

- clamp clear **after** POST (driver was bound once): `LnkCap2` lifts to 0xe but every
  retrain settles at Gen2. This was the first thing tried and it does not work.
- clamp clear **pre-POST**: unbind the driver, secondary bus reset via sysfs `reset`,
  restore `COMMAND` mem+master (the reset clears it), write 0x8841c, set `LnkCtl2` target
  Gen3 on both ends, retrain from the **upstream** port. Trains to Gen3 on the first
  attempt on both slots (CPU root port and chipset switch).

The endpoint's own retrain bit (`LnkCtl` bit 5 on the card) bounces without changing
the speed; only the upstream port's retrain works.

Once trained, the link survives the driver POST that follows (nouveau for the Tensor
stage, then nvidia): devinit re-clamps `LnkCap`/`LnkCap2` to Gen1 but does not downshift
the live link. Consequences, all cosmetic:

- `lspci -vv` prints `LnkSta: Speed 8GT/s (strange), Width x1 (ok)`; "strange" is lspci's
  word for "faster than LnkCap allows".
- `nvidia-smi --query-gpu=pcie.link.gen.current` reports 2 permanently (it reads the
  re-clamped cap, not the trained link). `pcie.link.gen.max` is likewise wrong.
- `/sys/bus/pci/devices/<bdf>/current_link_speed` = `8.0 GT/s PCIe` and measured copy
  bandwidth (~0.8 GB/s x1) are the truth. `cmp100-unlock status` shows both readings.

Gen4 is out of reach: with the clamp cleared `LnkCap2` advertises 2.5/5/8 GT/s only.
Lane width is not touched by any of this (x1 comes from the IFR at flash offset 0x214,
non-volatile).

## What the CPU can write

Writes to 0x88084/0x880a8/0x88078/0x8860c from the CPU do not stick (tried; they read
back unchanged). 0x8841c is CPU-writable only in the pre-POST window described above.
0x8c040 is CPU-writable when unbound, and only after the signed writes have been applied
on this boot (Gen2 fallback path).

Root port LnkCtl2 (CAP_EXP+0x30) must target >= the wanted speed or the endpoint never
trains to it no matter what it advertises. Some BIOSes leave it at Gen1 for x1 slots; the
script raises it (to Gen3, or Gen2 on the fallback path) and it resets on reboot.

## Hook and nouveau parameters

Hook parameters that work on 1df4 (do not change): `armed=1 replace_on_return=2
load_test_size=0x5d00`, inserted before the nouveau bind. Nouveau must be loaded
`modeset=2 noaccel=1 runpm=0`: with acceleration on, unbind runs the TTM eviction
through a copy engine that was already torn down and Oopses if any DRM client held a BO.

The hook's kprobes are global; `bus=` only selects which BAR0 it reads. Never let two
cards probe nouveau at once (the script disables `drivers_autoprobe` for the run).
