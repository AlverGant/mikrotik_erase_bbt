# Getting rid of false bad blocks on Mikrotik NAND devices

Mikrotik NAND flash devices can grow their bad block table over time.
When burning an OpenWRT image on top of flash and particularly if a micro SD card is inserted during the flasing operation, lots of bad blocks develop, usually rendering the device unusable, incapable even of returning to RouterOS. It is somewhat unclear to my why that happens but the answer lie in the electronics, maybe a timing issue on the bus which the CPU talks to the flash which is shared with the micro SD card.

If that's the case, here are the instructions to repair it.

First you will need:
```
a TFTP server and a network cable connected to ETH1/POE on RouterBoard
a serial console access to the Mikrotik device
an OpenWRT modified image capable of wiping out BBT (bad block table)
```

I will provide instructions to build such an image from scratch if necessary and also to create a TFTP server on an OpenWRT device with a USB connector but any TFTP server will do it.

# Compiling a TFTP bootable ELF image
On an Ubuntu 14.04 64 bit machine or VM with at least 16GB storage and 8GB RAM run the following commands:
```
sudo apt update 
sudo apt -y upgrade
sudo apt install -y autoconf bison build-essential ccache file flex \
g++ git gawk gettext git-core libncurses5-dev libnl-3-200 libnl-3-dev \
libnl-genl-3-200 libnl-genl-3-dev libssl-dev ncurses-term python \
quilt sharutils subversion texinfo unzip wget xsltproc zlib1g-dev bc
sudo apt-get -y autoremove
cd ~
git clone -b chaos_calmer git://github.com/openwrt/archive
cd archive
make distclean
./scripts/feeds clean
./scripts/feeds update -a
./scripts/feeds install -a
make defconfig
make menuconfig
```
On the menu, check <*> all the following options:
```
Target System - Atheros AR7xxx
Subtarget - Mikrotik devices with NAND flash
Target Profile - default profile no WiFi
Target Images - ramdisk, tar.gz e squashfs
Base System = +ca-certificate
Network / File Transfer = +wget
Utilities = +rbcfg
```
Now compile (this will take a long time):
```
make -j"${nproc}" V=s
```
If everything goes well, you will find the compiled images on **~/souce/bin/targets/ar71xx/mikrotik**.

There copy **openwrt-ar71xx-mikrotik-vmlinux-lzma.elf** to your TFTP server

Now looking at Mikrotik console, reboot it, interrupt boot on BIOS and select both options:
```
Boot from ethernet and
DHCP boot
```
If everything is fine with the TFTP server and Mikrotik hardware, it will boot OpenWRT and end up on prompt.
Now erase the NAND flash:
```
mtd erase -N /dev/mtd5
mtd erase -N /dev/mtd6
```
