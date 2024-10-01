+++
title = 'bens.haus'
date = 2024-09-29T16:06:20-07:00
draft = false
+++

## Open Source ARM SoC Gateway/Router

This is a how-to guide for building and configuring your own gateway/router. This guide is about the software (bootloader and OS) and not the hardware.

### Acquire a [Globalscale ESPRESSOBIN Ultra](https://globalscaletechnologies.com/product/espressobin-ultra/)
This device was apparently the result of some kickstarter project. In hindsight, these devices are overpriced. The devices I received came running outdated and unmaintained forks of upstream open source projects (e.g. Linux and U-Boot). Their forks--having missed out on years of upstream development--are also very buggy.

But it is has most of the features one would expect from a basic small-office/home-office router including a 4 port switch (useful for VLANs). It has a WiFi/Bluetooth module, but it is not very good and requires the use of binary blobs. So I do not recommend using WiFi on this device.

### Build bootloader from source and flash
This is where the real "building" starts and basic familiarity with command-line Linux is expected. This step can be further broken down:
1. Acquire an x64 Linux build host. This can be a virtual machine running on Windows. Choosing a Debian-based distribution (e.g. Ubuntu Server) will generally make it easier to follow most [documentation](https://trustedfirmware-a.readthedocs.io/en/latest/plat/marvell/armada/build.html).
2. Install programs needed to compile the bootloader (build dependencies). Consult your distribution's packages and associated documentation. If you are missing a required dependency, the build will fail and usually give a fairly obvious error message about what program is missing.
3. [Get source code. Compile, test, and flash bootloader]((https://github.com/bschnei/ebu-bootloader)).
 
Note that flashing the bootloader is not a requirement for a stable device, but the remaining steps and configurations were developed assuming the bootloader is up-to-date. In particular [U-Boot's Standard Boot](https://docs.u-boot.org/en/stable/develop/bootstd.html) isn't supported in older version of U-Boot. This means you'll have to live with some combination of painful/hacky kernel upgrades and complex U-Boot config.

### Configure U-Boot
On this device, U-Boot stores it's configuration (environment variables) in dedicated storage completely separate from the bootloader (SPINOR), the eMMC, or any other block devices. This means that flashing the bootloader will not change the U-Boot environment variables.

### Install a Linux distribution
Choose a distribution that has good support for ARM, acquire their ARM64 install image with EFI support and flash it to a USB

### Configure

