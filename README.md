# usbliter8

Tethered bootrom exploit for Apple A12, S4/S5 & A13 SoCs (A12X/Z can theoretically be supported as well, but it's not implemented yet).

## Bug & exploit write-up

Available in [this archived blog post](https://web.archive.org/web/20260719005132/https://ps.tc/pages/blog-usbliter8.html).

## Usage

### Hardware requirements

The exploit abuses a very low level bug of the USB controller. This means that default Mac/PC USB stack can't normally reach it. So instead we use Raspberry Pi's RP2350-based microcontroller boards.

The board we use is [**Waveshare RP2350 USB-A**](https://www.waveshare.com/wiki/RP2350-USB-A) with Lightning to USB-A cable and R13 resistor *optionally* [removed](https://qsantos.fr/2025/11/21/fixing-the-rp2350-usb-a-not-working-as-usb-host/).

Other RP2350-based boards can be used as well if you cut a Lightning cable and solder it directly to corresponding pins.

Typically GPIO12 & 13 are used for D+ & D- signals respectively, but that's configurable. Do NOT use USB-C cables, as those often have a very different pinout. And keep the remaining cable (with the Lightning end) relatively short.

Here is a list of the boards we tested the exploit on:

* Waveshare RP2350 USB-A
* Waveshare RP2350 Zero
* Pimoroni TINY2350
* Raspberry Pi Pico 2

RP2040 can be theoretically used as well, but this is currently **NOT** very stable and Apple A13 SoC does **NOT** work at all.

### Flashing firmware

Compiled **UF2** images are available for [RP2350 boards here](https://github.com/JoshAtticus/usbliter8/releases/tag/rp2350) and for [RP2040 boards here](https://github.com/JoshAtticus/usbliter8/releases/tag/rp2040). They can be flashed either via mass storage protocol of RP2350 bootrom or via [picotool](https://github.com/raspberrypi/picotool).

If your board is not among supported ones, you can create your own board configuaration file (in `/boards` folder) and refer to building instructions that come later in this README.

### Exploiting

1. Enter [DFU mode](https://theapplewiki.com/wiki/DFU_Mode#A11_and_newer_devices_without_clickable_home_buttons_(iPhone_8_and_above,_iPad_Pro_2018_and_above,_iPad_Mini_2021,_iPad_Air_2020_and_above,_iPad_10th_generation)). Do this while device is connected to your Mac/PC
    * Do NOT enter DFU by breaking LLB - this will not work

2. Unplug it from Mac/PC and replug it into your RP2350 board

3. In a few seconds, the exploit will finish

There are 2 ways you can watch the process:

1. RP2350 appears as a virtual COM-port - the exploit logs will be printed there

2. Via on-board LED

If LED is RGB:

1. Blinking orange - RP2350 is booting (takes ~2 second)
2. Steady orange - idle, ready to exploit
3. Blue - exploit in progress
4. Green - exploit succeeded!
5. Red - exploit failed

If LED is single-color:

1. Slow blinking (200ms period) - RP2350 is booting (takes ~2 second)
2. Breathing - idle, ready to exploit
3. Rapid blinking (100 ms) - exploit in progress
4. Steady - exploit succeeded!
5. Off - exploit failed

The exploit takes 0.7 - 1.2 seconds to run. After either success or failure, you need to reboot the microcontroller board - via on-board button, or via `picotool`, or just by replugging power.

### After exploiting

Replug the device back to your Mac/PC.

In USB serial number you shall see PWND string in the end, for instance:

```
CPID:8020 CPRV:11 CPFM:03 SCEP:01 BDID:0E ECID:XXXXXXXXXXXXXXXX IBFL:3C SRTG:[iBoot-3865.0.0.4.7] PWND:[usbliter8]
```

There is a Python control tool (`usbliter8ctl`) in the repo which allows you to demote production mode or boot raw iBoot ("raw" as in decrypted and free of any container).

The control tool depends on `pyusb` that you can get from **pip**.

```
➜  usbliter8 git:(main) ✗ ./usbliter8ctl
usage: usbliter8ctl [-h] {boot,demote} ...

Love is Control

positional arguments:
  {boot,demote}
    boot         boot raw iBoot
    demote       demote production mode

options:
  -h, --help     show this help message and exit
```

## Building

If you want to compile the firmware yourself, you need to get CMake, [Pico SDK](https://github.com/raspberrypi/pico-sdk), [picotool](https://github.com/raspberrypi/picotool) (depends on `libusb`) and [ARM toolchain](https://developer.arm.com/downloads/-/arm-gnu-toolchain-downloads).

Due to the racy nature of the exploit, code changes (even seemingly unrelated) might decrease reliability. We believe this is due to RP2350 executing code from external QSPI flash. The actual instructions can go to a little cache, and then evictions from that cache break timings and then the entire thing breaks.

Here are the exact versions of Pico SDK and ARM toolchain that we used:

* Pico SDK - SDK Release 2.2.0 (commit `a1438dff1d38bd9c65dbd693f0e5db4b9ae91779`)
* ARM toolchain - 15.2.Rel1

```
cd usbliter8
cmake -S . -B build -DPICO_BOARD=$PICO_BOARD -DPICO_SDK_PATH=$PICO_SDK_PATH -DPICO_TOOLCHAIN_PATH=$PICO_TOOLCHAIN_PATH
cmake --build build
```

Change `$PICO_SDK_PATH` and `$PICO_TOOLCHAIN_PATH` to your paths. `$PICO_BOARD` is one of the following:

* waveshare_rp2350_usb_a
* waveshare_rp2350_zero
* pimoroni_tiny2350
* pico2

Even though usage of RP2040 is discouraged, you can still try:

* adafruit_feather_rp2040
* pico

You can also add your own board definition file to `/board` catalog. The board name must match board name in Pico SDK.

Or use the "unknown" one (picked by default for not supported boards). It uses GPIO12/13 for USB data signals and disables LED.

## Authors

* [@__gsch](https://x.com/__gsch) of Paradigm Shift - bug & exploitation
* [@hdesk@infosec.exchange](https://infosec.exchange/@hdesk/) of Paradigm Shift - exploitation
* **REDACTED** of Paradigm Shift - post-exploitation & development

## Credits

* **sekigon-gonnoc** - for the [PIO USB library](https://github.com/sekigon-gonnoc/Pico-PIO-USB)
