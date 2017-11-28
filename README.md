# Raspberry Pi Model Zero
[![CircleCI](https://circleci.com/gh/nerves-project/nerves_system_rpi0.svg?style=svg)](https://circleci.com/gh/nerves-project/nerves_system_rpi0)
[![Hex version](https://img.shields.io/hexpm/v/nerves_system_rpi0.svg "Hex version")](https://hex.pm/packages/nerves_system_rpi0)

This is the base Nerves System configuration for the Raspberry Pi Zero and
Raspberry Pi Zero W.

If you are *not* interested in [Gadget Mode](http://www.linux-usb.org/gadget/) it might be worth checking out [nerves_system_rpi](https://github.com/nerves-project/nerves_system_rpi).
That image configures the USB port in host mode by default and
is probably more appropriate for your setup.

![Fritzing Raspberry Pi Zero image](assets/images/raspberry-pi-model-zero.png)
<br><sup>[Image credit](#fritzing)</sup>

| Feature              | Description                     |
| -------------------- | ------------------------------- |
| CPU                  | 1 GHz ARM1176JZF-S              |
| Memory               | 512 MB                          |
| Storage              | MicroSD                         |
| Linux kernel         | 4.9 w/ Raspberry Pi patches     |
| IEx terminal         | OTG USB serial port (`ttyGS0`). Can be changed to HDMI or UART. |
| GPIO, I2C, SPI       | Yes - Elixir ALE                |
| ADC                  | No                              |
| PWM                  | Yes, but no Elixir support      |
| UART                 | 1 available - `ttyAMA0`         |
| Camera               | Yes - via rpi-userland          |
| Ethernet             | Yes - via OTG USB port          |
| WiFi                 | Supported on the Pi Zero W      |
| Bluetooth            | Not supported yet               |

## Console and kernel message configuration

The goal of this image is to use the OTG port for console access. If you're
debugging the boot process, you'll want to use the Raspberry Pi's UART pins on
the GPIO connector or the HDMI output. This is enabled by updating the
`cmdline.txt` file. This may be overrided with a custom `fwup.conf` file if
you don't want to rebuild this system. Add the following to your `cmdline.txt`:

```
console=ttyAMA0,115200 console=tty1 ...
```

If you'd like the IEx prompt to come out the UART pins (`ttyAMA0`) or HDMI
(`tty1`), then modify `rootfs_overlay/etc/erlinit.config` as well.

## Supported OTG USB modes

The base image activates the `dwc2` overlay, which allows the Pi Zero to appear as a
device (aka gadget mode). When plugged into a host computer via the OTG port, the Pi
Zero will appear as a composite Ethernet and serial device. The virtual serial
port provides access to the IEx prompt and the Ethernet device can be used for
firmware updates, Erlang distribution, and anything else running over IP.

## Supported WiFi devices

The base image includes drivers for the onboard Raspberry Pi Zero W wifi module
(`brcmfmac` driver). Due to the USB port being placed in gadget mode, this
system does not support USB WiFi adapters.

## Linux kernel and RPi firmware/userland

There's a subtle coupling between the `nerves_system_br` version and the Linux
kernel version used here. `nerves_system_br` provides the versions of
`rpi-userland` and `rpi-firmware` that get installed. I prefer to match them to
the Linux kernel to avoid any issues. Unfortunately, none of these are tagged
by the Raspberry Pi Foundation so I either attempt to match what's in Raspbian
or take versions of the repositories that have similar commit times.

## Installation

If you're new to Nerves, check out the
[nerves_init_gadget](https://github.com/fhunleth/nerves_init_gadget) project for
creating a starter project for the Raspberry Pi Zero or Zero W. It will get you
started with the basics like bringing up the virtual Ethernet interface,
initializing the application partition, and enabling ssh-based firmware updates.

## Linux kernel configuration notes

The Linux kernel compiled for Nerves is a stripped down version of the default
Raspberry Pi Linux kernel. This is done to remove unnecessary features, select
some Nerves-specific features, and to save space. To reproduce the kernel
configuration found here, do the following (this is somewhat tedious):

1. Start with `arch/arm/configs/bcmrpi_defconfig`. This is the kernel
   configuration used in the official Raspberry Pi images.
2. Turn off all filesystems except for `ext4`, `squashfs`, `tmpfs`, `proc`,
   `sysfs`, and `vfat`. Squashfs only needs ZLIB support.
3. `vfat` needs to default to `utf8`. Enable native language support for
   `ascii`, `utf-8`, `ISO 8859-1`, codepage 437, and codepage 850.
4. Disable all network drivers and wireless LAN drivers except for Broadcom
   FullMAC WLAN.
5. Disable PPP and SLIP
6. Disable the WiFi drivers in the Staging drivers menus
7. Disable TV, AM/FM, Media USB adapters, DVB Frontends and Remote controller
   support in the Multimedia support menus.
8. Go to `Device Drivers->Sound card support`. Disable `USB sound devices` in
   ALSA. Disable `Open Sound System`.
9. Go to `Device Drivers->Graphics support`. Disable `DisplayLink`
10. Disable everything in `HID support` (NOTE: revisit for Bluetooth)
11. Disable everything in input device support (can't plug it in anyway)
12. In the `Device Drivers > USB support` menu, enable gadget mode and disable
   all host mode. It should be possible to completely disable USB host mode if
   all of the USB drivers in previous steps were disabled. See `DesignWare USB2
   Core Support->DWC Mode Selection` and select `CDC Composite Device (Ethernet
   and ACM)`. If you want dual mode USB host/gadget support, you'll need to
   reenable a few things. There have been unresolved issues in the past with dual
   mode support. It's possible that they are fixed, but be sure to test. They were
   noticed on non-Mac platforms.
13. In `Kernel Features`, select `Preemptible Kernel (Low-Latency Desktop)`,
    disable the memory allocator for compressed pages.
14. In `Userspace binary formats`, disable support for MISC binaries.
15. In `Networking support`, disable Amateur Radio support, CAN bus subsystem,
    IrDA subsystem, Bluetooth, WiMAX, Plan 9, and NFC. (TBD - this may be too
    harsh, please open issues if you're using any of these and it's the only
    reason for you to create a custom system.)
16. In `Networking options`, disable IPsec, SCTP, Asynchronous Transfer Mode,
    802.1d Ethernet Bridging, L2TP, VLAN, Appletalk, 6LoWPAN, 802.15.4, DNS
    Resolver, B.A.T.M.A.N, Open vSwitch, MPLS, and the Packet Generator in Network
    testing.
17. In `Network options->Wireless`, enable "use statically compiled regulatory
    rules database". Build in `cfg80211` and `mac80211`. Turn off `mac8022` mesh
    networking and LED triggers. Turn off `cfg80211` wireless extensions
    compatibility.
18. In `Kernel hacking`, disable KGDB, and Magic SysRq key.
19. In Device Drivers, disable MTD support. In Block devices, deable everything
    but Loopback and RAM block device. Disable SCSI device support. Disable RAID
    and LVM.
20. In `Enable the block layer`, deselect everything but the PC BIOS partition
    type (i.e., no Mac partition support, etc.).
21. In `Enable loadable module support`, select "Trim unused exported kernel
    symbols". NOTE: If you're having trouble with an out-of-tree kernel module
    build, try deslecting this!!
22. In `General Setup`, turn off `initramfs/initfd` support, Kernel .config
    support, OProfile.
23. In `Device Drivers -> I2C` compile the module into the kernel and disable
    everything but `BCM2708 BSC` support.
24. In `Device Drivers -> SPI` compile in the BCM2835 SPI controller and User
    mode SPI device driver support.
25. In `Device Drivers -> Dallas's 1-wire support`, disable everything but the
    GPIO 1-Wire master and the thermometer slave. (NOTE: Why is the thermometer
    compiled in? This seems historical.)
26. Disable `Hardware Monitoring support`, `Sonics Silicon Backplane support`
27. In `Device Drivers -> Character devices -> Serial drivers`, disable 8250 and
    SC16IS7xx support. Disable the RAW driver.
28. In `Networking support->Network options`, disable `IP: kernel level
    autoconfiguration`
29. In `Networking support->Network options->TCP: advanced congestion control`
    disable everything except for `CUBIC TCP`.
30. Disable `Real Time Clock`.
31. Disable everything in `Cryptographic API` and `Library routines` that can be
    disabled. Sometimes you need to make multiple passes.
32. Disable EEPROM 93CX6 support, PPS support, all GPIO expanders, Speakup core,
    Media staging drivers, STMicroelectronics STMPE, anything "Wolfson".
33. Disable most ALSA for SoC audio support and codecs. NOTE: We probably should
    support a few, but I have no clue which ones are most relevant and there are
    tons of device drivers in the list.
34. Disable IIO and UIO.
35. Disable NXP PCA9685 PWM driver


[Image credit](#fritzing): This image is from the [Fritzing](http://fritzing.org/home/) parts library.
