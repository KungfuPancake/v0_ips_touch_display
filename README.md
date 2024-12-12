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
CONFIG_DRM_PANEL_MIPI_DBI=m
CONFIG_TOUCHSCREEN_EDT_FT5X06=m
CONFIG_BACKLIGHT_GPIO=m
CONFIG_BACKLIGHT_PWM=m
```
You don't need any modules to enable a graphical console, but you'll need a framebuffer device for your display, since X currently does not play well with the modesetting driver for mipi-dpi-spi. `/dev/fb0` usually gets assigned to the HDMI port of your SBC, so watch out for `/dev/fb1`. `dmesg | grep spi` helps to find out if the module was loaded correctly and if a framebuffer was assigned.

Then build the image. Replace the BOARD value with your SBC, e.g. `orangepipc` if you have an Orange Pi PC.
```
./compile.sh BOARD=<your sbc> BRANCH=edge kernel-config
./compile.sh BOARD=<your sbc> BRANCH=edge BUILD_DESKTOP=no BUILD_MINIMAL=yes KERNEL_CONFIGURE=no RELEASE=bookworm
```

Write the image to your SD card, then boot the system. After the initial setup, proceed with these steps:

```
git clone --recurse-submodules https://github.com/KungfuPancake/v0_ips_touch_display.git
cd v0_ips_touch_display/
./mipi-dbi-cmd/mipi-dbi-cmd /lib/firmware/panel-mipi-dbi-spi.bin panel-mipi-dbi-spi.txt
```

Now you will have to procure a DTS overlay for your SBC. If you're lucky you'll find your SBC in the list of known working SBCs and you can just use the existing overlay. If your board hasn't been tested yet you'll have to write your own overlay. This is the hard part. Take inspiration from one of the existing overlays, then find out how the peripheral system of your board works and how to address the bus and gpios. Test your overlay with:

```
armbian-add-overlay <your overlay>
```

If everything goes well, your display and touchscreen should get detected upon boot. You can verify this with `dmesg`. Depending on your kernel config you will get a graphical console, too.
We're mostly done now, just some housekeeping left. First, create a udev rule for the touchscreen to account for the display rotation. You can take the rule file from this repository:

```
cp 51-touchscreen.rules /etc/udev/rules.d/
udevadm control --reload-rules && udevadm trigger
```

If your touchscreen isn't as precise as you'd like, follow the instructions in the [KlipperScreen docs](https://klipperscreen.readthedocs.io/en/latest/Troubleshooting/Touch_issues/) to work out a transformation matrix.

To make X use the framebuffer of your LCD, add a configuration file pointing to the framebuffer you want:
```
cp 99-fbdev.conf /etc/X11/xorg.conf.d/
```
If your framebuffer device is not `/dev/fb1`, change the file contents as necessary.

Now is the time to install the whole Klipper stack, including KlipperScreen. It should just work™️, including touch control and DPMS.

### Known working SBCs
#### Orange Pi PC
Build the image:
```
./compile.sh BOARD=orangepipc BRANCH=edge kernel-config
./compile.sh BOARD=orangepipc BRANCH=edge BUILD_DESKTOP=no BUILD_MINIMAL=yes KERNEL_CONFIGURE=no RELEASE=bookworm
```

#### Orange Pi Zero 3

#### Bigtreetech Pi V1.2
Build the image:
```
./compile.sh BOARD=bigtreetech-cb1 BRANCH=edge kernel-config
./compile.sh BOARD=bigtreetech-cb1 BRANCH=edge BUILD_DESKTOP=no BUILD_MINIMAL=yes KERNEL_CONFIGURE=no RELEASE=bookworm
```



