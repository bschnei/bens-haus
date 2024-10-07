+++
title = 'DiY Router'
date = 2024-09-29T16:06:20-07:00
draft = false
+++

## Do-it-Yourself, Open Source ARM Gateway/Router

This is an outline of steps I followed to get up-to-date, mainstream U-Boot and Linux running on an ARM SoC network appliance. After many years of using [dd-wrt](https://dd-wrt.com/) as a router OS, I was tired of the dated UI, cumbersome update process, and lack of kernel updates--which are important for good security. I also become increasingly comfortable with Linux system administration and thought it would be a fun project to try to build my own router.

### Acquire a [Globalscale ESPRESSOBIN Ultra](https://globalscaletechnologies.com/product/espressobin-ultra/)
I believe this device was the result of a kickstarter project. The hardware (outside of the WiFi/Bluetooth module) seems fine, but the devices I received came running outdated and unmaintained forks of upstream open source projects (e.g. Linux and U-Boot). Their forks--having missed out on years of upstream development--are also very buggy.

It has most of the features one would expect from a basic small-office/home-office router including a 4 port switch (useful for VLANs). It also has a WiFi/Bluetooth module, but it is not very good and requires the use of closed-source drivers. I recommend separate, dedicated WiFI access point hardware and avoiding using this device's WiFi.

### Acquire an x64 Linux build host
If you do not have a device that runs Linux natively, you can also use a virtual machine. Choosing a Debian-based distribution (e.g. Ubuntu Server) will typically make it easier to follow project [documentation](https://trustedfirmware-a.readthedocs.io/en/latest/plat/marvell/armada/build.html).

### Build bootloader from source and flash
Instructions for this step can be found [here](https://github.com/bschnei/ebu-bootloader).
 
Note that flashing the bootloader is not a requirement for a stable device, but the remaining steps and configurations were developed assuming the bootloader is up-to-date. In particular [U-Boot's Standard Boot](https://docs.u-boot.org/en/stable/develop/bootstd.html) isn't supported in older version of U-Boot. This means you'll have to live with some combination of painful/hacky kernel upgrades and complex U-Boot config. Your CPU's maximum speed will also be limited.

### Configure U-Boot
On this device, U-Boot stores it's configuration (environment variables) in dedicated storage completely separate from the bootloader (SPINOR), the eMMC, or any other block devices. This means that flashing the bootloader will not change the U-Boot environment variables.

With a modern version of U-Boot, the only configuration is to set the environment variable `bootcmd` to `bootflow scan`. This will tell U-boot to cycle through available boot devices and look for an EFI system partition. When it finds one, it will try to load the EFI image at EFI/BOOT/BOOTAA64.EFI.

### Install a Linux distribution
Choose a distribution that has good support for 64-bit ARM (ARMv8) and their install image supports EFI booting. Acquire their install image and flash it to a USB storage device. You can confirm your USB device was imaged correctly by looking for an EFI image at the location noted above.

Boot from the USB device and install your distribution. This will include partitioning and formatting block devices. The ESPRESSObin Ultra comes with 7.6G of block storage on eMMC. If this is not enough for your chosen distro, it is possible to use the device's internal SATA port to add signifcant internal storage capacity.

I have been using Arch Linux on desktop for a few years, and while reliability is not desktop Arch's strength, it is absurdly customizable which is nice for building a lean operating system. Unfortunately the project doesn't officially support the ARM architecture. The Arch Linux ARM project is an unofficial port that I use, but the project doesn't share the community or infrastructure of Arch Linux. Spend time in the forums and documentation of the distributions you are considering and decide for yourself.

### Configure basic networking and get SSH working
The good news is that systemd is the default system manager for both Debian-based distributions (Ubuntu, Raspbian) and Arch. A recommended first goal is to get to a point where you can ssh into your router when its WAN port is connected to your existing LAN. Until you do so, you will be working on the device via its serial console which means it has to be physically near another computer instead of in a closet or rack where it belongs.

I recommend disabling any firewall for this first step because it will be one more thing to troubleshoot when your ssh attempts fail. A sample WAN port configuration for systemd-networkd might look like the following:
```
[Match]
Name=wan

[Network]
DHCP=yes
DNSSEC=no
BindCarrier=end0
```

The ESPRESSSObin Ultra uses the Distributed Switch Architecture (DSA). The `end0` interface is the "cpu" or "conduit" Ethernet controller. It also need to be configured:

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

Try to find your device on the LAN (nmap is useful for that), and ping it. If pings are failing, you have to troubleshoot networking (see the great guide on the Arch Linux wiki).

If you haven't already, install and configure SSH. While there are some applications (like pi-hole) that come with an optional GUI, all system management and maitnenance will happen over SSH. Now is an excellent time to make sure it is working well (e.g. create and use SSH keys).

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

### Customize furhter
IPV6 can be enabled though I've found some applications on an Android phone struggle with ipv6 and would recommend avoiding it for now.

Install and configure unbound so that you will handle your own DNS lookups instead of your ISP (or Google) knowing every website you visit.

Install and configure pi-hole for network-wide ad-blocking.

Configure VLANs.
