# Linux on the Samsung Galaxy Book4 Edge 15.6" (NP750XQA)

Snapdragon X Plus X1P42100. Device tree, three kernel patches, kernel config, an
EC driver, firmware, and the userspace pieces for charging, memory, audio and the camera.
This is what the laptop runs as a daily machine. As-is, no support.

Base kernel: `zensanp/linux-book4-edge` at commit `2685c75587ff`
(branch `x1e80100-book4e-7.2-14-tmp`, Linux 7.2). Apply `patches/*.patch` on
top and build with `config/book4-edge.config`. `dts/` holds the same board file
as patch 0003, for reading.

## What works

| Function | State |
|---|---|
| Display (KDB KD156N2030A03 eDP), backlight | works (`patches/0001`) |
| Boot splash | `userspace/boot-splash` (plymouth does not work here) |
| GPU (Adreno X1-45, freedreno/turnip) | works; see `userspace/udev/90-book4-gpu-floor.rules` for the devfreq floor |
| Keyboard, touchpad | work (`userspace/udev/61-…` fixes the tablet misdetection). Other SKUs use a different touchpad address: `docs/touchpad-variants.md` |
| UFS storage | works |
| Wi-Fi (WCN7850) | works with a board file built by `scripts/make-board-2.sh` |
| Bluetooth | works; set the address, see below |
| Speakers (2x WSA883x), 4 DMICs | work; EQ/limiter and echo cancel in `userspace/audio`. Headset jack not verified |
| Camera (OV02C10, 2 MP) | works through libcamera; needs `userspace/camera` |
| USB-C charging | works with `userspace/charging/book4-pd-charge` |
| Battery, AC, EC temperatures, fan level | `driver/samsung-galaxybook-ec` |
| Memory under load | freeze fixes: `patches/0004`, MGLRU and zram (`docs/memory.md`); tested for hours, not yet days |
| Suspend | not verified |
| HDMI out, SD card, second USB2 port | not wired / not tested |
| Video decode (iris) | driver built, DT node left disabled |
| RTC | never probes; time is stale until NTP |

## Build

```sh
git clone https://github.com/zensanp/linux-book4-edge && cd linux-book4-edge
git checkout 2685c75587ff
git am ../patches/*.patch                 # panel, camera hflip, the board DTS, CMA counter cap
cp ../config/book4-edge.config .config && make olddefconfig
make -j"$(nproc)" Image vmlinuz.efi modules dtbs
sudo make modules_install
```

Then, for everything below the kernel:

```sh
sudo ./install.sh                                  # firmware, charging, memory, audio, camera, udev
cd userspace/camera/libcamera && makepkg -si      # libcamera with the OV02C10 helper
cd driver/samsung-galaxybook-ec && sudo dkms install .   # EC driver
scripts/make-board-2.sh && sudo cp board-2.bin /usr/lib/firmware/ath12k/WCN7850/hw2.0/
sudo userspace/boot-splash/install.sh                # optional: boot splash + msm gate
```

Install `arch/arm64/boot/vmlinuz.efi`, the DTB and an initramfs on the ESP and
add a systemd-boot entry (`boot/loader-entry-example.conf`). Boot with
`efi=noruntime`: there are no EFI variables at runtime, so on the internal disk
systemd-boot must live at `EFI/Microsoft/Boot/bootmgfw.efi`.

## Prebuilt snapshot

If you would rather not build: the release page carries
`book4-edge-<version>.tar.zst` with the kernel image, DTB, initramfs (DSP firmware
embedded) and modules built from exactly this tree, plus `SHA256SUMS`. Unpack, copy
`boot/*` to the ESP and `lib/modules/<version>` to `/lib/modules/`, run
`depmod <version>`, adapt the loader entry, then run `install.sh` for the rest. It is
a one-time snapshot of the tagged commit; later kernels come from building the source.

## Packages

Beyond a base Arch ARM install: `linux-firmware linux-firmware-qcom
linux-firmware-atheros mesa vulkan-freedreno pipewire wireplumber alsa-ucm-conf
swh-plugins i2c-tools earlyoom iwd networkmanager bluez bluez-utils dtc` and the
three libcamera packages built from `userspace/camera/libcamera/`. Mesa 26.2 or
newer for the Adreno X1-45. Wi-Fi runs through NetworkManager with the iwd backend
(`userspace/network/`); wpa_supplicant was not tested. There is no working RTC, so
keep `systemd-timesyncd` enabled: the clock is wrong until the network is up.

## Initramfs

`boot/mkinitcpio-book4.conf` goes to `/etc/mkinitcpio.conf.d/`; it lists the
modules the display, storage and DSPs need before the root filesystem, and embeds
the DSP firmware. Then:

```sh
mkinitcpio -k <kernel version> -g /boot/initramfs-book4.img
```

## Firmware

`firmware/` carries the ten files `/usr/lib/firmware/qcom/x1p42100/SAMSUNG/NP750XQA/`
needs; `install.sh` puts them in place. Five of them are Qualcomm firmware signed
for this board: the audio DSP `qcadsp8380.mbn` (it also runs the battery and
charger service), the compute DSP `qccdsp8380.mbn`, their DT blobs, and the GPU
zap shader `qcdxkmsucpurwa.mbn`. Without them the machine boots to a console with
no GPU, no audio and no battery state. They come from Samsung's Windows driver
packages, are not in linux-firmware, and carry no redistribution licence; see
`firmware/README.md`. If you have a Windows install, `scripts/extract-firmware.sh`
takes the same files from it and checks them against `firmware/SHA256SUMS`.

GPU, Bluetooth and Wi-Fi firmware come from `linux-firmware`. The Wi-Fi board
file is not shipped (its licence forbids the modified copy); `scripts/make-board-2.sh`
builds it. Rebuild the initramfs after installing firmware: the `FILES=` line in
`boot/mkinitcpio-book4.conf` embeds the DSP files.

## Pieces

- `dts/`: the board file. BSD-3-Clause, credits in the header.
- `patches/`: `0001` adds the panel to panel-edp; `0002` mirrors the
  front camera's HFLIP (board-specific hack, explained in the message); `0003`
  adds the board DTS to the tree; `0004` caps a free-CMA counter that drifts
  during uptime and froze the desktop (`docs/memory.md`); `0005` is the UFS
  no-timestamp quirk for this laptop's storage; `0006` gives the sound card one
  name on every keyboard variant, so the UCM profile matches without a per-SKU link.
  `patches/debug/` holds CSIPHY diagnostics that the verified kernel carried; not a fix.
- `config/`: the kernel config and what was changed from the base and why.
- `driver/samsung-galaxybook-ec/`: ENE KB9058 EC over I2C: battery, AC,
  three temperature sensors, fan level. Read-only by default. GPL-2.0. Builds
  out-of-tree or with dkms.
- `userspace/charging/`: the EC does not run charging on this board; the
  S2MM006 PD controller and the ISL9241 charger are driven directly by Windows'
  EmuEc.sys. `book4-pd-charge` does the same three things (close the consumer
  switch, request the 20 V PD contract, program the charger from the battery's
  own request). Without it the machine trickle-charges at ~300 mA and only
  charges after a reboot. Read `reports/ec/07-pdic-s2mm006.md` first if you
  touch it: two PDIC writes will cut power to the machine. `i2c-wake/` is a
  runtime overlay that enables the three I2C buses on a DTB that lacks them;
  the daemon loads it automatically when needed.
- `userspace/memory/`: zram swap (half of RAM, zstd) plus an earlyoom
  policy as the last-ditch guard. Comments in the files say why systemd-oomd
  is not used here. `zram-mod/build.sh` builds zram out of tree for a kernel
  that lacks `CONFIG_ZRAM` (the community builds before this config).
  `mglru.conf` turns on the anti-thrash setting of MGLRU, which the config
  now builds; together with `patches/0004` this is the fix for the freezes
  (`docs/memory.md`).
- `userspace/audio/`: the ALSA UCM profile (`ucm2/`; alsa-ucm-conf has no
  Samsung entry, and without a profile the card has no usable devices), the
  WirePlumber channel map (the speaker PCM is 4 slots, FL/RL/FR/RR by the
  kernel's definition), PipeWire EQ + limiter measured on the hardware, echo
  cancellation, and a disabled mic low-pass that the DTS's 2.4 MHz DMIC clock
  made unnecessary. UCM is matched on the card long name, which includes the
  SKU suffix; `install.sh` names the include file after the running card.
- `userspace/camera/`: libcamera 0.7.2 PKGBUILD with a sensor helper for
  the OV02C10 (without it AGC never raises gain), a tuning file, and
  `configuration.yaml` forcing the software ISP to CPU mode: the GPU path never
  feeds statistics to the AGC on this platform, so the picture stays black in
  dim rooms. Firefox/Zen need `media.webrtc.camera.allow-pipewire = true`;
  Chromium the `WebRtcPipeWireCamera` flag (`docs/browsers.md`).
- `userspace/boot-splash/`: a DRM boot splash replacing plymouth, which
  fails on this hardware (see its README). Brings the msm modprobe gate with it;
  install both or neither.
- `scripts/`: `extract-firmware.sh` pulls the five signed files out of a
  Windows install; `make-board-2.sh` builds the Wi-Fi board file from stock
  linux-firmware and vendors `ath12k-bdencoder` (ISC, qca-swiss-army-knife).
- `reports/`: the EC and PD controller protocol notes, the DSDT
  disassembly, the panel EDID. `docs/camera-debugging.md` has the methods that
  worked while bringing the camera up, and `docs/touchpad-variants.md` covers the
  four touchpads Samsung fits in this chassis and how to read which one you have.

## Bluetooth address

Nothing on this platform injects `local-bd-address`, so the controller comes up
with the QCA default address. Take the address from Windows (Device Manager,
Bluetooth adapter, Advanced) and either add it to the DTS (bytes reversed, see
the comment in the `bluetooth` node) or set it at boot with
`btmgmt -i hci0 public-addr XX:XX:XX:XX:XX:XX`.

## What is verified

Everything in the table above was verified on this machine with a kernel built
from this base and these patches. Its config differs from `config/book4-edge.config`
only by the options listed in `config/README.md` (zram, hidraw), and its DTB is
reproduced by this DTS exactly (decompiled and diffed) except for three Type-C I2C
buses the DTS now enables for the charging daemon. The build steps above were run from a fresh clone and produced the snapshot
release; booting that snapshot, and the camera without `patches/debug/`, are
being verified on a second install before the first tag.

## Credits

zensanp (base tree), Kirill A. Korinsky and Valentin Manea (X1E80100 Book4 Edge
DTS), Jens Glathe (eDP power/backlight wiring, ThinkBook 16), saddytech (EC
mailbox protocol), the libcamera and linux-firmware projects.

## Licences

Kernel patches, the EC driver and the scripts: GPL-2.0-only. Device tree:
BSD-3-Clause. libcamera patches: LGPL-2.1-or-later. Tuning file: CC0-1.0.
