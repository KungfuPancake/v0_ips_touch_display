# A better Voron V0 KlipperScreen Display
![Mounted Display on a Voron V0](https://github.com/user-attachments/assets/242b8273-a739-4a9a-9520-3f2ceb1d92aa)
*Warning:* This is (currently) not a straightforward mod. You will need to:
* have PCBs made and solder SMD components including a 40 pin connector with a 0.5mm pin pitch
* build a wiring harness
* build a custom Linux kernel or a complete OS image
* possibly write and then apply a DTS overlay to make the kernel aware of the display

In return you get a very high quality 3.5" display on your V0 that has just the right size to not look out of place for a very low price.

Most of the hoops you'll have to jump through stem from the display controller (ILI9488). It is MIPI-DBI compatible, but contrary to its predecessors it does not support 16 bit color modes when using the SPI interface.
This means more data has to be pushed into the display (three transactions per pixel instead of two) and the color format has to be set correctly. The first problem is solved by increasing the SPI speed, but support for non 16 bit color formats have only recently been added to the MIPI-DBI kernel module. This means you will currently have to roll your own Kernel on at least Armbian and Raspberry Pi OS.

Once Linux 6.11 has been adopted by Raspberry Pi OS/Armbian a big chunk of that work will go away.

## BOM
### Mechanics
* ~20g Filament (primary color)
* ~20g Filament (accent color)
* 4x M3x3.7x3 heat set insert
* 4x M3x4 SHCS
* 4x M2x4 self tapping
### Electronics
* 1x [ILI9488 IPS TFT display with FT6236 touch on Aliexpress](https://www.aliexpress.com/item/1005001315519214.html)
* 1x PCB - use e.g. JLCPCB
* 1x 40pin 0.5 pitch FPC connector (make sure the flat flex contacts are on the bottom side)
* 1x 6pin 0.5 pitch FPC connector (make sure the flat flex contacts are on the bottom side)
* 2x 15k 0603
* 1x 100 0603
* 1x IRLML2502 N-channel MOSFET
* 1x 6x2 2.54mm header
* 1x 6x2 Dupount housing + female crimp pins
* 28AWG wire

* whatever you need to connect the wires to your SBC, plus heatshrink/sleeving/whatever you feel like doing. go wild!

## Instructions
### Printed parts
The display uses two parts for mounting: a skirt adapter and the display frame.
For the skirt adapter we're currently using this one [Voron0.2 2.8 Waveshare Display by hartk1213](https://mods.vorondesign.com/details/ig1ygdPg4TS0C5XY6HTw). Print this in the same material and color as your other skirt elements.

The display frame is a bit tricky. The display does not have any mounting points and the project started off with the backside being attached by double sided adhesive tape, but that turned out to be quite unreliable and destructive in case the display had to be removed. We have sinced moved to a clip in design that makes sure that the display always stays in place and can be removed again. To make sure that the display clips in easily but is then held securely, you'll have to get the dimensions just right. Even if you dialed in your ABS shrinkage correctly, multiple attempts might be necessary. Start with a 101% scaling and work your way down. In case you have to remove the display from the frame you might have to cut the border with pliers. Be extremely careful not to crack the glass! At least one display died in the making of this frame.
Remember to turn on build plate supports when printing, as the PCB is recessed in the backside of the frame. Built-in supports might come in the future.

### PCB
![pcb](https://github.com/user-attachments/assets/d58cfda2-b134-403d-af5a-29b13cfef3d7)
Since the Display just exposed two bare flat flex connectors, an adapter PCB is needed. The gerber files in this repository can be used to order PCBs at e.g. [JLCPCB](https://jlcpcb.com/). Most settings don't matter, but set the PCB thickness to 0.8mm as the cutout in the printed frame has that exact depth. Solder on the parts from the BOM, starting with the FPC connectors and finish with the pin header. You should test the display with the PCB attached before mounting everything. Just make sure that the PCB does not contact the bare metal shield on the back of the display. A piece of kapton tape works great.

### Wiring harness
Build the wiring harness by attaching a 6x2 Dupont housing with female crimps on the display side and a connector suitable for your SBC (likely with Dupont, too) on the other side. Either use the pinouts for the tested boards or work out your own by using the schematics provided by your board manufacturer and whatever you can get working on it. Keep the wiring harness as short as possible and the length for the SPI lines as equal as possible. We're gonna send serial data with up to 64MHz through it.

### Assembly
![IMG_4101](https://github.com/user-attachments/assets/004e1b84-8833-459d-809d-be9d2162ac60)

Start by melting in the heat set inserts. Put a screw into every insert and remove any leftover material as it may interfer with the display. Then clip in the display while carefully guiding the flat flex through the cutout. Lastly, attach the flat flex to the FPC connectors, then screw down the PCB into the mounting location.
Test the display again and if everything works, attach the display frame to the skirt adapter with this M3x4 SHCS screws. The two screws on the top side will barely reach the insert, but depending on tolerances, using M3x6 might interfer with the display body. Use your best judgement.

### SBC
Despite claiming to follow the Raspberry Pi GPIO header scheme, most non-RPI SBCs have some differences on the pinout and the peripherals.
If your SBC is not listed here, try to find some documentation or schematics for your board. You will need:
* 1x SPI with one CS/CE line
* 1x I²C
* 1x GPIO with IRQ capability
* 4x GPIO outputs, optionally with one PWM capable pin

### Raspberry Pi OS
TBD

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

Write the image to your SD card, then boot the system. After the initial setup, proceed with cloning this repository, then generate the firmware file for the display:
```
git clone --recurse-submodules https://github.com/KungfuPancake/v0_ips_touch_display.git
cd v0_ips_touch_display/
./panel-mipi-dbi/mipi-dbi-cmd /lib/firmware/panel-mipi-dbi-spi.bin panel-mipi-dbi-spi.txt
```
Please be aware that this firmware file contains a difference to most ILI9488 init sequences you'll find online. -VCOM has to be set to the correct value, otherwise you'll get a glow effect and eventually a display burnout. The correct values are: `command 0xC5 0x00 0x4D 0x80`. If you want to write your own init sequence or use some other preexisting one, remember to add those values!

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
#### Raspberry Pi
| Pin   |    |    | Pin   |
|-------|----|----|-------|
|       | 1  | 2  |       |
| SDA   | 3  | 4  |       |
| SCL   | 5  | 6  |       |
| T_RST | 7  | 8  |       |
|       | 9  | 10 |       |
|       | 11 | 12 | LED   |
|       | 13 | 14 |       |
|       | 15 | 16 | INT   |
| 3V    | 17 | 18 | DC    |
| MOSI  | 19 | 20 |       |
|       | 21 | 22 | L_RST |
| SCK   | 23 | 24 | CS    |
| GND   | 25 | 26 |       |
|       | 27 | 28 |       |
|       | 29 | 30 |       |
|       | 31 | 32 |       |
|       | 33 | 34 |       |
|       | 35 | 36 |       |
|       | 37 | 38 |       |
|       | 39 | 40 |       |

#### Orange Pi PC
| Pin   |    |    | Pin   |
|-------|----|----|-------|
|       | 1  | 2  |       |
| SDA   | 3  | 4  |       |
| SCL   | 5  | 6  |       |
| LED   | 7  | 8  | INT   |
|       | 9  | 10 |       |
| T_RST | 11 | 12 |       |
|       | 13 | 14 |       |
|       | 15 | 16 |       |
| 3V    | 17 | 18 | DC    |
| MOSI  | 19 | 20 |       |
|       | 21 | 22 | L_RST |
| SCK   | 23 | 24 | CS    |
| GND   | 25 | 26 |       |
|       | 27 | 28 |       |
|       | 29 | 30 |       |
|       | 31 | 32 |       |
|       | 33 | 34 |       |
|       | 35 | 36 |       |
|       | 37 | 38 |       |
|       | 39 | 40 |       |

```
./compile.sh BOARD=orangepipc BRANCH=edge kernel-config
./compile.sh BOARD=orangepipc BRANCH=edge BUILD_DESKTOP=no BUILD_MINIMAL=yes KERNEL_CONFIGURE=no RELEASE=bookworm
```
```
armbian-add-overlay orangepi-pc.dts
```

#### Orange Pi Zero 3
| Pin   |    |    | Pin   |
|-------|----|----|-------|
|       | 1  | 2  |       |
| SDA   | 3  | 4  |       |
| SCL   | 5  | 6  |       |
|       | 7  | 8  | T_RST |
|       | 9  | 10 | LED   |
|       | 11 | 12 |       |
|       | 13 | 14 |       |
|       | 15 | 16 |       |
| 3V    | 17 | 18 | INT   |
| MOSI  | 19 | 20 |       |
|       | 21 | 22 | DC    |
| SCK   | 23 | 24 | CS    |
| GND   | 25 | 26 | L_RST |

```
./compile.sh BOARD=orangepizero3 BRANCH=edge kernel-config
./compile.sh BOARD=orangepizero3 BRANCH=edge BUILD_DESKTOP=no BUILD_MINIMAL=yes KERNEL_CONFIGURE=no RELEASE=bookworm
```
```
armbian-add-overlay orangepi-zero3.dts
```

#### Bigtreetech Pi V1.2
| Pin   |    |    | Pin   |
|-------|----|----|-------|
| 3V    | 1  | 2  |       |
| CS    | 3  | 4  |       |
| SCK   | 5  | 6  |       |
|       | 7  | 8  |       |
| GND   | 9  | 10 |       |
| L_RST | 11 | 12 | T_RST |
| DC    | 13 | 14 |       |
| INT   | 15 | 16 | LED   |
|       | 17 | 18 |       |
| SDA   | 19 | 20 |       |
|       | 21 | 22 |       |
| SCL   | 23 | 24 |       |
|       | 25 | 26 |       |
| MOSI  | 27 | 28 |       |
|       | 29 | 30 |       |
|       | 31 | 32 |       |
|       | 33 | 34 |       |
|       | 35 | 36 |       |
|       | 37 | 38 |       |
|       | 39 | 40 |       |

```
./compile.sh BOARD=bigtreetech-cb1 BRANCH=edge kernel-config
./compile.sh BOARD=bigtreetech-cb1 BRANCH=edge BUILD_DESKTOP=no BUILD_MINIMAL=yes KERNEL_CONFIGURE=no RELEASE=bookworm
```
```
armbian-add-overlay btt-pi.dts
```



