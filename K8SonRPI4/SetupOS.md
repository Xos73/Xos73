# Setup the OS
## Before first boot
### Format SD cards with the correct OS
I used the Raspberry Pi imager to download and transfer the required OS to the SD cards. You can find a copy of it at https://www.raspberrypi.com/news/raspberry-pi-imager-imaging-utility/.
Initially, I used raspbian OS 32bit, but because I wanted to go for a full k8s environment (and not k3s), I switched to the Ubuntu arm 64bit version

* Choose OS
* Other general purposes OS
* Ubuntu
* Ubuntu Server 20.04.3 LTS 64-bit

### Edit files on the SD card
To prevent from having to couple a screen to my RPI4 devices, I opted for the headless approach. Ubuntu is using cloud-init based configuration files. The cloud-init documentation has more details https://cloudinit.readthedocs.io/.

Below you'll find the changes I made on my SD cards:

> **Note:** If you want to do the same with Raspbian OS, you'll need to:
>
> * `touch ssh` A file names "ssh" should be present
> * add a `wpa_supplicant.conf` file in the root, containing your needed WiFi configuration


#### Configure IP address
On the Raspberry Pi 4, I have both an Ethernet connection and a WiFi connection. I've connected all Ethernet cards to a 5-port switch to have a network backbone for my cluster. My backbone has a 192.168.99.0/24 network. The WiFi connections are used as an "external" facing network.
***Note:** I had issues configuring the WiFi connection in headless mode. Had to fix it later after boot*

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
      "<wiFiNetworkName>":
        password: "<yourPassword>"

```
#### Allow ssh server
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
### Web references
Guide was based upon https://roboticsbackend.com/install-ubuntu-on-raspberry-pi-without-monitor/

Headless setup with redirecting output to console https://limesdr.ru/en/2020/10/17/rpi4-headless-ubuntu/

Step-by-step at Ubuntu, but needing a monitor https://ubuntu.com/tutorials/how-to-install-ubuntu-core-on-raspberry-pi

## After reboot
### Switch Ethernet config from cloud-init to netplan

When booting from the SD card, cloud-init is used. Ubuntu can still use cloud-init, nevertheless they moved their config to netplan

#### Create netplan config

Add the following to `/etc/netplan/00-installer-config.yaml`:

```
network:
  version: 2
  ethernets:
  eth0:
    addresses: [192.168.99.10/24]
  wifis:
  wlan0:
    dhcp4: true
    access-points:
      "<wiFiNetworkName>":
        password: "<yourPassword>"

```
#### Disable cloud-init

```bash
cat <<EOF | sudo tee /etc/cloud/cloud.cfg.d/99-disable-network-config.cfg
network: {config: disabled}
EOF
```

### Enable cgroup settings

```bash
sudo sed -i -e '$s/$/\ cgroup_enable=memory swapaccount=1 cgroup_memory=1 cgroup_enable=cpuset/' /boot/firmware/cmdline.txt
```

### Configure the needed kernel modules

```bash
modprobe overlay
modprobe br_netfilter

cat <<EOF | sudo tee /etc/modules-load.d/crio.conf
overlay
br_netfilter
EOF

cat <<EOF | sudo tee /etc/sysctl.d/99-kubernetes-cri.conf
net.bridge.bridge-nf-call-iptables  = 1
net.ipv4.ip_forward                 = 1
net.bridge.bridge-nf-call-ip6tables = 1
EOF

sudo sysctl --system
sudo swapoff -a
```

### Perosnalisation
#### Rename your server
```bash
sudo hostnamectl set-hostname <newHostname>
```

#### Change to vi as default editor
```bash
sudo update-alternatives --set editor /usr/bin/vim.basic
```

#### Add your own user to the system and create your ssh key
```bash
sudo adduser <yourUser>
sudo usermod -aG sudo <yourUser>
```

#### Don't forget to adapt sudo rights
```bash
sudo visudo
```
or
```bash
sudo visudo -f /etc/sudoers.d/99-<yourUser>

# User rules for <yourUser>
<yourUser> ALL=(ALL) NOPASSWD:ALL

```

### Update the system and reboot the system

```bash
sudo apt-get update
sudo apt-get upgrade -y
sudo apt-get dist-upgrade -y
sudo apt-get install apt-transport-https
sudo reboot
```
