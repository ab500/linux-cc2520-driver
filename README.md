linux-cc2520-driver
===================

A kernel module that powers the CC2520 802.15.4 radio in Linux.

The general idea is to create a character driver that exposes 802.15.4 frames
to the end user, allowing them to test and layer different networking stacks
from user-land onto the CC2520 radio. We're specifically targeting running
the IPv6 TinyOS networking stack on a Raspberry Pi.

To compile you'll need an ARM cross compiler and the source tree of a compiled
ARM kernel. Update the Makefile to point to your kernel source, and run the
following command:

make ARCH=arm CROSS_COMPILE=arm-linux-gnueabi-

Please note that we're cross-compiling from an x86 machine.

More info can be found [on our lab wiki.](http://energy.eecs.umich.edu/wiki/doku.php?id=proj:rpibr:start)


Installing
----------

Currently this process requires compiling against the 3.6 version of the Linux
kernel from the Raspberry Pi's GitHub (not the 3.2 standard version).

### Step One: Setup a 3.6 Kernel Source

First you'll need to get the Raspberry Pi 3.6 kernel source tree onto your
computer. To do this make some new folders

```
mkdir rpi
cd rpi
```

First clone this repository.

    git clone git@github.com:ab500/linux-cc2520-driver.git


Then clone the 3.6.y kernel. We use this kernel because it has a much better
implementation of the generic spi driver. We're going to use the code by msperl
to get a really low-latency SPI implementation.

```
git clone git://github.com/raspberrypi/linux.git
```

Now you need to patch two files in the linux tree. The first changes a small
part of the device configuration to let the SPI driver know where it can find
DMA-accessible memory. The second updates the spi driver to msperl's version and
changes the spi driver to deassert the chip-select line. Not keeping the
`SPI_CS_TA` flag active will have the SPI driver toggle the CS pin between
transmissions. The CC2520 radio requires a SPI toggle to terminate command
sequences that don't have a predetermined length.

    cd linux
    git apply ../linux-cc2520-driver/patches/bcm2708.patch
    git apply ../linux-cc2520-driver/patches/spi-bcm2708.patch

Finally you're ready to compile the kernel for the Pi. First make sure you have
the right build environment. You'll need an ARM GCC cross-compiler installed
(arm-linux-gnueabi-gcc).

Follow the instructions on the Raspberry Pi main site here, ignoring the part
that checks out the kernel from source, you already have the kernel on your
machine.

http://elinux.org/RPi_Kernel_Compilation

**Some Tips**
  * I found the best way to do things was to start with the Raspbian image for the 3.2 kernel,
    make sure it works on the Pi, and then mount the SD card on your computer and replace
    the kernel. To do this you'll need to replace the kernel on the boot partition, the modules
    directory, and the firmware. The tutorial above walks you through almost all of this.
  * I would checkout the tools and firmware repositories into your rpi directory just because
    it's a little cleaner and keeps everything in the same place.

When you're done, log into your pi and make sure you're now running the 3.6 kernel:

```
pi@raspberrypi ~ $ uname -a
Linux raspberrypi 3.6.11+ #3 PREEMPT Tue Dec 25 13:31:30 EST 2012 armv6l GNU/Linux
```

### Step Two: Enable the SPI driver at boot
Now that we've gotten the kernel installed we need to enable automatic loading
of the SPI driver at boot. Edit <code>/etc/modprobe.d/raspi-blacklist.conf</code>
and comment out the line that blacklists the SPI driver from boot. It should look
like this when you're done:

```
# blacklist spi and i2c by default (many users don't need them)

#blacklist spi-bcm2708
blacklist i2c-bcm2708
```

Reboot the Pi. Watch the boot messages, or consult /var/log/kern.log for
the following line to ensure the SPI driver is loaded:

```
[   14.914109] bcm2708_spi bcm2708_spi.0: DMA channel 0 at address 0xf2007000 with irq 16
[   15.098289] bcm2708_spi bcm2708_spi.0: DMA channel 4 at address 0xf2007400 with irq 20
[   15.216224] spi_master spi0: will run message pump with realtime priority
[   15.279480] bcm2708_spi bcm2708_spi.0: SPI Controller at 0x20204000 (irq 80)
[   15.469266] bcm2708_spi bcm2708_spi.0: SPI Controller running in interrupt-driven mode
```

### Step Three: Checkout and Build This Module

Finally we're ready to compile/load this module and really get cooking.

```
cd ~\rpi\linux-cc2520-driver
make ARCH=arm CROSS_COMPILE=arm-linux-gnueabi-
```

You should see some build output and end up with a file called cc2520.ko.

### Step Four: Move the module and support files to the pi

Now lets move this module to the Pi, as well as some support files to test and
do other things.

To do this I use SFTP.

```
sftp pi@your-pi-hostname

sftp> put cc2520.ko
sftp> put tests/read.c
sftp> put tests/write.c
sftp> put ioctl.h
sftp> put setup.sh
sftp> put reload.sh

```

### Step Five: Load the Module and Build Test Utilities

We're ready to finally test things. Move to the Pi now, we'll compile
the test utilities on the Pi itself. They test the basic operation of the
Pi. I've also written two really really basic shell scripts to load the module
and set it up.

**To Compile the Test Utilities**

```
gcc write.c -o write
gcc read.c -o read
```

**To Load the Module**

Simply run

    sudo ./reload.sh

in the same directory as cc2520.ko.

**To Test the Driver**

First check the <code>kern.log</code> file for the debug output above. Also make sure your raspberry pi
hasn't frozen and paniced. You're doing pretty good.

Next you'll need another 802.15.4 mote. I've been using Epic-based devices. Use the SimpleSackTest
app in the shed and install it on a mote. It simply sends an 802.15.4 packet with an incrementing
counter in it, requesting an ACK, and will ACK any packets sent to it. On a standard mote the LEDs
are setup to indicate the following:

  * **Blue** - Toggles every time a packet is properly acknowledged by the radio.
  * **Green** - Toggles every time a packet is sent by the mote.
  * **Red** - Toggles every time the mote successfully receives a packet.

When the RPi radio is off only the green light will blink. Running the write and read apps
on the RPi will cause all the lights to blink.

The output will look something like this:

**read.c**
```
Receiving a test message...
result 14
read  0x0D 0x61 0x88 0xD4 0x22 0x00 0x01 0x00 0x01 0x00 0x76 0xD5 0x13 0xEB
Receiving a test message...
result 14
read  0x0D 0x61 0x88 0xD5 0x22 0x00 0x01 0x00 0x01 0x00 0x76 0xD6 0x13 0xEB
Receiving a test message...
result 14
read  0x0D 0x61 0x88 0xD6 0x22 0x00 0x01 0x00 0x01 0x00 0x76 0xD7 0x13 0xEB
Receiving a test message...
result 14
read  0x0D 0x61 0x88 0xD7 0x22 0x00 0x01 0x00 0x01 0x00 0x76 0xD8 0x13 0xEA
```

**write.c**
```
Sending a test message...
result 12
Sending a test message...
result 12
Sending a test message...
result 12
Sending a test message...
result 12
Sending a test message...
result 12
Sending a test message...
result 12
```

Current Status
---------------
The driver is starting to shape up to be feature-complete. Looking for obscure
timing bugs at the moment, but we support LPL, CSMA/CA, unique filtering, and
Soft-ACK features. It's basically a fully-featured radio implementing something
that's really close to the default TinyOS radio stack.

It runs more-or-less a generic, standard MAC layer. Nothing fancy.

Some notes
----------
  * You'll need to load the spi-bcm2708 driver for this module to work. You can do this by commenting out the line in the blacklisted modules file found in /etc.
  * Make sure to update the directory for the kernel in the Makefile
  * Update the GPIO defs in cc2520.h
