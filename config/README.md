# Kernel config

`book4-edge.config` is the full config the port is built with (base
`zensanp/linux-book4-edge @ 2685c75587ff`, `ARCH=arm64`).

Differences from the config the machine was first brought up with, and why:

| option | why |
|---|---|
| `INPUT_UINPUT=m` | `/dev/uinput` for ydotool and on-screen keyboards |
| `UHID=m`, `HIDRAW=y` | userspace HID (HID-over-GATT) and raw HID access |
| `BT_RFCOMM=m`, `BT_RFCOMM_TTY=y` | Bluetooth headset microphone (HFP/HSP); bluetoothd logs "RFCOMM server failed" without it |
| `PKCS8_PRIVATE_KEY_PARSER=m` | EAP-TLS Wi-Fi with PKCS#8 keys; also silences iwd's modules-load error |
| `ZRAM=m`, `ZSMALLOC=m`, LZ4 + ZSTD backends | compressed swap in RAM; see `userspace/memory` |
| `LRU_GEN=y`, `LRU_GEN_ENABLED=y` | MGLRU reclaim; with `userspace/memory/mglru.conf` a memory squeeze stops one process instead of freezing the desktop (`docs/memory.md`) |
| `CMA_SYSFS=y`, `CMA_DEBUGFS=y` | the CMA pool's own counters in `/sys/kernel/mm/cma/`, to compare with the drifting global count (`docs/memory.md`) |
| `SAMSUNG_GALAXYBOOK4_EDGE=y` | turns on the touchpad the firmware names (`patches/0007`); built in, so it runs before the I2C driver loads |
| `UCLAMP_TASK=y`, `UCLAMP_TASK_GROUP=y` | speed floors and ceilings per group of work (the window in front quick, background work cool): `~/book4-edge/power/tacit-power-policy-plan.md` step 2b |

Everything else is unchanged from the base tree's config.
