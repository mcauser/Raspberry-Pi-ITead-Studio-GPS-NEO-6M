# Raspberry Pi + ITead Studio GPS
## (u-blok NEO-6M)

[ITead Studio product page](http://imall.iteadstudio.com/development-platform/raspi/raspberry-pi-gps-add-on.html)

![Raspberry Pi + ITead Studio GPS](http://i.imgur.com/1mwTp90.jpg)

## Materials

[Raspberry Pi - Mobel B](http://raspberrypiaustralia.com.au/collections/raspberry-pi-boards/products/raspberry-pi-model-b)

[ITead Studio Raspberry PI GPS Add-on](http://raspberrypiaustralia.com.au/collections/crusts-add-ons/products/raspberry-pi-gps-add-on)

[0.5m USB cable](http://raspberrypiaustralia.com.au/collections/cables-headers/products/0-5m-usb-a-male-to-micro-b-lead)

[USB Power Supply (5V 2.1 Amp)](http://raspberrypiaustralia.com.au/collections/power-supply/products/raspberry-pi-power-supply)

[SanDisk Ultra 16GB MicroSDHC Class 10 UHS-1 + Adapter 30MB/s (SDSDQUA-016G-U46A)](http://www.sandisk.com/products/memory-cards/microsd/ultra-class10-for-android/?capacity=16GB)

Or any [compatible SDHC/SDXC card](http://elinux.org/RPi_SD_cards) or MicroSDHC + adapter, with Class >= 10 for greater performance.

## Setup

1. Install Raspbian OS
2. Prepare serial port
3. Install apps
4. Test

### Install Raspbian OS

Download the latest Raspbian Wheezy image from [raspberrypi.org](http://www.raspberrypi.org/downloads/) (around 2.9GB)

[Prepare your SD card](http://computers.tutsplus.com/articles/how-to-flash-an-sd-card-for-raspberry-pi--mac-53600).

Boot and run [raspi-config](http://elinux.org/RPi_raspi-config), expand root fs, enable ssh, change root pass, set hostname, set timezone, set keyboard layout etc.

### Prepare serial port

The Broadcom UART (or serial port) appears as `/dev/ttyAMA0` under Linux.

By default the UART is enabled to allow you to connect a terminal window and login.

You need to disable this to free it up for the GPS Module for a serial connection.

#### Disable kernel logging

:warning: **Be careful doing this, as a faulty command line can prevent the system booting.**

Edit the boot options so the UART does not provide a terminal connection by default.

By default, upon boot sequence the entire boot procedure is output on the UART serial port, which might confuse your micro controller which doesn't expect all that gibberish.

eg. If you see a ton of messages like this appear, you are logging your kernel messages to the console.

    Jun 06 23:53:30 mmmpie kernel: [  289.077967] SysRq : HELP : loglevel(0-9) reboot(b) crash(c) terminate-all-tasks(e) memory-full-oom-kill(f) debug(g) kill-all-tasks(i) thaw-filesystems(j) sak(k) show-memory-usage(m) nice-all-RT-tasks(n) poweroff(o) show-registers(p) show-all-timers(q) unraw(r) sync(s) show-task-states(t) unmount(u) show-blocked-tasks(w)

Take a backup:

```
$ sudo cp /boot/cmdline.txt ~/cmdline.txt.bak
```

Disable the kernel log output on /dev/ttyAMA0:

```
$ sudo nano /boot/cmdline.txt
```

Change:

```
dwc_otg.lpm_enable=0 console=ttyAMA0,115200 kgdboc=ttyAMA0,115200 console=tty1 root=/dev/mmcblk0p2 rootfstype=ext4 elevator=deadline rootwait
```

To:

```
dwc_otg.lpm_enable=0 console=tty1 root=/dev/mmcblk0p2 rootfstype=ext4 elevator=deadline rootwait
```

**Make sure you do not add any line breaks - it must all stay on a single line.**

(you are removing these two commands:)

```
console=ttyAMA0,115200
kgdboc=ttyAMA0,115200
```

The `console` keyword outputs messages during boot, and the `kgdboc` keyword enables kernel debugging.

`ttyAMA0` is the device (/dev/ttyAMA0) and `115200` is the baud rate.

#### Disable spawn login on serial port

take a backup:

```
$ sudo cp /etc/inittab ~/inittab.bak
```

Edit inittab and comment out the line that spawns a login (getty):

```
$ sudo nano /etc/inittab
```

Near the end of the file...

Change:

```
#Spawn a getty on Raspberry Pi serial line
T0:23:respawn:/sbin/getty -L ttyAMA0 115200 vt100
```

To:

```
#Spawn a getty on Raspberry Pi serial line
#T0:23:respawn:/sbin/getty -L ttyAMA0 115200 vt100
```

#### Reboot

```
$ sudo shutdown -r now
```

After rebooting for the new settings to take effect, you can use `/dev/ttyAMA0` like any normal linux serial port, and you wont get any unwanted traffic confusing the attached devices.

To double check, your changes run:

```
$ cat /proc/cmdline
```

You should see the contents of your `/boot/cmdline.txt` at the end of the line.

### Install Apps

#### Install gpsd

[gpsd](http://en.wikipedia.org/wiki/Gpsd) (acronym for Global Positioning System Daemon) is an [open source project](http://www.catb.org/gpsd/) which provides a daemon which streams GPS data via a TCP socket, allowing you to communicate with a whole host of different GPS devices.

Install with dependencies:

```
$ sudo apt-get install gpsd gpsd-clients python-gps
```

Package gpsd gives you binaries:

* gpsd - interface daemon for GPS receivers
* gpsdctl - tool for sending commands to gpsd over its control socket

Package gpsd-clients gives you binaries:

* cgps - client resembling xgps, but without the pictorial satellite display and able to run on a serial terminal or terminal emulator
* gegps - This program collects fixes from gpsd and feeds them to a running instance of Google Earth for live location tracking
* gpsctl - control the modes of a GPS
* gpsdecode - decode RTCM or AIVDM streams into a readable format
* gpsmon - real-time GPS packet monitor and control utility
* gpspipe - tool to connect to gpsd and retrieve sentences
* gpxlogger - This program collects fixes from gpsd and logs them to standard output in GPX, an XML profile for track logging
* lcdgps - A client that passes gpsd data to lcdproc, turning your car computer into a very expensive and nearly feature-free GPS receiver
* xgps - simple test client for gpsd with an X interface
* xgpsspeed - speedometer that uses position information from the GPS

Package python-gps gives you binaries:

* gpscat - dump the output from a GPS
* gpsfake - test harness for gpsd, simulating a GPS
* gpsprof - profile a GPS and gpsd, plotting latency information

#### Run gpsd

The gpsd service needs to be started, using the following command:

```
$ sudo service gpsd start
```

You can also run it directly:

```
$ sudo gpsd -F /var/run/gpsd.sock /dev/ttyAMA0
```

Once gpsd is running, you can telnet to port 2947. You should see a greeting line that's a JSON object describing GPSD's version.

Type `?WATCH={"enable":true,"json"};` to start raw and watcher modes.

You should see lines beginning with `{` that are JSON objects representing reports from your GPS; these are reports in GPSD protocol.

#### Test gpsd using cpgs

There is a simple GPS client which you can run to test everything is working:

```
$ cgps -s
```

It may take a few seconds for the data to come through.

This is what it looks like:

![cgps](http://i.stack.imgur.com/s8RmG.png)

If all you get is timeouts, see below.

If you are cold-starting a brand new GPS, it may take 15-20 minutes after it gets a skyview for it to download an ephemeris and begin delivering fixes.

## Troubleshooting

```
$ gpsd -h

usage: gpsd [-b] [-n] [-N] [-D n] [-F sockfile] [-G] [-P pidfile] [-S port] [-h] device...
  Options include: 
  -b		     	    = bluetooth-safe: open data sources read-only
  -n			    = don't wait for client connects to poll GPS
  -N			    = don't go into background
  -F sockfile		    = specify control socket location
  -G         		    = make gpsd listen on INADDR_ANY
  -P pidfile	      	    = set file to record process ID 
  -D integer (default 0)    = set debug level 
  -S integer (default 2947) = set port for daemon 
  -h		     	    = help message 
  -V			    = emit version and exit.
A device may be a local serial device for GPS input, or a URL of the form:
     {dgpsip|ntrip}://[user:passwd@]host[:port][/stream]
     gpsd://host[:port][/device][?protocol]
in which case it specifies an input source for GPSD, DGPS or ntrip data.
```

### Check the baud rate

If `cgps` repeatedly times out, try running debug mode in the foreground:

```
$ sudo gpsd -b -N -D 3 -n -F /var/run/gpsd.sock /dev/ttyAMA0
```

This GPS's [specifications](http://imall.iteadstudio.com/development-platform/raspi/raspberry-pi-gps-add-on.html) lists the baud rate at 38400 and only starts working after gpsd attempts 4800, 9600, 19200 then finally 38400.

Set the baud rate of port /dev/ttyAMA0 to 38400:

```
$ sudo stty -F /dev/ttyAMA0 38400
```

(after reboot, baud rate returns to default 115200)

### Don't wait for the GPS

Some GPS models require you to use the -n option, which instructs gpsd not to wait for client connection to poll the GPS. This GPS does not require it.

```
$ sudo gpsd -n -F /var/run/gpsd.sock /dev/ttyAMA0
```

### Take ownership of the serial port

To access the serial power without having to become root, you will need to take ownership of the serial port.
It is as easy as adding the group `dialout` to your login id. In this example user = `pi`. You may need to add the group to other users, such as `www-data` for your web server.

Check which groups the current user is in:

```
$ id | grep dialout
```

Add the `pi` user to the `dialout` group:

```
$ sudo usermod -a -G dialout pi
```

### Check for other processes using the serial port

```
$ ps aux | grep ttyAMA0
```

### Attempt to identify the device

```
$ gpsctl /dev/ttyAMA0
```

### Dump raw output from device

`gpscat` is a simple program for logging and packetizing GPS data streams.

It takes input from a specified file or serial device (presumed to have a GPS attached) and reports to standard output. The program runs until end of input or it is interrupted by `^C` or other means. It does not terminate on a bad backet; this is intentional.

```
$ gpscat -s 38400 /dev/ttyAMA0
```

### Unexpected or out of date ephemeris

Part of the message broadcast by a GPS satellite is called the [ephemeris](http://en.wikipedia.org/wiki/Ephemeris). The broadcast ephemeris contains information about the approximate locations of the other GPS satellites in the constellation one might expect to see overhead at a particular place, date and time. Once received by your ground based GPS receiver, the ephemeris is used to hunt for the signals of other GPS satellites in the constellation expected overhead.

If your GPS receiver has not been switched on for a long time its ephemeris may be out of date. Likewise if your GPS receiver's current on-board ephemeris is related to a location in another part of the world, perhaps where it was constructed, tested or last used, then the expected constellation may be significantly different to what is overhead. This is why a brand new GPS can sometimes take up to 20 minutes before it starts giving a position, it's downloading the broadcast ephemeris. Usually the broadcast ephemeris will be automatically stored so the next time you switch on, unless you've gone half way round the world, the problem should not occur. If it does then there may be a problem with your GPS unit.

## Documentation and specifics

### ITead Studio GPS Expansion Board

[Raspberry PI GPS Add-on](http://imall.iteadstudio.com/development-platform/raspi/raspberry-pi-gps-add-on.html) is customised for Raspberry Pi interface based on u-blok NEO-6 GPS module. Through the serial port on Raspberry Pi, data returned from GPS module can be received, thus information such as the current location and time can be searched.

Model: IM131227001

[GPS Add-on Data Sheet](ftp://imall.iteadstudio.com/Modules/IM131227001/DS_IM131227001.pdf)

[GPS Add-on Schematic](ftp://imall.iteadstudio.com/Modules/IM131227001/SCH_IM131227001.pdf)

### u-blok NEO-6M GPS Module

The [u-blok NEO-6 module](http://www.u-blox.com/en/gps-modules/pvt-modules/previous-generations/neo-6-family.html) series brings the high performance of the u-blox 6 position engine to the miniature NEO form factor. u-blox 6 has been designed with low power consumption and low costs in mind. Intelligent power management is a breakthrough for low-power applications.

Model: NEO-6M-0-001

[NEO-6 Product Summary](http://www.u-blox.com/images/downloads/Product_Docs/NEO-6_ProductSummary_%28GPS.G6-HW-09003%29.pdf)

[NEO-6 Data Sheet](http://www.u-blox.com/images/downloads/Product_Docs/NEO-6_DataSheet_%28GPS.G6-HW-09005%29.pdf)

[NEO-6 Hardware Integration Manual](http://www.u-blox.com/images/downloads/Product_Docs/LEA-6_NEO-6_MAX-6_HardwareIntegrationManual_%28GPS.G6-HW-09007%29.pdf)

[u-blox 6 Receiver Description and Protocol Specification](http://www.u-blox.com/images/downloads/Product_Docs/u-blox6_ReceiverDescriptionProtocolSpec_%28GPS.G6-SW-10018%29.pdf)

[Additional documentation and resources](http://www.u-blox.com/en/download/documents-a-resources/u-blox-6-gps-modules-resources.html)
