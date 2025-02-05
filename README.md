# Getting rid of false bad blocks on Mikrotik NAND devices
## This has been tested on RB433/RB433AG/RB493G

Mikrotik NAND flash devices can grow their bad block tables over time.
When burning an OpenWRT image directly onto flash and, particularly if a micro SD card is inserted during the flashing operation, lots of false bad blocks do develop, eventually reaching a high percentage of the storage, rendering the device unusable, incapable even of returning it to RouterOS. It is somewhat unclear to me why this happens but the answer may lie in the electronics, maybe there is a timing issue on the bus which the CPU talks to the flash, that bus is shared with the SD card.

If that's your case like it was mine, here are the instructions to repair it.

First you will need:
```
a TFTP and DHCP server with a network cable connected to ETH1/POE on RouterBoard
a serial console access to the Mikrotik device
a (supplied) OpenWRT ELF modified image capable of wiping out BBT (bad block table)
```

I will provide bellow instructions to build such an image from scratch if necessary, but it is already compiled in binary form in this repository.

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
Now put the provided **patch_mtd** on **~/archive/package/system/mtd/src** and right there do:
```
patch < patch_mtd
```
And put provided **patch_nand_base** on **~/archive/build_dir/target-mips_34kc_uClibc-0.9.33.2/linux-ar71xx_mikrotik/linux-3.18.84/drivers/mtd/nand** and patch nand driver
```
patch < patch_nand_base
```

Now compile (this will take a long time, for me it took half an hour):
```
cd ~/archive 
make -j"${nproc}" V=s
```
If everything goes well, you will find the compiled images on **~/archive/bin/ar71xx**

There, make a copy of **openwrt-ar71xx-mikrotik-vmlinux-initramfs.elf** to your TFTP server

Now at the Mikrotik console, power cycle it and press any key to interrupt boot on BIOS and select both options:
```
Boot from ethernet
DHCP boot
```
If everything is fine with the TFTP server and Mikrotik hardware, it will boot OpenWRT and end up on a prompt.
Now erase both NAND flash partitions:
```
mtd -n erase /dev/mtd5

mtd -n erase /dev/mtd6
```
OK, now you are able to use the standard Mikrotik procedure to return to RouterOS, just don't forget to reset routerboot BIOS to PXE boot. It's even possible to install a brand new openWRT but I am not covering this here right now. VERY IMPORTANT, don't forget to remove the SD card during these procedures.
Also don't forget to repartition FLASH to just one partition; this can be acomplished from Mikrotik BIOS

Good luck!
