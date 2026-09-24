# Changelog

## 1.1.0

- PCIe Gen3 x1 is the default. Pre-POST clear of the CYA speed clamp in
  `NV_XVE_PRIV_MISC_1` (0x8841C) after a secondary bus reset, then retrain from the
  upstream port. Mechanism credit: duggasco/CMP100-210.
- Measured 0.806 / 0.845 GB/s H2D / D2H per card (4.1x Gen1, 2.1x Gen2).
- Gen2 signed-write path kept as `--gen2` fallback and used automatically if a card
  loses Gen3 during the Tensor stage.
- `CMP100_PCIE=3|2|0` config knob; `CMP100_GEN2=0` still accepted.
- Refuses the Gen3 path on cards with `OPT_PCIE_BOOT_GEN23/GEN3_DISABLE` fused.
- On failure the `driver_override` hold is released so a manual nvidia bind works.
- New `cmp100-pcie-bw` bandwidth tool; `status` flags the nvidia-smi gen readout.
- 18 mocked tests.

## 1.0.0

- Tensor unlock (0x409664 = 0x888) via signed ACR chain through nouveau, plus PCIe
  Gen2 via two signed writes and an upstream retrain with retries.
- Single `install.sh`, hash-locked payload build from stock firmware, systemd oneshot,
  boot-id crash marker, `cmp100.skip` kernel cmdline, 14 mocked tests.
