# OctoPrint for Multiple Printers: How to Get It Working

As of 2022, Raspberri PI units are almost impossible to find at a reasonable cost, if at all. If you have multiple 3d printers to control, the 1-1 model of server per printer that relied on rpi, needs some revisitng.
We need to run multiple octoprint server instances, docker is a very reasonable option, but there are a few subtleties that need addressed.

Say we wish to control 3d printers, called "mini1" and "mini2" amd "mk3s".

create 3 docker volumes:

```
docker volume create octoprint_mini1
docker volume create octoprint_mini2
docker volume create octoprint_mk3s1
```

Start the containers:

```
docker run -p 5100:80 -v octoprint_mini1:/octoprint --device /dev/ttyMINI1:/dev/ttyACM0  --name mini1 -dit --restart unless-stopped    octoprint/octoprint

docker run -p 5200:80 -v octoprint_mini2:/octoprint --device /dev/ttyMINI2:/dev/ttyACM0   --name mini2 -dit --restart unless-stopped  octoprint/octoprint

docker run -p 5300:80 -v octoprint_mk3s1:/octoprint --device /dev/ttyMK3S1:/dev/ttyACM0   --name mk3s -dit --restart unless-stopped  octoprint/octoprint
```

Everything is self explanatory.  The only "twist" is this part:

```
 --device /dev/ttyMINI1:/dev/ttyACM0
```

The container needs to have device mapped for it to be avaible internally. 
Allocation of ACM0/ACM1/ACM2/... is basically a musical chair game between available ports and connection order.
(This is not a big deal when you have a single host connected to one printer, that is alsmot guaranteed to be ACM0)



## unqieuly identify the decice you wish to control
Run the follwoing command for each of the ACM0/.../ACMn devices 
```
udevadm info -a -p $(udevadm info -q path -n /dev/ttyACM0) 
```

Scroll down a bit and you should find the block that for the device. 
```
  ...
  looking at parent device '/devices/pci0000:00/0000:00:14.0/usb1/1-5/1-5.1':
    KERNELS=="1-5.1"
    SUBSYSTEMS=="usb"
    DRIVERS=="usb"
    ATTRS{idVendor}=="2c99"
    ATTRS{serial}=="CZPX0522X017XC11111"
    ATTRS{idProduct}=="000c"
    ATTRS{manufacturer}=="Prusa Research (prusa3d.com)"
    ATTRS{product}=="Original Prusa MINI"
    ...
```

ATTRS{idVendor}=="2c99" is the vendor id for Prusa Research
ATTRS{serial}=="CZPX0522X017XC11111"  uniquely identify the device.

## create approritate udev rules

Create a customer rule file in /etc/udev/rules.d/

```
vi /etc/udev/rules.d/3d.rules 
```

example:

``` 
KERNEL=="ttyACM[0-9]*", SUBSYSTEM=="tty", ATTRS{idVendor}=="2c99", ATTRS{serial}=="CZPX0522X017XC11111", SYMLINK="ttyMINI1", RUN="/usr/bin/docker restart mini1"

KERNEL=="ttyACM[0-9]*", SUBSYSTEM=="tty", ATTRS{idVendor}=="2c99", ATTRS{serial}=="CZPX1022X017XC22222", SYMLINK="ttyMINI2", RUN="/usr/bin/docker restart mini2"

KERNEL=="ttyACM[0-9]*", SUBSYSTEM=="tty", ATTRS{idVendor}=="2c99", ATTRS{serial}=="CZPX3021X004XC33333", SYMLINK="ttyMK3S1", RUN="/usr/bin/docker restart mk3s1"
```

explanation

```
KERNEL=="ttyACM[0-9]*", SUBSYSTEM=="tty", ATTRS{idVendor}=="2c99", ATTRS{serial}=="CZPX0522X017XC11111"
```

identfies the device


```
SYMLINK="ttyMINI1"
```

Assignes a unique name, but is a symlink. When a docker is lanuched with a symlink as a mapped device, the symlink is derefenced, which means if the device is disconnected, reconnected and gets a different name, the underlying ACM device will still be in use in the container.

So to avoid that confusion:
'''
RUN="/usr/bin/docker restart mini1"
'''

This part restarts the container running octoprint, so it would always use the correct device.
This happens in most cases less than once a day, so the restart overhead is negliable, but this could be more efficient with clever scripting that can keep track if the mapping actually changed. I am satisified at this point.



save the file and either reboot or (better) just reload the rules:
```
   sudo udevadm control --reload-rules
```


You can view the assignment of symlinks to devices, and just for fun, disconnect them all and reconnect them in a different order and see how the linking changes - that is why the dockers need to be restarted.
```
ls -l /dev/ | grep ACM
crw-rw----   1 root dialout 166,   0 Nov 15 13:39 ttyACM0
crw-rw----   1 root dialout 166,   1 Nov 15 13:44 ttyACM1
crw-rw----   1 root dialout 166,   2 Nov 15 13:39 ttyACM2
lrwxrwxrwx   1 root root           7 Nov 15 13:39 ttyMINI1 -> ttyACM0
lrwxrwxrwx   1 root root           7 Nov 15 13:39 ttyMINI2 -> ttyACM2
lrwxrwxrwx   1 root root           7 Nov 15 13:44 ttyMK3S1 -> ttyACM1
```

While I do not use webcams (yet) the same principle applies here as well, and you map a webcam device to the container exactly in the same manner. I might get a webcam and update this tutorial in the upcoming weeks. In the meantime I leave this as an excersie for the reader. I would leave the camera part as an excercise to the reader.


Last but not least
Now we have 3 servers running on the following addresses
http://10.0.2.22:5100
http://10.0.2.22:5200
http://10.0.2.22:5300

Let's say you own a domain called foo.com, would it be nice to map each printer to http://mini1.foo.com, http://mini2.foo.com, http://mk3s1.foo.com ?

nginx to the rescue. 
I run nginx like so:

```
docker run  -v /home/erez/nginx/nginx.conf:/etc/nginx/nginx.conf  --net host  -dit --restart unless-stopped nginx
```

Notice the --net host argument, there is probably a better way to that, probably with docker-compose, but I wanted nginx to connect to the containers running on the host "directly" so I gave it the host network stack. nginx file is included in this project.
Not exactly nginx poetry but it does work.
