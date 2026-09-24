# When it goes wrong

## Rule 1

If the unit failed after "unbind", do not touch bind/unbind by hand. Reboot.
A faulted nouveau teardown leaves the device lock held; a second write hangs in D state
and cannot be killed.

## Messages

`journalctl -u cmp100-unlock -b` and `/var/log/cmp100-unlock/run-*.log` have the trace.

| message | cause | fix |
|---|---|---|
| `a previous run did not finish (marker ...)` | last boot's run hung or crashed | read that boot's log, then `cmp100-unlock --force` or `rm /var/lib/cmp100-unlock/in-progress` |
| `must start bound to nvidia (is: '')` | nvidia not loaded/probed yet; the script waits 30 s and tries modprobe/bind itself | if persistent, check `dmesg | grep NVRM` and Secure Boot |
| `Tensor baseline 0xffffffff` | BAR0 not decoding | bad riser/slot; the script already forces COMMAND mem+master |
| `active ... is neither stock nor a known payload` | linux-firmware updated to an untested revision | rerun install.sh; if the hashes in tools/build_payloads.py don't match your files, open an issue with sha256s |
| `hook built for ...` | kernel upgraded | `sudo ./install.sh` |
| `X.service is active; a DRM client...` | desktop up during a manual run | run at boot, or stop the DM |
| `did not reach Gen2 after 3 attempts` | root port capped at Gen1 by BIOS, or a Gen1-only riser | `lspci -vv -s <upstream> \| grep -E 'LnkCap|LnkCtl2'`; Tensor stays unlocked regardless |
| `produced no BOOT_RETURN` | nouveau never reached ACR load | `dmesg \| grep -E 'nouveau|gv100'` |

## Disabling

Disable without uninstalling: add `cmp100.skip` to the kernel command line, or
`systemctl disable cmp100-unlock`.
