# Results

Host: Ubuntu 22.04.5 HWE, kernel 6.8.0-138-generic, NVIDIA 550.163.01, two CMP 100-210
(`10de:1df4`, rev a1). Card 1 on CPU root port 00:01.1 (BIOS target Gen1), card 2 behind
a PCIe switch (04:00.0). Both x1 risers.

## Unattended boot run (journal, cmp100-unlock 1.0.0)

    20:38:07  cmp100-unlock 1.0.0 targets: 0000:01:00.0 0000:05:00.0 gen2=1
    20:38:14  0000:01:00.0: Tensor stage PASS
    20:38:16  0000:05:00.0: Tensor stage PASS
    20:38:34  0000:01:00.0: Gen2 stage PASS (attempt 2)
    20:38:44  0000:05:00.0: Gen2 stage PASS (attempt 1)
    20:38:47  PASS

    0000:01:00.0 driver=nvidia tensor=0x00000888[UNLOCKED] link=Gen2 upstream=0000:00:01.1 target=Gen2
    0000:05:00.0 driver=nvidia tensor=0x00000888[UNLOCKED] link=Gen2 upstream=0000:04:00.0 target=Gen4

40 s from unit start to both cards handed back to `nvidia`; no Oops, no Xid, no AER.
Firmware on disk is stock afterwards (`ucode_load.bin` b349f035...).

Card 1 needed the second retrain cycle: after the signed writes NVIDIA reports
`gen 1/max 1`; the first CHANGE_SPEED plus root-port retrain leaves Gen1 but the rebind
makes NVIDIA advertise `max 2`; the second CHANGE_SPEED then trains to 5 GT/s directly.
The upstream CmpUnlocker script dies at the "did not advertise Gen2" check before that
second cycle.

## cmp100-bench (after the boot above)

    GPU0 0000:01:00.0 SMs=80 clk=1147MHz  HMMA dependent latency=36.8 cycles  FP16 tensor throughput=86.9 TFLOP/s
    GPU1 0000:05:00.0 SMs=80 clk=1147MHz  HMMA dependent latency=36.8 cycles  FP16 tensor throughput=86.6 TFLOP/s

Locked reference on the same cards: ~512 cycles per dependent HMMA.
Upstream published: 36.4 cycles, 74-75 TFLOP/s on an 8192^3 GEMM (a different metric).

## Idempotence

Second `systemctl start cmp100-unlock` on an unlocked machine:

    all targets already unlocked; nothing to do

No unbind, no module load, exit 0, ~3 s (the time is nvidia-smi).
