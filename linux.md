% Linux 

# Hardware

## Know your hardware

 Before you struggle opening the box or finding the adequate screwdriver, you can try out the following:


List hardware: `lshw` or `inxi -Fxz`
For example:

- Network: `lshw -C network`
- Hard disks: `lshw -C disk`

Product model:
```
$ inxi -M
Machine:   System: xxx product: xxx v: 01
           Mobo: xxx v: x Bios: xx v: xx date: xx/xx/xxxx
```
    or
```
sudo dmidecode -t baseboard | grep -i 'Product'
```


List RAM: `sudo dmidecode -t memory`

List PCI devices: `lspci`.
For example, get the video card:
```
$ lspci -vnn | grep VGA -A 12
```

- list USB devices: lsusb
- list block devices: lsblk
- list SCSI devices (useful for SATA disks): `cat /proc/scsi/scsi` or `lsscsi`

## Get hard disk info

`sudo hdparm -i /dev/sda`

## Monitor luminosity

Get name of device:

```
$ xrandr -q | grep ' connected' | head -n 1 | cut -d ' ' -f1
eDP-1
```

Then set luminosity: `xrandr --output eDP-1 --brightness 0.7`


## Kernel

To list unused kernels:

```
kernelver=$(uname -r | sed -r 's/-[a-z]+//')
dpkg -l linux-{image,headers}-"[0-9]*" | awk '/ii/{print $2}' | grep -ve $kernelver
```

And then, uninstall these images with `sudo apt-get purge linux-image-xxxx`


## Setting keyboard layout

`setxkbmap -layout fr`

## Using accents with a QWERTY layout

Put this in `.profile`: `xmodmap ~/.xmodmaprc` where your `.xmodmaprc` defines your keyboard tricks:

```
! letters

keycode  24 = q Q acircumflex
keycode  25 = w W ecircumflex
keycode  26 = e E eacute
keycode  27 = r R egrave
keycode  30 = u U ucircumflex
keycode  31 = i I icircumflex
keycode  32 = o O ocircumflex

keycode  38 = a A agrave
keycode  40 = d D ediaeresis
keycode  43 = h H ugrave
keycode  44 = j J udiaeresis
keycode  45 = k K idiaeresis
keycode  46 = l L odiaeresis

keycode  52 = z Z adiaeresis
keycode  54 = c C ccedilla


keycode  108 = Mode_switch
```

Alternatively, it is possible to use the "English US International with dead letters" keyboard and then use composition: ` + e gives è. See [here](https://www.ellendhel.net/article.php?ref=2011+09+12-0).



# System

## Services

[Create a service with Systemd](https://doc.ubuntu-fr.org/creer_un_service_avec_systemd)

Example for Junior CTF using CTFd platform:

```
# cat ctfd.service 
[Unit]
Description=Capture The Flag server (CTFd)
After=network-online.target

[Service]
Type=simple
User=root
Group=root
ExecStart=/usr/bin/python /usr/share/CTFd/serve.py &
Restart=on-failure
TimeoutStopSec=300

[Install]
WantedBy=multi-user.target


$ cat ctfd-docker.service 
[Unit]
Description=Docker containers for challenges of Junior CTF
After=ctfd.service

[Service]
Type=oneshot
User=axelle
Group=axelle
RemainAfterExit=yes
ExecStart=/bin/bash /home/axelle/juniorctf/up.sh
ExecStop=/bin/bash /home/axelle/juniorctf/down.sh

[Install]
WantedBy=multi-user.target
```

If you want a service that runs periodically, then use a **timer** (which is a special service) (or use crontab). [This explains how to create a timer](https://www.linuxtricks.fr/wiki/systemd-creer-des-services-timers-unites) (in French).

Basically,

1. Create a service file. The service file dictates the executable to run and in which conditions.
2. Create a timer file (extension .timer) to explain how frequently to run the service.
3. Install the service and timer files to `/lib/systemd/system`
4. Reload: `sudo systemctl daemon-reload`
5. Enable and then activate the timer: `sudo systemctl enable/start xxx.timer`

Example: the service file:

```
[Unit]
Description=Allows kid to log only at certain times of the day
After=graphical.target

[Service]
Type=oneshot
ExecStart=/home/xx/scripts/parentalcontrol.sh

[Install]
WantedBy=multi-user.target
```

The timer file:

```
[Unit]
Description=Allows kid to log only at certain times of the day


[Timer]
OnUnitActiveSec=5min

[Install]
WantedBy=multi-user.target
```

### Commands for Services

- Listing the service config file: `systemctl show SERVICENAME`
- Editing a unit configuration file: `sudo systemctl edit --full SERVICENAME`, then do `sudo systemctl daemon-reload` and finally `sudo systemctl restart SERVICENAME` (see [here](https://www.2daygeek.com/linux-modifying-existing-systemd-unit-file/))
- List failed services: `sudo systemctl list-units --failed`

### Journal for Services

- Dump to a file: `journalctl -x -u service > file`
- Wrap long lines: `journalctl -u service | less` or `journalctl -u service --no-pager`
- After bug `journalctl -xb`


## Network

### Interfaces

```bash 
$ sudo ifconfig <interface> <address> netmask <mask>
```

or with the `ip` syntax use commands such as:

- `ip link set eth0 down`
- `ip address add 192.168.0.77 dev eth0`


### Routes

```bash
ip route show
ip route add 10.20.0.0/24 dev rndis0
ip route add default via 10.20.30.1
```

### Name resolution

In recent Linux Mint / Ubuntu distribution, you no longer directly edit /etc/resolv.conf to specify your name server as the file's header says:

```bash
# Dynamic resolv.conf(5) file for glibc resolver(3) generated by resolvconf(8)
#     DO NOT EDIT THIS FILE BY HAND -- YOUR CHANGES WILL BE OVERWRITTEN
```

Instead, you specify name servers in /etc/resolvconf/resolv.conf.d/base
```bash
$ cat /etc/resolvconf/resolv.conf.d/base
nameserver 4.2.2.2
nameserver 8.8.8.8
nameserver 8.8.4.4
```

If there are some name servers you want to favour, then, you need to put them
in `/etc/resolvconf/resolv.conf.d/head`.

Once this is done, you need to update name resolution with the command `resolvconf -u`.

If you want to add a [DNS server temporarily](https://notes.enovision.net/linux/changing-dns-with-resolve):

1. Add it in `/etc/systemd/resolved.conf`
2. Restart service: `service systemd-resolved restart`
3. Check: `systemd-resolve --status`




### Troubleshooting

I had issues with my Ethernet link. In my case, it was solved by commenting out dns=dnsmasq in NetworkManager:

```
[main]
plugins=ifupdown,keyfile,ofono
#dns=dnsmasq

[ifupdown]
managed=false
```

### Wireless

Get your SSID: `iwgetid`

### IPv6

In /etc/sysctl.conf

```
# Disable IPv6
net.ipv6.conf.all.disable_ipv6 = 1
```

### mDNS

*"Avahi is a Linux implementation of Zero Configuration Networking ( Zeroconf ) which implements muticast DNS ( mDNS ) allowing ip address to hostname resolution without the use of standard LAN side DNS services."*

To install it: `sudo apt-get install libnss-mdns`. And ensure that `/etc/nsswitch.conf` has mDNS mentioned:

```
hosts:          files mdns4_minimal [NOTFOUND=return] dns mdns4
```


- avahi requires that port 5353/udp is open.
- To find all IPv4 services on the internal network: ` avahi-browse -at | grep IPv4`
- You can ping `host.local` on the intranet.

### CIFS / AFP

To access a remote Apple Timecapsule:

`sudo mount.cifs //ip/share /mnt/point -o username=theusername,password=thepassword,vers=1.0,sec=ntlm,uid=youruser`

- ip is the IP address of the time capsule
- `share` is the name of the sharepoint on the timecapsule
- `username` and `password` specify the credentials to log on the time capsule
- **do not forget** `vers=1.0` **and** `sec=ntlm`
- uid to specify the Linux user id to give access to


## Locale

```
export LANGUAGE=en_US.UTF-8
export LANG=en_US.UTF-8
export LC_ALL=en_US.UTF-8
locale-gen en_US.UTF-8
sudo dpkg-reconfigure locales
```

`localectl set-locale LANG=en_US.utf8`

## Package management

To re-install a package:
```bash
$ sudo apt-get --reinstall install package
```

## NTP

To list NTP servers I use:

```
$ ntpq -p
     remote           refid      st t when poll reach   delay   offset  jitter
==============================================================================
*ks3352891.kimsu 138.96.64.10     2 u   99  128  377   37.876   31.229  24.093
...
```

## SSH

```
ssh-keygen -t rsa -b 4096
ssh-keyscan -H 192.168.0.9 >> known_hosts
```

## ZFS

### Install ZFS

On Linux Mint 19:

```bash
sudo apt-get install zfs-dkms zfsutils-linux
```

Then, to import a pool: `zpool import POOLNAME`

## Firewall

```bash
$ iptables -t nat -F ==> flush the NAT table
```

Redirect an IP address to yourself (or another IP address):

```bash
sudo iptables -t nat -A OUTPUT -p all -d SOURCE-IP -j DNAT --to-destination DEST-IP
```

To remove a rule,

1. List rule number: `sudo iptables -t nat -v -L OUTPUT -n --line-number`
2. Remove the given number: `sudo iptables -t nat -D OUTPUT NUM`

Here OUTPUT refers to the part of iptables to work on


```
sudo iptables -t nat -v -L PREROUTING -n --line-number
sudo iptables -t nat --delete PREROUTING 4
```


## LVM

We have:

- physical volumes e.g. disks or partitions of disks
- volume groups: they have a name and you can add physical volumes to them.
- logical volumes:  set its size, its name, where to mount it to and the volume group

Creating a physical volume:

- List the disk you want to use with `lsblk`
- Create a partition (e.g. `fdisk`). Be sure to set partition type `8e` for Linux LVM.
- Create the volume: `pvcreate /dev/sdc1`

Creating the volume group:

- `vgcreate vgelk /dev/sdc1`
- If you want to add future partitions, do `vgextend vgelk /dev/sdc2`

Creating a logical volume:

- `lvcreate -n var -L 150G vgelk`
- Format the logical volume: `mkfs -t ext4 /dev/vgelk/var`
- Copy the contents of `/var` (or other) to that logical volume:

To copy/backup completely a directory (make sure links are ok):
```
mkdir /mnt/var_new
mount /dev/vgelk/var /mnt/var_new
rsync -avHPSAX /var/ /mnt/var_new/
```
References:

- https://www.computernetworkingnotes.com/rhce-study-guide/learn-how-to-configure-lvm-in-linux-step-by-step.html
- http://www.lerrigatto.com/move-var-to-a-new-partition-with-lvm/

## User management 

### Adding user to group and take into account immediately

1. Add user to new group B.
2. Get the current group of the user:
```bash
$ id -g
```
Let's call that group A.
3. Change group to B:
```bash
$ newgrp B
```
4. Reset back to original group:
```bash
$ newgrp A
```

## Adding udev rules without rebooting

```bash
sudo udevadm control --reload-rules
sudo service udev restart
sudo udevadm trigger
```

## Cinnamon


- Configure sound levels: `cinnamon-settings sound`
- Lock screen: `cinnamon-screensaver-command --lock`


## Mate

Specify keyboard bindings in `mate-control-center`

## MDM

To have the correct keyboard in MDM, at the end of `/etc/mdm/Init/Default`, insert:

```
/usr/bin/setxkbmap fr
```


# Sound

## Listing devices

```bash
$ sudo aplay -l
**** List of PLAYBACK Hardware Devices ****
card 0: PCH [HDA Intel PCH], device 0: ALC269VB Analog [ALC269VB Analog]
  Subdevices: 0/1
  Subdevice #0: subdevice #0
card 1: J20 [Jabra EVOLVE 20], device 0: USB Audio [USB Audio]
  Subdevices: 1/1
  Subdevice #0: subdevice #0
```


# Apps

## Mail

d * removes all email
q

## Recording desktop

```
$ gtk-recordmydesktop
```

To stop: Ctrl-Mod-s


## Bluez

```bash 
$ wget http://www.kernel.org/pub/linux/bluetooth/bluez-5.20.tar.xz
$ tar -xvf bluez-5.20.tar.xz
$ cd bluez-5.20/
$ sudo apt-get install libudev-dev libical-dev libreadline-dev
$ ./configure –enable-library –disable-systemd
$ make
$ make check
$ sudo make install
$ sudo cp attrib/gatttool /usr/bin/
$sudo cp tools/btmgmt /usr/bin/
```

## Piwigo

[Piwigo gallery on nginx with debian](https://www.howtoforge.com/install-piwigo-gallery-on-nginx-with-debian-wheezy)

```
create database gallery01; grant all on gallery01.* to 'gallery'@'localhost' identified by 'PASSWORD'; flush privileges; \q;
```


## Useful packages (at some point...)

- To install glib2:
```
sudo apt-get install libgtk2.0-dev
```


- To install [Java](http://tecadmin.net/install-oracle-java-8-jdk-8-ubuntu-via-ppa/):

`export JAVA_HOME=/usr/lib/jvm/java-8-oracle`
