# A better Voron V0 KlipperScreen Display
*Warning:* This is (currently) not a straightforward mod. You will need to:
* have PCBs made and solder SMD components including a 40 pin connector with a 0.5mm pin pitch
* build a wiring harness
* build a custom Linux kernel or a complete OS image
* apply a DTS overlay to make the kernel aware of the display

Once Linux 6.11 has been adopted by Raspberry Pi OS/Armbian a big chunk of that work will go away.

## Instructions
### Hardware
Despite claiming to follow the Raspberry Pi GPIO header scheme, most non-RPI SBCs have some differences on the pinout and the peripherals.
If your SBC is not listed here, try to find some documentation or schematics for your board. You will need:
* 1x SPI with one CS/CE line
* 1x I²C
* 1x GPIO with IRQ capability
* 4x GPIO outputs, optionally with one PWM capable pin

### Raspberry Pi OS

### Armbian
Armbian has an easy to set up build environment based on Docker. All operations can be done via a provided shell script.

`git clone https://github.com/armbian/build armbian`

Depending on the target board you may have to change some Kernel options before building. These settings are required:
```
TOUCHSCREEN_EDT_FT5X06=m
BACKLIGHT_GPIO=m
BACKLIGHT_PWM=m
```
Follow the instructions for building the OS image for your SBC or deduct the necessary commands if your SBC is not listed.

#### Orange Pi PC
On the build machine:
```
./compile.sh BOARD=orangepipc BRANCH=edge kernel-config
./compile.sh BOARD=orangepipc BRANCH=edge BUILD_DESKTOP=no BUILD_MINIMAL=yes KERNEL_CONFIGURE=no RELEASE=bookworm
```
Write the image to a SD card (or to another boot media, e.g. SSD or eMMC) and boot the system. Once you have completed the initial setup,
install the DTS overlay and the firmware file.
On the SBC:
```
git clone https://github.com/KungfuPancake/v0_ips_touch_display.git
git clone https://github.com/notro/panel-mipi-dbi.git
cd v0_ips_touch_display/
armbian-add-overlay sun8i-h3-v0disp.dts
../mipi-dbi-cmd/mipi-dbi-cmd /lib/firmware/panel-mipi-dbi-spi.bin panel-mipi-dbi-spi.txt
reboot
```

* Orange Pi Zero 3

* Bigtreetech Pi V1.2




