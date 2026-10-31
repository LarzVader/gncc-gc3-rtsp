# RTSP firmware for the GNCC GC3

**[Deutsche Anleitung](README.de.md)**

Turn your GNCC GC3 indoor camera into a plain network camera: RTSP stream, no cloud, no
app, no account, no connection to the internet. All you need is a microSD card. You do not
have to open the camera or solder anything, and you do not need a Linux PC.

Copy the package to a card, enter your Wi-Fi, put the card in the camera, power it on, and
wait five minutes.

This is the GC3 edition of [gncc-gt1-pro-rtsp](https://github.com/LarzVader/gncc-gt1-pro-rtsp).
It is the same installer and the same firmware base. What is different about the GC3, and
how it was found out, is in [What was found out](#what-was-found-out).

## What you get

- **H.264 over RTSP**: `rtsp://<camera>:8554/main` (1920×1080) and `/sub` (640×360), 15
  frames per second, usable with VLC, ffmpeg, Frigate, Home Assistant, Scrypted, Blue Iris
  and others
- **ONVIF** device discovery
- **Night vision**: the IR LEDs and the IR filter switch by themselves, and the night image
  is black and white
- **No way out**: the camera ignores the gateway and DNS server of your router, so it cannot
  reach the internet, even without a rule on your router
- **Its own passwords** on every camera, random unless you set them. There is no default
  password
- **SSH access** as root, as this is your camera
- **Microphone off** by default
- **A full backup** of the flash on your card before anything changes, and a way back to the
  stock firmware

## Is this for your camera?

| | |
| --- | --- |
| Camera | **GNCC GC3**, the indoor camera with a fixed lens and USB-C power. Its board is marked `K301A0_V1.1`. |
| Tested with | stock firmware 7.3.77, on two cameras |
| Card | microSD, **FAT32**, at least 256 MB |
| Wi-Fi | 2.4 GHz with WPA2. WPA2/WPA3 mixed works, WPA3 only does not. |

The installer checks the camera first. If your camera is another model or has a firmware
it does not know, it stops and tells you, and it does not change anything. You can run
just this check by setting `CHECK_ONLY=yes` in `settings.txt`.

## Installation

**1. Download** the file `gc3-rtsp-sdcard-*.zip` from
[Releases](https://github.com/LarzVader/gncc-gc3-rtsp/releases) and unpack it directly onto
the empty card, so that the folders `wifitest` and `rtsp-setup` are in the top folder of the
card.

**2. Enter your Wi-Fi** in the file `settings.txt` on the card:

```
WIFI_SSID=name of your network
WIFI_PASSWORD=your password
```

**3. Install.** Power the camera off. **Put the card in first, then power on.** The stock
firmware looks for the card only once, right at the start: a card that goes in later is
ignored. Then leave the camera alone for about 5 minutes:

| LEDs | Meaning |
| --- | --- |
| Red flashes briefly, about once every 4 seconds | checking, backup and building, about 1 minute |
| Red blinking fast | writing the firmware, about 1½ minutes. **Do not unplug.** |
| Red steady, white blinking | the new firmware is running and connecting to your Wi-Fi |
| **Red steady, white steady** | **done**, the camera is on your network |
| Red and white blinking together | stopped. `RESULT.txt` on the card tells you why. |
| White steady only, from the start, nothing else | the installer did not run: the stock firmware started as usual. Power off and try again, see [Troubleshooting](docs/troubleshooting.md). |

With `CHECK_ONLY=yes` the end looks different: red steady, white 4 seconds on and 1 second
off. Nothing was installed then.

The red LED of the GC3 is on all the time once this firmware runs. That is a quirk of this
board, see [What was found out](#what-was-found-out).

**4. Read the results.** Power off the camera, take out the card and put it into your
computer:

- `CAMERA-ACCESS-<serial number>.txt`: stream addresses and passwords
- `camera-ip.txt`: the address of the camera. You also find it in the device list of your
  router, under the name `gc3`.

**5. Copy the folder `backup-<serial number>` to your computer.** It is your way back to the
stock firmware. It holds private data of your camera, like your old Wi-Fi password and its
cloud certificates: never share it.

Leave the card out from now on, the camera does not need it. The card holds your Wi-Fi
password and the passwords of the camera, so keep it safe or delete `settings.txt` and
`CAMERA-ACCESS-*.txt`. Deleted files on a FAT card can often be recovered, so format the
card before it leaves your house.

More cameras? Use the same card again, one camera after the other. Every camera gets its
own passwords. They all call themselves `gc3` in your router, so tell them apart by their
addresses.

## Watching the stream

```bash
ffplay -rtsp_transport tcp "rtsp://camera:PASSWORD@192.168.1.50:8554/main"
```

In VLC: *Media → Open Network Stream*, with the same address.

There is also a small web page at `http://<camera>/`, with the same user name and password.

## Good to know

- **Give the camera a fixed address** in your router, so its stream address stays the same.
- **The picture is upside down for about the first minute after power-on.** The sensor of the
  GC3 starts rotated, and the firmware sets the orientation once the stream runs, then again
  every minute. Mount the camera the normal way up.
- **At night the picture goes dark for about 5 seconds every 10 minutes.** The firmware uses
  no light sensor, so it switches its IR light off now and then to check whether it is day
  yet. `ir_peek_s` in `ipcd.conf` sets the interval in seconds.
- **RTSP runs over TCP only.** A request for UDP gets `461 Unsupported Transport`, and ffmpeg,
  VLC and the others then switch to TCP by themselves. Over Wi-Fi, UDP would only lose packets.
  `rtsp_udp = 1` in `ipcd.conf` allows UDP anyway.
- **The clock stays wrong** without an NTP server on your network, as the camera has no way
  to the internet. Set `NTP_SERVER` in `settings.txt` if you have one. This does not affect
  the video.
- **Do not press the reset button during operation.** It is only for the recovery at
  power-on.
- There is no setup portal: this firmware never opens an access point. Change your Wi-Fi with
  a file on the card, see [Troubleshooting](docs/troubleshooting.md#the-white-led-keeps-blinking-the-camera-does-not-get-onto-the-wi-fi).

## If something goes wrong

Read [Troubleshooting and recovery](docs/troubleshooting.md). It also explains how to go back
to the stock firmware.

The installer never writes U-Boot, the part of the camera that starts everything. U-Boot has
an updater that works with the reset button no matter what else is in the flash. **On the GC3
this updater has not been tried yet.** It was tested on the GT1 Pro, whose U-Boot is the same
except for 14 bytes, and the GC3's U-Boot has the same commands in it. Until someone has
needed it, treat the way back as untested.

## What was found out

The details, with the numbers, are in [docs/gc3-findings.md](docs/gc3-findings.md). In short:

- The GC3 is the **same platform as the GT1 Pro and the GK2**: Anyka AK3918EV330, 8 MB flash
  with the same partition layout, the same AltoBeam Wi-Fi driver, and a kernel that is byte for
  byte the one of the GT1 Pro's stock firmware.
- The **sensor is a SmartSens SC2331**, 1920×1080, instead of the GT1 Pro's 3 MP JX-Q03. The
  frame rate you set is the frame rate you get: 15.
- The stock firmware has the same **SD card hook** as the GT1 Pro's, so the installer works
  without opening the camera. The board has no labelled serial console pads, and none were
  needed.
- The **device tree differs**: the Wi-Fi chip has a power switch on a GPIO that the firmware
  must turn on, the IR filter sits on other pins with the opposite polarity, the LEDs sit on
  other pins, and the red LED is on when its GPIO reads off. The firmware reads the IR filter
  pins from the device tree now, so this is handled for any board of this family.
- The **sensor starts rotated by 180°** in its own register; one register write sets it
  upright.
- The night image is **black and white** through the ISP, the same way as on the GT1 Pro.

## How it can be legal to do this

A firmware for this camera needs files that are not free, like the media libraries of the chip
vendor and the driver for the sensor. Nobody may publish those, so a ready-made firmware image
cannot exist. But they are already on your camera. So **this package holds no vendor files**:
it holds the free parts, and your camera puts the firmware together itself, from its own files.
The result is compared bit by bit with the tested firmware.

How that works, and what else is different from the stock firmware:
[How it works](docs/how-it-works.md). Do you want to build it yourself?
[Building](docs/building.md).

## Risks

You replace the firmware of a device, at your own risk and only on a camera you own.

- **You lose the warranty** and everything the app and the cloud did: remote viewing through
  the app, cloud recordings, motion notifications, two-way talk.
- **Tested on two cameras**, both with stock firmware 7.3.77 and the same board. That is not
  much. The images in this release are exactly what the second camera installed and runs;
  the first one installed an earlier build of the same sources and runs the same kernel and
  APP image after an update. The installer refuses a camera whose files differ, unless you
  force it.
- **The way back has not been tried on a GC3.** Restoring the stock firmware with
  `restore-stock.txt` and the reset button recovery were tested on the GT1 Pro only. Keep your
  backup folder.
- **A power cut while the red LED blinks fast** leaves the flash half written. The reset button
  recovery is the way out, and on the GC3 it is untested, see above. In the worst case the
  camera needs a flash programmer.
- **The kernel is Linux 4.4 from 2019**, with the vendor's binary drivers, and will never get
  security updates. The firmware has no route to the internet for that reason. Do not open a
  port to it from the outside; reach it through a VPN if you must.
- **No encryption on the network**: RTSP and the web page use Digest authentication over plain
  TCP, SSH is the only encrypted way in. Anyone on your network can see the video traffic.
- **Physical access means full control.** The stock firmware runs whatever a card named
  `wifitest` tells it, and this firmware takes new settings and updates from a card too. That
  is how this package gets in, and it stays that way.
- **The card and the backup hold secrets**: your Wi-Fi password, the camera's passwords, and
  in the backup the old Wi-Fi password and the cloud certificates of the camera.
- **Recording people** is regulated nearly everywhere. The microphone is off by default; what
  you record, where and whom is your responsibility.

If it worked or did not work on yours, please tell us in an
[issue](https://github.com/LarzVader/gncc-gc3-rtsp/issues), with your firmware version and the
`install.log` from the card. It holds no passwords, no serial number and no Wi-Fi name. Do
not attach `dmesg.txt` or `boot.log`: they hold the MAC address of your camera.

## Disclaimer

This is a private project. It is not affiliated with, endorsed by or supported by GNCC,
apemans, Osaio, Anyka, SmartSens, AltoBeam or anyone else whose names appear here. Those names
are the trademarks of their owners and are used only to say which hardware this is about. The
software is provided "as is", without warranty of any kind, see [LICENSE](LICENSE). Nothing
here is advice; check the laws that apply to you before you record anything.

## Thanks

This project stands on the work of others:

- **[gncc-gt1-pro-rtsp](https://github.com/LarzVader/gncc-gt1-pro-rtsp)**: the installer that
  builds the firmware on the camera, and the ipcd patches. This repository is its GC3 edition.
- **[akost-firmware](https://github.com/michelostojski/akost-firmware)** by Michel Ostojski
  (MIT): the free firmware for this chip. Everything hard was done there.
- **[ipcd](https://github.com/medevil84/ipcd)** by medevil (MIT): the program that does the actual
  camera work, with RTSP, ONVIF and the web page.
- **[thingino](https://github.com/themactep/thingino-firmware)**: the model for how a camera
  firmware should update itself.
- **[TECKIN-TC100 Anyka hacks](https://github.com/ThatUsernameAlreadyExist/TECKIN-TC100-Anyka-AK3918-camera-hacks)**:
  the SD card hook that opens the door without any tools.
- The GPL source drop of the [Tuya BSP](https://github.com/tuya/tuya-bsp-gpl-public-components):
  the kernel and the toolchain.
- Significant parts of this project, from the installer to the analysis of the GC3 and this
  documentation, were worked out together with an AI.

## License

The work in this project is under the [MIT license](LICENSE). The package also holds free software
by others, under its own licences: see [THIRD-PARTY.md](THIRD-PARTY.md). Each release has the source
code that comes with it.
