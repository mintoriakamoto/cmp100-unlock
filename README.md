# cmp100-unlock

Boot-time Tensor-core and PCIe Gen2 unlock for the NVIDIA **CMP 100-210** mining card
(GV100, PCI IDs `10de:1d84` and `10de:1df4`), running the stock proprietary driver.

    git clone https://github.com/father-lab/cmp100-unlock
    cd cmp100-unlock
    sudo ./install.sh --run

That is the whole procedure. One systemd oneshot runs at every boot before anything
touches the GPUs, unlocks each card, and exits. Nothing stays resident. Nothing is
flashed: no VBIOS, no eFuse. Reset or power loss puts the card back to stock.

## What you get

Measured on two `10de:1df4` cards, driver 550.163.01, Ubuntu 22.04 HWE 6.8:

| | stock | unlocked |
|---|---|---|
| HMMA (`wmma m16n16k16`) dependent latency | ~512 cycles | 36.8 cycles |
| FP16 tensor throughput per card | ~6 TFLOP/s | 86.7 TFLOP/s |
| PCIe link | Gen1 x1 (2.5 GT/s) | Gen2 x1 (5 GT/s) |
| `nvidia-smi` name | CMP 100-210 | Tesla V100-PCIE-12GB |

Run `cmp100-bench` after install to see the numbers on your card (it uses PTX JIT
through libcuda, no CUDA toolkit required).

## How it works

The card's Tensor gate is a register (`0x409664`) that only the signed ACR firmware
chain can write. `gv100_nouveau_acr_hook.ko` kprobes nouveau's ACR loader and, for
one boot of one card, swaps in a modified `ucode_load.bin` (derived at install time
from *your* stock firmware, hash-checked) that writes `0x888` into that register and
returns cleanly. The script then hands the card back to the NVIDIA driver.

Per card, in order:

1. unbind `nvidia`, read the Tensor register (skip if already `0x888`)
2. arm hook, stage payload, bind `nouveau` with `modeset=2 noaccel=1 runpm=0`,
   verify `BOOT_RETURN ... function_rc=0 ... tensor_409664=0x00000888` in dmesg
3. unbind `nouveau`, restore stock firmware, remove hook, rebind `nvidia`
4. Gen2: two more signed writes (`0x88610=1`, `0x8872c=6`), raise the upstream port's
   target link speed if the BIOS pinned it at Gen1, correct the XP rate field at
   `0x8c040`, pulse CHANGE_SPEED, verify 5 GT/s on the wire and in `nvidia-smi`.
   Retries the retrain cycle up to 3 times; some slots only advertise Gen2 after a
   driver rebind.

Every step verifies before the next. On any failure the script restores stock
firmware, removes the hook, logs, and stops. It deliberately does **not** try to
re-bind a card after a failed unbind (that wedges the device lock; a reboot is the
only safe recovery). A marker file prevents the next boot from re-running into the
same hang until you clear it or pass `--force`. Kernel cmdline `cmp100.skip`
disables the unit entirely.

The whole pass takes about 3 s per card for Tensor and 5-8 s for Gen2.

## Requirements

- CMP 100-210: `10de:1d84` or `10de:1df4`. Check with `lspci -nn | grep 10de`.
- Proprietary NVIDIA driver (550.163.01 tested; users report newer works). **Not**
  the open kernel module flavour: Volta is not supported there.
- `linux-firmware` providing `nvidia/gv100/{gr/fecs_sig.bin,acr/bl.bin,acr/ucode_load.bin}`
  at the tested revision (SHA-256 in `tools/build_payloads.py`). The installer refuses
  to derive payloads from any other version.
- Kernel headers for the running kernel, `build-essential`, `busybox`, `pciutils`,
  `psmisc`, `zstd`, `python3`. The installer apt-installs what is missing.
- Secure Boot off, or a signing setup that lets the unsigned hook load.
- No display manager holding the GPU while it runs. At boot the unit is ordered
  before `display-manager.service`, so a desktop is fine as long as you don't run the
  script by hand with the desktop up. Dedicated compute box: `systemctl set-default multi-user.target`.

Tested: Ubuntu 22.04 HWE (6.8.0-138), Debian 13, bare metal, x1 risers and a PCIe
switch. The upstream CmpUnlocker project tested Proxmox passthrough too.

## Commands

    sudo cmp100-unlock status        # read-only: tensor reg, link speed, driver per card
    sudo systemctl start cmp100-unlock
    journalctl -u cmp100-unlock -b   # or /var/log/cmp100-unlock/run-*.log
    sudo cmp100-unlock --force       # after a failed run on a previous boot
    sudo cmp100-unlock --no-gen2     # tensor only
    sudo ./uninstall.sh [--purge]

Config in `/etc/cmp100-unlock.conf`: explicit BDF list, Gen2 on/off, services to
stop first, retrain attempts.

## Troubleshooting

`Tensor baseline 0xffffffff` : BAR0 not decoding. The script sets COMMAND memory+bus-master
after every unbind; if you still see this the slot/riser is flaky.

`X is active; a DRM client on the GPU...` : stop gdm/lightdm/sddm/switcheroo-control, or reboot
and let the unit run before them.

`produced no BOOT_RETURN` : hook loaded but nouveau didn't reach the ACR loader. Check
`dmesg | grep -E 'nouveau|gv100'`. Usually the firmware hash check would have caught this earlier.

`NVIDIA policy did not advertise Gen2` / stuck at Gen1 : your upstream port targets Gen1
(`lspci -vv -s <root> | grep LnkCtl2`). The script raises it automatically; if the BIOS has an
explicit "PCIe Gen1" slot setting, change it there.

Kernel upgrade: `sudo ./install.sh` again rebuilds the hook. The unit refuses to run with a
hook built for another kernel.

Card in a bad state after a failure: **reboot**. Do not poke sysfs bind/unbind by hand.

## Layout

    install.sh / uninstall.sh
    sbin/cmp100-unlock            the unlock script (bash, ~500 lines, fail-closed)
    src/gv100_nouveau_acr_hook.c  kprobe hook, accepts 10de:1d84 and 10de:1df4
    tools/build_payloads.py       derives the 5 payloads from stock firmware, hash-locked
    tools/cmp100-bench            HMMA latency / FP16 throughput via PTX JIT
    systemd/cmp100-unlock.service
    modprobe.d/cmp100-unlock.conf blacklist nouveau + nvidia_drm, RMPcieLinkSpeed=0x1
    tests/                        mocked full-flow tests (no hardware)
    docs/                         register notes, results, methodology

## Credits

The signed-ACR technique, hook, payload builder and register map come from
[Brazzo978/CmpUnlocker-100-210](https://github.com/Brazzo978/CmpUnlocker-100-210) (GPL-2.0).
This repo packages it as a single boot-time installer, adds `10de:1df4` support, the
`noaccel` nouveau workaround for the unbind Oops, the upstream-port target-speed fix and
the retrain retry loop that gets Gen2 on slots where the original script stops.

GPL-2.0-only. You are writing to a GPU's firmware loader as root; read the script
before running it.
