# Touchpad variants on the Galaxy Book4 Edge

## The symptom

The touchpad does nothing while the keyboard works, and the I2C controller named in
the device tree is already the right one. A device tree copied from another Book4
Edge SKU produces exactly this. Samsung fits four different touchpads in this
chassis, and two values in the touchpad node belong to whichever one is installed.

Written from the DSDT of an NP750XQA (X1P42100), where the touchpad works, compared
against the DSDT of an NP750XQB (X1P26100), where it does not.

## In this port

The device tree describes all four touchpads as `fail-needs-probe`, and the board code in
`patches/0007` reads TPTY, TPDA and TPDO (below) at boot and turns on the matching node with
the firmware's descriptor register. Nothing needs editing per SKU. The rest of this page
explains the values, and how to read them by hand on another kernel.

## What varies between SKUs

The firmware picks the touchpad's I2C address at runtime from a vendor code it calls
TPTY. The ACPI device SPTP carries all four addresses and returns one of four
hardware IDs.

| TPTY | Reported device | I2C address |
| --- | --- | --- |
| 0 | ELAN0B00 | 0x15 |
| 1 | ZNT0001 (Zinitix) | 0x40 |
| 2 | SYN2602 (Synaptics) | 0x20 |
| 3 | IMG4100 | 0x50 |

The HID descriptor register varies with the vendor as well. ACPI returns it from
`_DSM` as TPDO, so `hid-descr-addr` changes with the touchpad too. On the NP750XQA
it is 0x0e.

That is why a node copied from another SKU can name the right bus and the right
interrupt and still find nothing: `reg` and `hid-descr-addr` are both wrong for the
installed part.

## What does not vary

The `Device (SPTP)` block is byte for byte identical in the two DSDTs, down to the
four address templates and the interrupt descriptor. Neither the bus nor the
interrupt needs changing between these SKUs.

| Item | ACPI | Device tree |
| --- | --- | --- |
| Touchpad controller | `\_SB.IC14` at 0xA94000 | label `i2c13` |
| Touchpad interrupt | GpioInt pin 0x03C0 | `tlmm 3` |
| Keyboard | `SSEC0001` at 0x05 | label `i2c0`, `tlmm 67`, `hid-descr-addr` 0x20 |

The keyboard row is the control. If the keyboard works and the touchpad does not,
`i2c_hid_of` and the I2C driver are both fine, and the fault is in the touchpad
node's own values.

## Read your own three values

The vendor code, the address and the descriptor register sit in memory that UEFI
fills in before Linux starts. Both DSDTs declare the same region:

```
OperationRegion (SNVS, SystemMemory, 0xD4EEA018, 0x0080)
```

TPDF, TPTY, TPDA and TPDO are at offsets 0x50 to 0x53 of it, so they begin at
0xD4EEA068. TPDA is the I2C address and TPDO is the descriptor register.

Those bytes survive into a device-tree boot with no ACPI at all, so they can be read
from the running system:

```sh
sudo python3 -c '
import mmap, os
a = 0xD4EEA068; p = a & ~0xFFF
f = os.open("/dev/mem", os.O_RDONLY | os.O_SYNC)
m = mmap.mmap(f, 0x1000, mmap.MAP_SHARED, mmap.PROT_READ, offset=p)
b = m[a - p : a - p + 4]
print("TPDF=0x%02x TPTY=0x%02x TPDA=0x%02x TPDO=0x%02x" % tuple(b))
print({0: "ELAN0B00", 1: "ZNT0001", 2: "SYN2602", 3: "IMG4100"}.get(b[1]),
      "addr 0x%02x  hid-descr-addr 0x%02x" % (b[2], b[3]))'
```

On the NP750XQA this prints `TPDF=0x00 TPTY=0x01 TPDA=0x40 TPDO=0x0e`, which matches
the values its device tree already uses. Confirm the region's base address in your
own DSDT rather than trusting the constant, since a firmware update could move it.

`dd if=/dev/mem` returns nothing on this kernel, and busybox is not installed by
default on Arch ARM, so use the mmap form above.

## Put the values in the device tree

Substitute TPDA in the node name and in `reg`, and TPDO in `hid-descr-addr`.
Everything else stays as it is.

```
&i2c13 {
	clock-frequency = <400000>;

	status = "okay";

	touchpad@40 {                   /* TPDA */
		compatible = "hid-over-i2c";
		reg = <0x40>;               /* TPDA */

		hid-descr-addr = <0xe>;     /* TPDO */
		interrupts-extended = <&tlmm 3 IRQ_TYPE_LEVEL_LOW>;

		pinctrl-0 = <&tpad_default>;
		pinctrl-names = "default";
	};
};
```

After a boot, `i2c_hid_of` should bind to the address and `/proc/bus/input/devices`
should list the touchpad. On the NP750XQA it reads
`hid-over-i2c 14E5:650E Touchpad`, where 14E5 is the vendor id the pad reports in
its HID descriptor.

## If it binds but the pointer does not move

First find out whether anything enumerated:

```sh
grep -i -A4 touchpad /proc/bus/input/devices
libinput list-devices
```

A device that exists and produces nothing under Wayland is usually a udev
classification problem. `input_id` reads some of these HID devices as graphics
tablets because they advertise `BTN_0`, `BTN_1` and an absolute axis, so libinput
binds them as a tablet pad and never as a pointer.

The internal keyboard on the NP750XQA hits this, and the rule that fixes it is
`userspace/udev/61-book4-keyboard-not-a-tablet.rules`. For a touchpad, match on the
name from `/proc/bus/input/devices` and clear `ID_INPUT_TABLET` and
`ID_INPUT_TABLET_PAD` only, leaving the other verdicts alone. The rule has to sort
after `60-input-id.rules`, which is where those properties are set.

## Do not decode the ACPI pin numbers

The `GpioInt` pin values in this DSDT are not TLMM pin numbers. The touchpad's reads
0x03C0 and works at `tlmm 3`. The keyboard's reads 0x0180 and works at `tlmm 67`.
The two do not even sort the same way, so no arithmetic turns one into the other.
Take interrupt pins from the X1E80100 Book4 Edge device tree, where they are already
correct, and treat them as chassis wiring.

The controller names do map cleanly, and are identical in both DSDTs:

| ACPI | SoC address | Device tree label |
| --- | --- | --- |
| IC10 | 0xA84000 | i2c9 |
| IC14 | 0xA94000 | i2c13 |
| IC16 | 0xA9C000 | i2c15 |
| IC18 | 0x884000 | i2c17 |
| IC19 | 0x888000 | i2c18 |
| IC21 | 0x890000 | i2c20 |
| IC23 | 0x898000 | i2c22 |

To build this table for another SKU, read each `Device (ICnn)` in the DSDT for the
`Memory32Fixed` base in its `_CRS`, then match that address against the `i2c@` nodes
in `hamoa.dtsi`.
