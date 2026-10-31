# The GNCC GC3: what was found out

Everything here comes from two GC3 cameras bought in 2025, both with stock firmware
7.3.77 and the board `K301A0_V1.1`. Other revisions may differ. Where something is an
inference rather than a measurement, it says so.

## How it was examined

1. One camera was opened (four screws, the lens side comes off the brass base) and
   photographed. That showed the chips, but no labelled serial console pads.
2. Instead of soldering, the stock firmware's SD card hook was used: a card with a folder
   `wifitest` and a script in it that reads all eight flash partitions twice to the card,
   plus `dmesg`, `lsmod`, `/proc/mtd` and the module list. Both reads were identical.
3. The dump was unpacked (`unsquashfs`), the device tree decompiled (`dtc -I dtb -O dts`)
   and everything compared with a dump of a GT1 Pro.
4. The firmware was built for the GC3 from this comparison, checked with `CHECK_ONLY=yes`
   (the camera builds the images and compares them with the tested ones without writing),
   then installed on both cameras. The sensor registers were read and set over I2C on the
   running camera.

## Hardware

| | GC3 | Notes |
| --- | --- | --- |
| Board | one round PCB, `K301A0_V1.1`, `LY-101 E522497 A`, date code `250708`; IR LED ring board `XS-JS-K320 V1.1 2025/04/21` | lens and IR board on one side, SoC side faces the back |
| SoC | marked `ANYKA AK3918EN080`; the kernel reports `AK39XXEV330`, ARM926EJ-S at 780 MHz, DDR2 at 390 MHz, 64 MB | same SoC as GT1 Pro and GK2 |
| Flash | SPI NOR, 8 MB, `zb25vq64` (Zbit), 8-pin package | GT1 Pro: XMC XM25QH64C. The kernel's flash table knows both. |
| Wi-Fi | AltoBeam ATBM603x on the SoC's USB host, soldered module with a stamped metal antenna | driver `atbm603x_x_usb.ko`, the same file as on the GT1 Pro |
| Sensor | **SmartSens SC2331**, 1920×1080, MIPI 2 lanes, I2C address 0x30, chip ID registers 0x3107/0x3108 read `0xcb 0x5c` | GT1 Pro: JX-Q03, 2304×1296. The SC2331 answers at the address its device tree node declares; the JX-Q03 does not. |
| ISP tuning | `isp_sc2331_mipi_2lane_h3b_15fps.conf` | taken from the camera by the installer |
| Other | 24 MHz crystal, USB-C for power, microSD slot, `RST` push button, connectors `IRCUT`, `SPK`, `LED` | no motors |

Pads with a voltage printed next to them (`5V`, `3V3`, `1V8`, `2.8V`, `1.2V`) and two oval
test pads exist, nothing is marked `TX` or `RX`. A serial console was not needed, so none
was looked for any further.

## Flash layout and stock firmware

The partition layout is the one of the GT1 Pro and the GK2:

```
UBOOT   0x000000  256K
ENV     0x040000    4K
ENVBK   0x041000    4K
DTB     0x042000   48K
KERNEL  0x04e000 1800K   uImage, Linux 4.4.192V2.1
ROOTFS  0x210000 1632K   squashfs
CONFIG  0x3a8000  400K   jffs2, settings
APP     0x40c000 4048K   squashfs, the vendor application
```

Compared with a GT1 Pro with stock firmware 7.3.37:

- **KERNEL and ENVBK are identical, byte for byte.** The kernel is the same build
  (`4.4.192V2.1`, 19 September 2024).
- **U-Boot differs in 14 bytes.** It has the same updater commands (`tfdownkernel`,
  `tfdownrootfs`, `tfdownsquashfs`, `fatload ... test_file`, `exfat load root.sqsh4`), so the
  reset button recovery should work the same way. It has not been tried on a GC3.
- **The device tree differs**, see the next section.
- **ROOTFS and APP** are the same except for eight files that differ (`anyka_ipc`,
  `libdev_info.so`, `sensor_module_load.sh`, `wifi_connect.sh` with the DHCP host name,
  `update_check.sh` with the update signature, `mount_reserve_dir.sh`, the DHCP client script,
  to which the GC3's adds public DNS servers as a fallback, and `factory_fw_version`) and two
  that are replaced: the sensor module and its ISP tuning (`sensor_sc2331.ko` and its `.conf`
  instead of the JX-Q03's). 468 files are identical, among them all Anyka libraries, all other
  kernel modules and `cfg80211.ko`.

The stock firmware calls itself `GC3_A3S11A3` on the network and expects updates signed
`ipc_gc3_a3s11a3`.

## Device tree: what differs from the GT1 Pro

| Node | GC3 | GT1 Pro | Consequence |
| --- | --- | --- | --- |
| `leds/wifi_pwr` | GPIO 65, active high | not present | The Wi-Fi chip has a power switch. `S40wifi` sets it to 1 before it loads the USB and Wi-Fi drivers, the same as the stock `wifi_module_load.sh` does. |
| `ircut` | a = GPIO 81, b = GPIO 42, day level a=1 b=0 | a = 81, b = 80, day level a=0 b=1 | Other pin, opposite polarity. GPIO 80 is the **red LED** on the GC3: the GT1 Pro's fixed numbers would blink the LED instead of moving the filter. `ipcd` now reads pins and polarity from `/proc/device-tree/ircut` (patch 0011). |
| `leds/blue_led` | GPIO 0 | GPIO 69 | The LED the device tree calls blue is **white** on the GC3. |
| `leds/red_led` | GPIO 80 | GPIO 42 | See *LEDs* below. |
| `gpiokeys/rst_detect` | GPIO 84 | GPIO 0 | |
| `dma_reserved` | 20 MB (`0x1400000`) | 28 MB | 1080p needs less. `ipcd` sizes its pools from this value. |
| `motor0`, `motor1` | disabled, but pins for two `HZF-24BYJ48` steppers are declared | disabled, no pins | The board design is shared with a pan/tilt model. |
| `saradc` | enabled | disabled | A light sensor input. This firmware does not use it; `photosensitive` is disabled in both device trees. |
| `adc_dac` speaker enable | GPIO 82, active high | GPIO 82, active low | The speaker was not tested. |

## LEDs

Facts: on the GC3 the LED the kernel calls `blue_led` lights **white**. With this firmware
the kernel reports `red_led` as off (`brightness` 0) during normal operation, and yet the
red LED is on. The stock firmware's `main.sh` writes 1 to `red_led` right after loading the
LED driver, with the comment that the driver "will reset the gpio pin", and during normal
stock operation only the white LED is on.

Inference: **for the red LED of the GC3, 0 means on and 1 means off.** The device tree marks
both LEDs active-low; whatever the vendor LED driver makes of that, the white LED follows
"1 = on" and the red one does not. This is the reason for the LED patterns in the README,
which are written as they appear, not as the installer's code intends them:

| Installer state | Code writes | On the GC3 you see |
| --- | --- | --- |
| working | red 1 for 3 s, red 0 for 1 s | red flashes for 1 s every 4 s |
| writing | red 1/0 every 0.2 s | red blinking fast |
| stopped | red 1, white 0 for 0.5 s; red 0, white 1 for 0.5 s | red and white blinking together |
| check finished | red 0, white 1 for 4 s; both 0 for 1 s | red steady, white 4 s on, 1 s off |
| firmware running | `ipcd` sets red 0, white blinking while connecting, then 1 | red steady, white blinking, then both steady |

The installer was left as it was tested. Making the red LED behave on the GC3 would mean
changing the installer and `ipcd`, and testing both again.

## The way in

`main.sh` of the stock firmware mounts the card and then checks for a folder `wifitest`;
if it is there, it runs `wifitest/wifi_test.sh` as root and does not start the cloud
application. This is identical to the GT1 Pro. Two things were learned on the GC3:

- **The card has to be in the slot at power-on.** The check runs once, early. The stock
  application mounts a card that is inserted later and happily records to it, but the hook
  does not run then. Two attempts without a result on the GC3 were most likely this.
- The hook first tries to load a Realtek Wi-Fi module that does not exist on this camera
  (`insmod: can't open '/usr/modules/rtl8188ftv.ko'`). That message in `debug/log.txt` is
  harmless.

## Sensor orientation

After the vendor sensor module has initialised the SC2331, its flip/mirror register
`0x3221` reads `0x66`, and with this firmware's pipeline the picture is upside down. Writing
`0x00` turns it upright:

```
i2ctransfer -f -y 0 w3@0x30 0x32 0x21 0x00
```

That line is in `/etc/config/sensor.init`, which `S60ipcd` runs 15 seconds after `ipcd`
started and then every 60 seconds. Measured on a camera: about 40 seconds after power-on
the register still read `0x66`, after 85 seconds `0x00`. So the picture of every start is
upside down for about the first minute. The ISP's own flip (`--flip`, `--mirror` in `ipcd`)
was not tried on this sensor, as the register write does the job.

## Wi-Fi

- `wifi_pwr` must be 1, see the device tree table. Whether the chip stays dead without it
  was not tried; the stock scripts switch it on, so this firmware does the same.
- WPA2 only, as on the GT1 Pro: with a mixed WPA2/WPA3 network the driver picks SAE and hangs
  in `AUTHENTICATING` (patch 0004).
- The GC3's CONFIG partition has no `wifimac.txt`. The MAC address comes from the driver and
  was the same before and after the installation on both cameras, so a DHCP reservation in the
  router keeps working.
- Both cameras send `gc3` as their DHCP host name. A router that makes DNS names from them
  gives that name to one of them only.

## Night vision

- The IR filter switches with the pins and polarity from the device tree (patch 0011). The
  switch is visible in the picture: with the IR light on, wood and skin turn bright and the
  colour balance shifts towards blue.
- Black and white at night is done in the ISP's YUV effect block (patch 0010, from the GT1
  Pro package): `uv_a` is 64 in daylight on this sensor too, and 0 at night. Measured on both
  cameras: zero chroma at night, normal colour by day; on one of them nine forced switches in
  each direction without a crash or a dropped stream.
- Automatic switching was observed in a dark room: the camera went to night mode by itself,
  and back to night after a forced day. The return to day at dawn had not been observed when
  this was written.
- The GC3 has no ambient light sensor in use, so the firmware switches the IR light off for a
  few seconds every 10 minutes at night to look (patch 0008).

## Frame rate and settings

`fps = 15` gives 15 frames per second on both streams (75 frames in 5 seconds, measured on
the RTSP stream). The GT1 Pro only reaches about 60 % of its setting; the GC3 does not have
that problem.

What the installer writes to the CONFIG partition for a GC3:

```
ipcd.conf     fps = 15, ir_d2n_lum = 6000, ir_n2d_lum = 1024, audio_in_enable = 0,
              http_user, http_pass
ipcd.args     --main-kbps 2000
sensor.init   i2ctransfer -f -y 0 w3@0x30 0x32 0x21 0x00
```

`max_exp_lines`, which the GT1 Pro package sets for its sensor, is left unset.

## Installer timings

| Step | GC3 |
| --- | --- |
| checks and settings | 10 s |
| backup of the flash, read twice | 17 s |
| building ROOTFS | 10 s |
| building APP | 24 s |
| writing APP, KERNEL, ROOTFS with read-back | about 90 s |
| total, `CHECK_ONLY=yes` | 65 s |
| total, installation | 135 s and 157 s on the two cameras |

The least free memory during a run was about 2.4 MB, with 26 MB available, during the
ROOTFS build.

## Test status of release 1.0

- The installer ran on both cameras from the stock firmware, on the second one with a
  check-only run first. The installer in this release differs from the one that ran only in
  comment and message strings.
- The kernel was rebuilt after those runs, with the same source and configuration, so that
  it no longer carries the name of the build machine. The three images of this release, by
  the checksums in `images.sha256`, were then written to the second camera over SSH, and it
  runs them: `/proc/version` shows the neutral build string, Wi-Fi, stream, orientation and
  night switching work as before.
- The first camera runs the ROOTFS image of this release as the installer wrote it, the APP
  image of this release from a later update over SSH, and the kernel from before the
  rebuild, which differs only in its build string.
- Not tested on any GC3: the reset button recovery, `restore-stock.txt`,
  `FORCE_UNKNOWN_FIRMWARE`, the speaker, the microphone.

## Open points

- Why the red LED is inverted, and whether it is the driver or the wiring.
- The return from night to day at dawn: not observed yet.
- The reset button recovery and `restore-stock.txt` on a GC3: not tried.
- The speaker and two-way audio: not tried. The microphone is off by default and was not
  tested either.
- The SAR ADC light sensor input: unused.
- Other board revisions and other stock firmware versions: unknown. The installer compares
  every vendor file it uses with the tested ones and stops on a difference.
