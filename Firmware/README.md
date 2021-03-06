# Firmware

For more information on the Firmware system, please see the [Firmware page on the Wiki](https://github.com/HaddingtonDynamics/Dexter/wiki/Firmware).

Note that updated versions of DexRun.c may not be able to support new features in the Gateware unless the drive image is updated. For example, the 6 and 7th access (twist and grip of end effector on new tool interface) requires an interface to the servos which is implemented in the FPGAs and so needs an updated image. 

## DexRun.c Manual update

While it is possible to update DexRun.c as documented here, it is very unlikely that this is worth doing or recommended for the average user. It is far better to just request and download a new SD Card image. For programmers, these notes will help you navigate some of the unusual points of the firmware build.

**Log into Dexter.**

[Establish a Network connection and SSH into Dexter](https://github.com/HaddingtonDynamics/Dexter/wiki/Dexter-Networking)

**Change Makefile:** if you haven't already
````
cd /usr/src/xillinux/xillybus-lite/demo
nano Makefile
````
change Makefile to:

````
GNUPREFIX=

CC=$(GNUPREFIX)gcc
AR=$(GNUPREFIX)ar
AS=$(GNUPREFIX)as
CXX=$(GNUPREFIX)g++
LD=$(GNUPREFIX)ld
STRIP=$(GNUPREFIX)strip

CFLAGS=-g -I. -O3 -pthread -lm -Wno-unused-result

APPLICATION=DexRun

INTDEMO=intdemo

OBJECTS=# core.o

all: $(APPLICATION) $(INTDEMO)

$(APPLICATION): $(OBJECTS) $(APPLICATION).o
        $(CC)  $(CFLAGS) $(OBJECTS) $(APPLICATION).o -o $(APPLICATION) -lrt -lm

$(INTDEMO): $(OBJECTS) $(INTDEMO).o
        $(CC)  $(CFLAGS) $(OBJECTS) $(INTDEMO).o -o $(INTDEMO)

clean:
        rm -f *~ $(APPLICATION)  $(INTDEMO) *.o

.c.o:
        $(CC) $(CFLAGS) -c -o $@ $<
````
ctr+x and hit y then enter to save a file from nano.

**Change pg:** pg is just a simple script that makes it easy to make DexRun from the share folder. 
````
cd /srv/samba/share
nano pg
````
change "uiotest" to "DexRun" if you haven't already. It should look like this:
````
#!/bin/sh
echo "changing directory"
cd /usr/src/xillinux/xillybus-lite/demo
cp -f /srv/samba/share/DexRun.c .
make
cp -f DexRun /srv/samba/share/.
````
You might want to rename pg to makeDexRun. This would also be a good place to backup the prior DexRun file so that if there are problems, they can be easily reverted.  

**Setup to auto-run DexRun on bootup:**
````
cd /etc/
nano rc.local
````
change "uiotest" to "DexRun"



**Put DexRun.c into /srv/samba/share/**

If you have access to the share from your PC, just save the file there. E.g. to
\\192.168.1.142\share

Otherwise, copy the file from you PC to a USB flash drive, and then eject it and plug it into Dexter, then:
```
mount /dev/sda1 /mnt/usbstick		(if mount doesn't work lsblk and look for usb name)
cd /srv/samba/share
cp /mnt/usbstick/DexRun.c .		(hit y to overwrite if it already exists)
````

In either case, you need to <br>
**Update the file date:**
````
date -s "5mar18 21:30"			(change to current date)
````

**Compile** And now you can compile DexRun.c
````
./pg 2>&1 | grep "error"		(if ./pg has an error I screwed up and gave you uncomplible code)
````

**Kill the currently running program** <sup><a href="https://stackoverflow.com/questions/160924/how-can-i-kill-a-process-by-name-instead-of-pid">1</a></sup>
````
pkill DexRun
````

You can check to see if DexRun is active with:
````
pgrep DexRun
````
if it returns a number, that's the program ID of DexRun. If no number is returned, then DexRun is not running.

**Run the new program**
<br>You can run it from the command line, or just restart Dexter to run it via the rc.local file.
```
./DexRun 1 3 1
```
DefaultMode: The first digit controls default settings. A value of 1 loads the default speeds, PID_Ps, AdcCenters.txt, caltables form HiMem.dta, etc... 0 leaves those settings in an unknown state.

ServerMode: The second digit controls where DexRun looks for commands. 1 = a socket connection on part 50000 expecting raw joint position data, 2 = the command line expecting oplets, 3 = a socket connection on port 50000 expecting commands from DDE with job, seq, start and ends times, an oplet, and a terminating ';'. 

RunMode: The third digit enables a real time monitor. 1 or 2 will start the monitor. The monitor gets position data from the Joint 6 and 7 servos and enables force calculations. Without it, the actual positions of Joint 6 and 7 positions will not be sensed, although they can still be moved. 
        
**Debugging notes**
<br>1. printf's used for debugging don't show up until a `\n` is sent. E.g. `printf("hello");` shows nothing at all. `printf(" world\n");` then shows "hello world"
<br>2. printfs slow down network communications when DexRun is not running from the shell. e.g. When it is run on startup from rc.local, any printf's will cause a delay to replies while the system times out waiting for the message to print to nothing. Use sparingly and avoid in areas where speed is critical.


**Back in DDE:**

Format of MOVETO:
<br>`make_ins("M", x(microns), y(microns), z(microns), dir_x, dir_y, dir_z, config_right_left, elbow_up_down, wrist_in_out)`
<br>Example:
<br>`make_ins("M", 0, 0.5/_um, 0.075/_um, 0, 0, -1, 1, 1, 1)`

Format of MOVETOSTRAIGHT:
<br>`make_ins("T", cartesian_speed(micron/sec), x1(microns), y1(microns), z1(microns), dir_x1, dir_y1, dir_z1, config_right_left1, elbow_up_down1, wrist_in_out1, x2, y2, z2, dir_x2, dir_y2, dir_z2, config_right_left2, elbow_up_down2, wrist_in_out2)`
<br>Example:
<br>`make_ins("T", 50000, .1 /_um, 0.5 /_um, 0.075 /_um, 0, 0, -1, 1, 1, 1, -.1/_um, 0.5/_um, 0.075/_um, 0, 0, -1, 1, 1, 1)`

