# TBS6910 kernel driver port

## Introduction

The [TBSDTV DVB driver set](https://github.com/tbsdtv/linux_media) is great, but may throw up hiccups with bespoke kernels.

This is how to port over a specific TBS driver (in this case, TBS6910) from scratch, with detailed workings from a recent pass on Raspberry Pi Linux (Kernel 6.6, Debian Bookworm, 64-bit) running on a Raspberry Pi Compute Module 4 and IO board. 

## Kernel groundwork

### Gathering kernel information

Checks latest kernel version is on the destination:
```
$ uname -r
6.6.31+rpt-rpi-v8
```

Track down the representative kernel sources for that kernel. Installing the kernel headers and looking at the changelog a good route for Raspberry Pi Linux:
```
$ sudo apt install raspberrypi-kernel-headers
$ zcat /usr/share/doc/linux-image-$(uname -r)/changelog.Debian.gz | grep -E -m 1 -o '[0-9a-f]{40}'
c1432b4bae5b6582f4d32ba381459f33c34d1424
```

Go off and find that commit at https://github.com/raspberrypi/linux or wherever the linux sources are, as without a refereence point, the modules may not match the kernel.

### (Optional) Creating a temporary build location

Mount a faster compilation disk if required:
```
$ sudo mount /dev/sda1 /mnt/
$ sudo mkdir /mnt/dvb-build
$ sudo chown $USER:$USER /mnt/dvb-build
```

### Compilation software install

Set up software on the host system doing the building:
```
$ sudo apt update
$ sudo apt upgrade
$ sudo apt install build-essential git bc bison flex libssl-dev libncurses5-dev
```

### Kernel sourcing and preperation

Download the TBS kernel sources, latest commit:
```
$ cd /mnt/dvb-build/
$ git clone --depth 1 https://github.com/tbsdtv/linux_media
```

Download the host kernel source. For example, for Raspberry Pi Linux:
```
$ zcat /usr/share/doc/linux-image-6.6.31+rpt-rpi-v8/changelog.Debian.gz | grep -E -m 1 -o '[0-9a-f]{40}'
c1432b4bae5b6582f4d32ba381459f33c34d1424
$ git clone --branch rpi-6.6.y https://github.com/raspberrypi/linux
$ cd linux
$ git checkout c1432b4bae5b6582f4d32ba381459f33c34d1424
```

Copy over the kernel header configuration elements and initialise:
```
$ cp /usr/src/linux-headers-6.6.31+rpt-rpi-v8/Module.symvers .
$ cp /usr/src/linux-headers-6.6.31+rpt-rpi-v8/.config .
$ make oldconfig
```

(Optional) Take a copy of the original source for future patching purposes, so we can track what is changed:
```
$ cd ..
$ cp -r linux linux.orig
```

## Porting DVB Elements

> [!WARNING]  
> This is all off-piste. Keep backups and run compilations to validate the work. 

### Moving over the Linux DVB frontend headers
Copy over the Linux kernel DVB frontend headers:
``` 
$ cp ../linux_media/include/media/dvb_frontend.h \
     ../linux/include/media/dvb_frontend.h 
$ cp ../linux_media/include/uapi/linux/dvb/frontend.h \
     ../linux/include/uapi/linux/dvb/frontend.h
```

> [!IMPORTANT]  
> There is a switch case clash with `include/uapi/linux/dvb/frontend.h`
> To fix, change this definition:
> `#define DTV_DVBT2_PLP_ID_LEGACY 43`

### Main tbsecp3 driver porting

Copy over the main driver directory:
```
$ cp -r linux_media/drivers/media/pci/tbsecp3 \
        linux/drivers/media/pci/
```

Look for the changes needed to be made for the `Kconfig` & `Makefile` in `linux/drivers/media/pci/`
```
$ grep -i -B1 tbsecp3 linux_media/drivers/media/pci/Kconfig 
source "drivers/media/pci/smipcie/Kconfig"
source "drivers/media/pci/tbsecp3/Kconfig"

$ grep -i -B1 tbsecp3 linux_media/drivers/media/pci/Makefile 
		netup_unidvb/	\
		tbsecp3/	\
```
Apply the changes that are seen to those in `linux/drivers/media/pci/`.

Locate the **frontend demodulator** and **tuner** names, using the `tbsXXXX` model name to track these down:
```
$ grep "config tbs6910_" linux/drivers/media/pci/tbsecp3/tbsecp3-dvb.c
    static struct tas2101_config tbs6910_demod_cfg[] = {
    static struct av201x_config tbs6910_av201x_cfg = {
```

In this case we have:
- **TBS6910** - the DVB card
- **TAS2101** - the frontend demodulator for the card
- **AV201x** - the tuner for the card

Use all three keywords as an aid to prune `linux/drivers/media/pci/tbsecp3/tbsecp3-dvb.c` to the basics:
1) Delete any unneeded headers for the other demodulators / tuners
2) Locate `switch(dev->info->board_id)` in the code, then:
    - Delete every case statement for unneeded boards
    - Leave the `default:` case statement alone
3) Remove all functions that are unneeded
    - Use the compilation steps below to aid with pruning out unused functions

### Tuner and demodulator porting

Check to see if either the frontend demodulator and tuner have been included in the default kernel:
```
$ ls linux/drivers/media/dvb-frontends/tas2101*
$ ls linux/drivers/media/tuners/av201*
```

If not, copy the files over and make changes to the `Kconfig` & `Makefile`:
```
$ cp -v linux_media/drivers/media/tuners/av201x* \
        linux/drivers/media/tuners/
$ grep -i -B10 -A10 av201 linux_media/drivers/media/tuners/Kconfig
$ grep -i -B1 av201 linux_media/drivers/media/tuners/Makefile

$ cp -v linux_media/drivers/media/dvb-frontends/tas2101* \
        linux/drivers/media/dvb-frontends/
$ grep -i -B10 -A10 tas2101 linux_media/drivers/media/dvb-frontends/Kconfig 
$ grep -i -B1 tas2101 linux_media/drivers/media/dvb-frontends/Makefile
```

Apply the changes as the user sees fit.

> [!IMPORTANT]  
> The following line needs to be added to `tas2101.c` to stop a `KERNEL_VERSION` error appearing: 
> `#include <linux/version.h>`

## Patching in the driver
Using a patch instead of the above can be applied as follows:
```
$ cd linux
$ patch -p1 < tbs6910.patch
```

(Optional) If a patch is to be created, simply do:
```
$ diff -urN linux.orig linux > tbs9210.patch
```

## Compilation

Enable the drivers manually or graphically:

**Manually**
```
$ vi .config
CONFIG_DVB_TBSECP3=m
CONFIG_MEDIA_TUNER_AV201X=m
CONFIG_DVB_TAS2101=m
```

**Graphically** 
```
$ make menuconfig
  Device Drivers
    Multimedia support
      Media drivers
        Media PCI Adapters
          <M> TBS ECP3 FPGA based cards
      Media ancillary drivers
        Customize TV tuners
          <M> Airoha Technology AV201x silicon tuner
        Customise DVB Frontends
          <M> Tmax TAS2101 based
```

Module compilation test runs of each part:
```
$ make prepare
$ make modules_prepare
$ make -j4 M=drivers/media/tuners
$ make -j4 M=drivers/media/dvb-frontends
$ make -j4 M=drivers/media/dvb-core
$ make -j4 M=drivers/media/pci/tbsecp3
```

The last one `tbsecp3` might have some linker warnings. This is normal as it wants to resolve symbols from the other modules. To complete this, run a compilation of the whole `drivers/media` directory:
```
$ make prepare
$ make modules_prepare
$ make -j4 M=drivers/media
```

## Installation

### Installing locally
To install onto the same system after compilation:
```
$ make modules_install M=drivers/media
```

### Installing elsewhere

To move to another system, bundle up the specific drivers:
```
$ tar cfvz ../tbs6910-drivers.tar.gz \
            drivers/media/dvb-core/dvb-core.ko \
            drivers/media/dvb-frontends/tas2101.ko \
            drivers/media/tuners/av201x.ko \
            drivers/media/pci/tbsecp3/tbsecp3.ko
```

Transfer it over the destination system and then unpack the modules and refresh the module symbols:
```
$ sudo tar xfvz tbs6910-drivers.tar.gz \
                --directory=/lib/modules/6.6.31+rpt-rpi-v8/kernel
$ sudo depmod -a
```

## Verification

Use `modprobe tbsecp3` or `insmod` to launch the tbsecp3 module.

Look at the kernel messages to check to see if working:
```
$ dmesg | grep TBS
[    6.207451] TBSECP3 driver 0000:01:00.0: enabling device (0000 -> 0002)
[    6.207629] TBSECP3 driver 0000:01:00.0: TurboSight TBS 6910 DVB-S/S2 + 2xCI 
[    6.226170] dvbdev: DVB: registering new adapter (TBSECP3 DVB Adapter)
[    6.387691] TBSECP3 driver 0000:01:00.0: MAC address 00:22:ab:d1:42:52
```

### Known issues

If however this is seen, then it's likely to be a 64-bit OS / 32-bit device issue:
```
$ dmesg | grep TBS
[    6.572114] TBSECP3 driver 0000:01:00.0: dma: memory alloc failed
[    6.572186] TBSECP3 driver 0000:01:00.0: probe error
[    6.572198] TBSECP3 driver: probe of 0000:01:00.0 failed with error -12
```

To fix, switch on **PCIe 32-bit DMA** support. For example, if running on a Raspberry Pi with a 64-bit OS, add the appropriate overlay to `config.txt`:
```
$ vi /boot/firmware/config.txt
dtoverlay=pcie-32bit-dma
```
