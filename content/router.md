+++
title = 'DiY Router'
date = 2024-09-29T16:06:20-07:00
draft = false
+++

## Do-it-Yourself, Open Source ARM Gateway/Router

Many U.S. ISPs charge their customers $10-15 per month for a gateway/router device that most people just think of as "the wifi". In my experience, it's usually not made clear that it is optional to rent and use this device. That is, you typically have to return it if you terminate service or pay for it if you fail to do so. You'll find these terms if you are willing to wade through the fine print, of course.

The terms are deliberately buried because loaning someone a $100 device and charging them $10 a month is a *great* business. ISPs probably see a greater return from this rental business than they do investing in the infrastructure to provide you fast and reliable internet. Which would explain a lot.

But an even better business to be in the 2020s is selling your internet activity to advertisers. In the case of my ISP (Starlink) their device sends the address of every website I visit or app I use directly to Google. If you are a Starlink customer and use their hardware to provide WiFi for your home, Google knows about *every* request for an internet resource from all of the devices in your home.

That shouldn't come as a big surprise given [how Google operates](https://www.theverge.com/2023/10/27/23934961/google-antitrust-trial-defaults-search-deal-26-3-billion). But it is something I like to bring up whenever I find myself in a conversation with someone who swears their phone is listening to their conversations. "No no," I reassure them, "processing audio is costly. Why would they bother with that when they already know about every interaction all of your personal devices have with the internet?"

The good news is that you can build your own router and configure it so that it does not send the address of every website you visit to your ISP (or whichever advertising company they are in bed with).

For additional motivation/inspiration, I recommend reading [this entertaining and extensive guide](https://wiki.futo.org/wiki/Introduction_to_a_Self_Managed_Life:_a_13_hour_%26_28_minute_presentation_by_FUTO_software#Why_Build_Your_Own_Router?). It contains a more accessible guide to building your own router than what you will find here--which is mostly notes to remind myself what I did.

What follows is an outline of steps to get up-to-date, mainstream U-Boot and Linux running on an ARM SoC network appliance (e.g. a hobbyist-level device like a [Raspberry Pi](https://www.raspberrypi.com/)). After many years of using [DD-WRT](https://dd-wrt.com/) as a router OS on a [Netgear Nighthawk R7000](https://www.netgear.com/home/wifi/routers/r7000/), I was very tired of the process of having to "flash" my router in order to update the kernel/OS.

DD-WRT's kernel for the device when I stopped using it in 2024 was version 4.19 which stopped receiving support from Linux in December 2024. Security vulnerabilities are discovered continuously in software. It follows that the ability to run a maintained kernel (and to keep it up-to-date with minimal fuss!) in the device separating your network from the rest of the internet are desirable properties.

### Choosing hardware and a Linux distribution

In 2023, I decided to get a [Globalscale ESPRESSOBIN Ultra](https://globalscaletechnologies.com/product/espressobin-ultra/) for this project. This line of devices is the result of a [kickstarter project](https://www.kickstarter.com/projects/874883570/marvell-espressobin-board).  The hardware (outside of the WiFi/Bluetooth module) seems fine, but the devices I received came running outdated and unmaintained forks of upstream open source projects (e.g. Linux and U-Boot). Globalscale's forks--lacking years of upstream development--are also very buggy.

It is your basic small-office/home-office router with some nice features like PoE via the WAN port, 4 port switch, and 4 LEDs. It also has a WiFi/Bluetooth module, but it is not very good and requires the use of closed-source drivers.  I recommend separate, dedicated WiFi access point hardware and avoiding this device's WiFi.

I would also recommend x86 (Intel/AMD) hardware and using the guide linked above. While ARM was less common in 2023 and some argue that is a security advantage, documentation and community resources were also less robust. If you mess up the configuration, it really won't matter if you are using an ARM device or not.

In any case, before purchasing any hardware, it is a good idea to spend time in any forums you can find where you might find users of the same device. Look for community resources hosted by the manufacturer, retailer, or one or more Linux distributions. The distributions in particular are typically going to be your starting point for support. It is fine to choose a device/distribution that does not have a huge community as long as you make those choices knowing what kind of support is available.

The level of support for 64-bit ARM (ARMv8) architecture is highly variable across distributions. I have been using Arch Linux on desktop for a few years, and while stability is not *desktop* Arch's strength, I've had fewer issues using it on headless systems. It can also be very lean which is desirable for security.

Unfortunately the project doesn't officially support the ARM architecture. [Arch Linux ARM](https://archlinuxarm.org/) is an unofficial port that I use, but the project doesn't share the community or infrastructure of Arch Linux. Spend time in the forums and documentation of the distributions you are considering and decide for yourself. Debian (and Ubuntu) are worthy of consideration. In any case, the remainder of this guide should work for any distribution that uses systemd as their system manager.

#### UEFI

If you do decide to use an ARM device, I strongly recommend ensuring it supports [UEFI](https://en.wikipedia.org/wiki/UEFI). For many ARM devices, this will mean the firmware bootloader was compiled with a modern version of U-Boot configured to support [UEFI on U-Boot](https://docs.u-boot.org/en/latest/develop/uefi/uefi.html).

UEFI specifies a standard location to find bootable images on [pecial filesystem partitions](https://en.wikipedia.org/wiki/EFI_system_partition).. This makes it easier for Linux distributions to support the device because it won't require as much maintenance of device-specific packages/documentation which will see far less support relative to packages used by all users (e.g. the distro's generic kernel).

The ESPRESSObin Ultra ships with a firmware bootloader that does not support UEFI. I fixed this (and other issues) and maintain firmware bootloader images [here](https://github.com/bschnei/ebu-bootloader?tab=readme-ov-file#espressobin-ultra-bootloader).

### Configure U-Boot

U-Boot is configured via the USB serial console which is available on the
device's Micro-USB port. Connect it to another host with a USB cable/adapter
and use a terminal emulator like [PuTTY](https://www.putty.org/) or
[Screen](https://www.gnu.org/software/screen/) to open the serial console.

On this device, U-Boot stores it's [environment
variables](https://docs.u-boot.org/en/latest/usage/environment.html) in
dedicated storage that does not get erased when flashing the bootloader. It's
recommended to reset the environment variables to their defaults by running:
```
==> env default -a
```
If U-Boot was compiled with support for [Standard Boot](https://docs.u-boot.org/en/latest/develop/bootstd/index.html), the only required configuration is to set the
environment variable `bootcmd` to `bootflow scan`:

```
==> env set bootcmd "bootflow scan"
==> env save
```
This will tell U-boot to cycle through available boot devices and look for an
EFI system partition When it finds one, it will try to load the EFI image at
`EFI/BOOT/BOOTAA64.EFI`.

Full U-Boot environment variables for my device look like this:
```
arch=arm
baudrate=115200
board=mvebu_armada-37xx
board_name=mvebu_armada-37xx
bootcmd=bootflow scan
bootdelay=2
cpu=armv8
eth1addr=00:51:82:11:22:01
ethact=ethernet@30000
ethaddr=F0:AD:XX:XX:XX:XX
ethprime=eth0
extra_params=pci=pcie_bus_safe
fdt_addr_r=0x6f00000
fdt_high=0xffffffffffffffff
fdt_name=-
fdtcontroladdr=3faf7910
fileaddr=6000000
filesize=790
gatewayip=10.4.50.254
hostname=marvell
image_name=-
initrd_addr=0xa00000
initrd_size=0x2000000
ipaddr=0.0.0.0
kernel_addr_r=0x7000000
loadaddr=0x6000000
netdev=eth0
netmask=255.255.255.0
pcb_rev=1.5.0
pcb_sn=CPE-2325-000121
ramdisk_addr_r=0xb000000
ramfs_name=-
rootpath=/srv/nfs/
serverip=0.0.0.0
soc=mvebu
stderr=serial@12000
stdin=serial@12000
stdout=serial@12000
vendor=Marvell
```

Most of these variables contain their default values are not actually used for
anything of importance on my device. U-Boot will simply wait the number of
seconds in `bootdelay` to allow the user to interrupt booting before running
the value of `bootcmd` which is `bootflow scan`.

If you chose not to upgrade the bootloader, `bootflow scan` will probably fail
as being an invalid command and you will have to configure U-Boot according to
your distribution's device-specific instructions.

BIOS-style booting uses the `booti` command as documented
[here](https://docs.u-boot.org/en/latest/usage/cmd/booti.html). Its inputs are
the locations in memory of the kernel, initramfs, and device tree. Your
`bootcmd` must load those files from a storage device to RAM (via `load`
command). This is a much more complicated configuration. Arch Linux ARM, for
example, wants you to set the following environment variables:

```
kernel_addr=0x2000000
ramdisk_addr=0x1100000
fdt_addr=0x1000000
fdt_high=0xffffffffffffffff
image_name=/boot/Image
ramdisk_name=/boot/initramfs-linux.uimg
fdt_name=/boot/dtbs/marvell/armada-3720-espressobin.dtb
get_env=if ext4load mmc 0 $loadaddr /boot/uEnv.txt; then env import -t $loadaddr $filesize; if test -n ${uenvcmd}; then run uenvcmd; fi; fi
get_images=ext4load mmc 0 $kernel_addr $image_name && ext4load mmc 0 $fdt_addr $fdt_name
get_ramdisk=ext4load mmc 0 $ramdisk_addr $ramdisk_name
bootargs=console=ttyMV0,115200 earlycon=ar3700_uart,0xd0012000 root=/dev/mmcblk0p1 rw rootwait
bootcmd=mmc dev 0; run get_env; if run get_images; then if run get_ramdisk; then booti $kernel_addr $ramdisk_addr $fdt_addr; else booti $kernel_addr - $fdt_addr; fi; fi
```

### Install a Linux distribution


#### Create a bootable USB thumb drive (Live USB)

Follow your distribution's instructions to prepare a [live
USB](https://en.wikipedia.org/wiki/Live_USB). What you are typically doing for
most distributions is flashing a pre-built disk image file (.iso) to a USB
flash device. This image usually contains a kernel that should boot all
"supported" devices and the basic system utilities needed to prepare storage
devices, download and install the distribution to non-removable storage (i.e.
eMMC/SATA).

While it looks promising, [Archboot](https://archboot.com/) didn't work
out-of-the-box for me.

#### Boot from the live USB

In order to use U-Boot's Standard Boot (`bootflow scan`), a distribution's live
USB image needs to be UEFI-compatible. If it isn't you will have to configure
U-Boot to load and boot the kernel/initramfs manually. Again, each distribution
does this differently so there is no universal configuration. The learning
curve for U-Boot is also steep and experimenting can only be done at the serial
console.

Arch Linux ARM's [device instructions and boot
image](https://archlinuxarm.org/platforms/armv8/marvell/espressobin), for
example, are not meant to work with Standard Boot and instruct manual U-Boot
configuration. Note that instructions for the ESPRESSObin also have to be
modified for the ESPRESSObin Ultra based on different storage and network
devices.

After you manage to successfully boot from a live USB, you can install your
distribution. This will include partitioning and formatting block devices. The
ESPRESSObin Ultra comes with 7.6G of block storage on eMMC. If this is not
enough for your chosen distribution and packages, it is possible to use the
device's internal SATA port to add significant internal storage capacity.


### Configure basic networking and get SSH working
The good news is that systemd is the default system service manager for both Debian-based distributions (Ubuntu, Raspbian) and Arch. A recommended first goal is to get to a point where you can ssh into your router when its WAN port is connected to your existing LAN. Until you do so, you will be working on the device via its serial console which means it has to be physically near another computer instead of in a closet or rack where it ultimately belongs.

I recommend disabling any firewall for this first step because it will be one more thing to troubleshoot when your ssh attempts fail. A sample WAN port configuration for systemd-networkd might look like the following:
```
[Match]
Name=wan

[Network]
DHCP=yes
DNSSEC=no
BindCarrier=end0
```

The ESPRESSSObin Ultra uses the [Distributed Switch Architecture (DSA)](https://docs.kernel.org/networking/dsa/dsa.html). The `end0` interface is the "cpu" or "conduit" Ethernet controller. It also need to be configured:

```
[Match]
Name=end0

# while this interface does need to be up for any of the ethernet
# interfaces to work, because it is just the DSA conduit, networkctl
# should ignore it for determining "online" status
[Link]
RequiredForOnline=no

# this interface doesn't need any IP address
[Network]
LinkLocalAddressing=no
```

These configuration files are typically meant to be created under `/etc/systemd/network`. After creating them, enable and start `systemd-networkd`. It can be very useful to look at the logs that are then produced to see if there are any obvious errors.

Try to find your device on the LAN (nmap is useful for that), and ping it. If pings are failing, you have to troubleshoot networking (see the great guide on the [Arch Linux wiki](https://wiki.archlinux.org/title/Network_configuration)).

If you haven't already, install and configure SSH. While there are some applications (like pi-hole) that come with an optional GUI, all system management and maintenance will happen over SSH. Now is an excellent time to make sure it is working well (e.g. create and use SSH keys).

### Configure a firewall
I use nftables so I have a sample configuration file that is meant to be used only when the device is ready to be used as a router. If you enable this configuration while SSH'd into the device while it's just a device on your LAN, your connection will get dropped and you'll have to go back to the serial console to fix it.

So the following configuration file can be set at `/etc/nftables.conf`, but do not enable the `nftables` service until you are ready to power down the device and make it the router on your network instead of a device on the network.

```
#!/usr/bin/nft -f
# vim:set ts=2 sw=2 et:

destroy table inet filter
table inet filter {
  chain input {
    type filter hook input priority filter; policy drop;
    ct state vmap { established : accept, related : accept, invalid : drop }
    iif "lo" accept comment "loopback"
    ip protocol icmp accept comment "icmp"
    meta l4proto ipv6-icmp accept comment "icmp v6"
    meta nfproto ipv6 udp sport dhcpv6-server udp dport dhcpv6-client accept comment "dhcpv6"
    iifname "br0" accept comment "allow devices on LAN ports to connect"
    reject
  }

  chain forward {
    type filter hook forward priority filter; policy drop;
    ct state vmap { established : accept, related : accept, invalid : drop }
    iifname "br0" accept comment "forward LAN traffic"
    reject with icmpx type no-route
  }
}
```

### Configure LAN interfaces

I recommend simply bridging all four LAN interfaces together to start. I have a file `lan.network`:

```
[Match]
Name=lan*

[Link]
RequiredForOnline=no

[Network]
Bridge=br0
BindCarrier=end0
```
Configure the bridge with `br0.netdev`:

```
[NetDev]
Name=br0
Kind=bridge
```
and `br0.network`:
```
[Match]
Name=br0

[Network]
Address=192.168.0.1/24
DHCPServer=yes
IPMasquerade=both
ConfigureWithoutCarrier=yes
MulticastDNS=yes
```

### See if it routes

Enable (but don't start) nftables so that the firewall will come up on next boot. 

Change the configuration for the WAN port to enable packed forwarding. Sample `wan.network`:

```
[Match]
Name=wan

[Network]
DHCP=ipv4
DNSSEC=no
BindCarrier=end0
IPv4Forwarding=yes
```
Get your serial console cable handy and be prepared to troubleshoot while physically connected to the device. Power the device off and connect the upstream network into the WAN port and downstream into a LAN port. Connect a serial console and power on the device.

Double check that nftables is running and didn't have any errors loading your configuration. Check systemd-networkd status too. If there are any errors, they will likely need to be resolved. If things seem fine over serial console, see if downstream devices are connecting to the router (i.e. getting assigned IP addresses, etc.).

### Customize further
IPV6 can be enabled though I've found some applications on an Android phone struggle with ipv6 and would recommend avoiding it for now.

Install and configure unbound so that you will handle your own DNS lookups instead of your ISP (or Google) knowing every website you visit.

Install and configure pi-hole for network-wide ad-blocking.

Configure VLANs.
