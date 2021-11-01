# Setup the OS #
## Format SD cards with the correct OS ##
I used the Raspberry Pi imager to download and transfer the required OS to the SD cards. You can find a copy of it at https://www.raspberrypi.com/news/raspberry-pi-imager-imaging-utility/.
Initially, I used raspbian OS 32bit, but because I wanted to go for a full k8s environment (and not k3s), I switched to the Ubuntu arm 64bit version

* Choose OS
* Other general purposes OS
* Ubuntu
* Ubuntu Server 20.04.3 LTS 64-bit -or- Ubuntu Core 20 64-bit

## Edit files on the SD card ##
To prevent from having to couple a screen to my RPI4 devices, I opted for the headless approach. Ubuntu is using cloud-init based configuration files. The cloud-init documentation has more details https://cloudinit.readthedocs.io/.

Below you'll find the changes I made on my SD cards:

### Configure IP address ###
On the Raspberry Pi 4, I have both an Ethernet connection and a WiFi connection. I've connected all Ethernet cards to a 5-port switch to have a network backbone for my cluster. My backbone has a 192.168.99.0/24 network. The WiFi connections are used as an "external" facing network.
***note:** I had issues configuring the WiFi connection in headless mode. Had to fix it later after boot*

Edit the file `network-config`:

```
version: 2
ethernets:
eth0:
  addresses: [192.168.99.10/24]
wifis:
wlan0:
  dhcp4: true
  access-points:
	"<WiFi network name>":
	  password: "<Your password>"

```
### Allow ssh server ###
When I wrote the Ubuntu image to my SD cards, the ssh server was already activated by default. Check below settings in the `user-data` file:

```
...
# Enable password authentication with the SSH daemon
ssh_pwauth: true
...
```

This file `user-data`also contains other useful information and parametrisation:

1. Change password of "ubuntu" user at first login (default settings)

```
...
# On first boot, set the (default) ubuntu user's password to "ubuntu" and
   # expire user passwords
   chpasswd:
     expire: true
     list:
     - ubuntu:ubuntu
...
```

2. Install additional packages on first boot (could be used to install std packages needed for setting up k8s)
3. Change keyboard layout (only needed when directly connecting to the rpi4, not through SSH)

## Web references ##

Guide was based upon https://roboticsbackend.com/install-ubuntu-on-raspberry-pi-without-monitor/

Headless setup with redirecting output to console https://limesdr.ru/en/2020/10/17/rpi4-headless-ubuntu/

Step-by-step at Ubuntu, but needing a monitor https://ubuntu.com/tutorials/how-to-install-ubuntu-core-on-raspberry-pi
