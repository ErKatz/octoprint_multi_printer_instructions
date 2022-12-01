# OctoPrint for Multiple Printers: How to Get It Working

As of 2022, Raspberri PI units are almost impossible to find at a reasonable cost, if at all. 
If you have multiple 3d printers to control, the 1-1 model of server per printer that relied on rpi needs a second thought.

To run multiple octoprint server instances, docker is a very reasonable option, but there are a few subtleties that must be addressed, otherwise things might work fine for a while... then get nuts when you reboot the system or disconenct and reconnect devices at an arbitrary order.

For this, lets say we wish to control 3d printers, called "mini1" and "mini2" amd "mk3s".

### Run Octoprint in docker containers

create 3 docker volumes (first time only):

```
docker volume create octoprint_mini1
docker volume create octoprint_mini2
docker volume create octoprint_mk3s1
```

Start the containers:

```
docker run -p 5100:80 -v octoprint_mini1:/octoprint --device /dev/ttyMINI1:/dev/ttyACM0  --name mini1 -dit --restart unless-stopped octoprint/octoprint

docker run -p 5200:80 -v octoprint_mini2:/octoprint --device /dev/ttyMINI2:/dev/ttyACM0  --name mini2 -dit --restart unless-stopped octoprint/octoprint

docker run -p 5300:80 -v octoprint_mk3s1:/octoprint --device /dev/ttyMK3S1:/dev/ttyACM0  --name mk3s1 -dit --restart unless-stopped  octoprint/octoprint
```


Everything is self explanatory and was taken from the documentation on docker hub.
The only deviation from the documentaiton is in device mapping:

```
 --device /dev/ttyMINI1:/dev/ttyACM0
```

The container needs to have device mapped for it to be avaible internally. 
The way linux mangages usb devices, they get allcoated their nubmer based on availability and connection order. 
That is basically a recipie for a muscial chair game, which is ok when you have one chair and one player (i.e. one rpi per printer) but not really when you try to manage multiple devices.

If the servers are running and you disconnect the printers and reconnect them in a different order, the servers would use the original device number.
This could be very problematic when each printer is different or has different filament with its specific settings.
**The device mapped to a container must be unique!**


## How to unqieuly identify the decice you wish to control
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

Take note of the attributes you wish to use in order to identify the device when it is connected.


## Create approritate udev rules

Create a custom rule file in /etc/udev/rules.d/

```
vi /etc/udev/rules.d/3d.rules 
```

example:

``` 
KERNEL=="ttyACM[0-9]*", SUBSYSTEM=="tty", ATTRS{idVendor}=="2c99", ATTRS{serial}=="CZPX0522X017XC11111", SYMLINK="ttyMINI1", RUN="/usr/bin/docker restart mini1"

KERNEL=="ttyACM[0-9]*", SUBSYSTEM=="tty", ATTRS{idVendor}=="2c99", ATTRS{serial}=="CZPX1022X017XC22222", SYMLINK="ttyMINI2", RUN="/usr/bin/docker restart mini2"

KERNEL=="ttyACM[0-9]*", SUBSYSTEM=="tty", ATTRS{idVendor}=="2c99", ATTRS{serial}=="CZPX3021X004XC33333", SYMLINK="ttyMK3S1", RUN="/usr/bin/docker restart mk3s1"
```

**explanation:** 

The first part identifies the device:

```
KERNEL=="ttyACM[0-9]*", SUBSYSTEM=="tty", ATTRS{idVendor}=="2c99", ATTRS{serial}=="CZPX0522X017XC11111"
```

This tells the udev system to create a symlynk with that name pointing to the devie

```
SYMLINK="ttyMINI1"
```

However, when a docker contianer is started with a symlink as a mapped device, **the symlink is derefenced**, which means if the device is disconnected, reconnected and gets a different name, the previous underlying ACM device will still be used by the container (same musical chair game goes on!).

So to avoid that confusion, the container is restarted "clean". 
The overhead is minimal and in most applications devices get connected/disconnected at most once a day (if at all)

```
RUN="/usr/bin/docker restart mini1"
```

save the file and either reboot or just reload the rules:

```
sudo udevadm control --reload-rules
```

The rules will apply for newly connected devices - so if you didn't reboot - disconnect and reconenct the devices.

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


## Webcams

Here is an example udev rule that creates a unique symlink for each camera, similar to the ACM rule above:

```
KERNEL=="video[0-9]*",  SUBSYSTEM=="video4linux",ATTR{index}=="0", ATTRS{idVendor}=="046d", ATTRS{serial}=="2D540000", SYMLINK="videoMINI2", RUN="/usr/bin/docker restart video_mini2"
```

In this case it is a Logitech camera (ATTRS{idVendor}=="046d")
It is crucial to specify ATTR{index}=="0", because for each camera, 2 /dev/video devices are created, one is the actual device and the other is a metadata provider. 

With that in mind run the follwoing command (simillar to above) for each of the video0/.../videon devices to get the correct idVendor and serial number values.

```
udevadm info -a -p $(udevadm info -q path -n /dev/video0) 
```

Properly configured, you should see the mapping as follows:

```
ll /dev/vid*
crw-rw----+ 1 root video 81, 0 Nov 25 19:43 /dev/video0
crw-rw----+ 1 root video 81, 1 Nov 25 19:43 /dev/video1
lrwxrwxrwx  1 root root      6 Nov 25 19:43 /dev/videoMINI2 -> video0
```

It is better to run a separate container for webcam functionality, because if we connect/reconnect the camera in the middle of a long print, we don't want to restart the octoprint instance that is managing that print. 

```
docker run -p 5250:80 --device /dev/videoMINI2:/dev/video0 -e ENABLE_MJPG_STREAMER=true --name video_mini2 -dit --restart unless-stopped  octoprint/octoprint
```

if you are getting a "no space left on device" error, it means the cameras try to grab more (USB) bandwidth than available, which is 480Mb total per controller for USB2.0 devices (even if it is USB3.0 controller and a USB3.0 hub, USB3.0 controllers are dual USB3/USB2, and the USB2.0 devcices don't the to use the 5Gb bandwidth that is only available to USB3.0 devices).

If you can move one camera to a different controller - great. If you don't have enough controllers, then 2 things need to take place.
1) follow the instructions here: https://stackoverflow.com/questions/11394712/libv4l2-error-turning-on-stream-no-space-left-on-device/26523421#26523421
2) change the docker launch command so it uses less bandwidth by adding -e MJPG_STREAMER_INPUT='-y -r 640x480 -f 15' to the docker run command:
```
docker run --name video_mini2 \
           -p 5250:80 \
           --device /dev/videoMINI2:/dev/video0 \
           -e ENABLE_MJPG_STREAMER=true  \
           -e MJPG_STREAMER_INPUT='-y -r 640x480 -f 15' \
           -dit --restart unless-stopped octoprint/octoprint
```
it is important to keep the -y flag that forces YUYV format, because the modprob quirk setting does not apply to compressed streams. 

## Last but not least
Now we have 3 servers running on the following addresses
http://10.0.2.22:5100
http://10.0.2.22:5200
http://10.0.2.22:5300

Let's say you own a domain called foo.com, would it be nice to map each printer to http://mini1.foo.com, http://mini2.foo.com, http://mk3s1.foo.com ?

nginx to the rescue. 
example for running nginx :

```
docker run  -v /home/erez/nginx/nginx.conf:/etc/nginx/nginx.conf  --net host  -dit --restart unless-stopped nginx
```

Notice the --net host argument, there is probably a better way to that (docker-compose).
A sample nginx.conf file is included in this repo.

