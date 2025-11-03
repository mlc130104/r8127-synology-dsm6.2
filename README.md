# r8127-synology-dsm6.2

This repo contains code and instructions to compile and install the RTL8127 driver for Synology DSM 6.
If you are running DSM 7+, there is actually a Chinese package source  offering precompiled packages for RTL8125/8126/8127 network chipsets.

My current setup:

- Synology DS1621+ (v1000)
- DSM 6.2.4 (Linux 4.4.59+)
- DieWu 10G SFP PCIe 3.0x4 Network Adapter TX403 (RTL8127AF)

Do not expect support if you run into problems, as this is just a note for myself to reproduce a working solution for DSM6.2.

## Compile module

Steps below are based [this](https://help.synology.com/developer-guide/getting_started/prepare_environment.html) and [this](https://help.synology.com/developer-guide/compile_applications/compile_open_source_projects.html) docs.

1. Create a suitable Linux environment (not your NAS!). I used a Ubuntu 22.04 LTS Live USB:
Download an image for your system architechture (again, not your NAS) and burn it with etcher.

https://releases.ubuntu.com/jammy/
https://etcher.balena.io

3. Set up environment:

```bash
sudo -i
apt-get update
apt-get install git python3 python3-pip
mkdir -p /toolkit
cd /toolkit
git clone https://github.com/SynologyOpenSource/pkgscripts-ng
```

3. Deploy chroot environment:
Download 3 files from Synology Achrive (https://archive.synology.com/download/ToolChain/toolkit/6.2). Leave them at Downloads folder to save time.

```
base_env-6.2.txz
```

``` #replace with your system achitechture
ds.v1000-6.2.env.txz
ds.v1000-6.2.dev.txz
```
You may need to rename the files manually to 6.2.4 (or your target version)

```bash
cd /toolkit/pkgscripts-ng
git checkout DSM6.2.4 # replace 6.2.4 with your system platform
./EnvDeploy -v 6.2.4 -p v1000 -t /home/ubuntu/Downloads # replace 'avoton' with your platform
cp /etc/ssl/certs/ca-certificates.crt /toolkit/build_env/ds.v1000-6.2.4/etc/ssl/certs/ca-certificates.crt # replace 'ds.v1000-6.2.4' with your platform and version
```

4. Chroot into environment:

```bash
chroot /toolkit/build_env/ds.v1000-6.2.4
```

5. Download code:

```bash
mkdir -p /usr/src
cd /usr/src
git clone https://github.com/openwrt/rtl8127.git
```
6. Edit Makefile
After
``	ccflags-y  += $(EXTRA_CFLAGS)
else
``
replace all following lines with:

`` BASEDIR := /usr/local/x86_64-pc-linux-gnu/x86_64-pc-linux-gnu/sys-root/usr/lib/modules/DSM-6.2.4
	KERNELDIR ?= $(BASEDIR)/build
	PWD :=$(shell pwd)
	DRIVERDIR := $(shell find $(BASEDIR)/kernel/drivers/net/ethernet -name realtek -type d)
	ifeq ($(DRIVERDIR),)
		DRIVERDIR := $(shell find $(BASEDIR)/kernel/drivers/net -name realtek -type d)
	endif
	ifeq ($(DRIVERDIR),)
		DRIVERDIR := $(BASEDIR)/kernel/drivers/net
	endif

	KERNEL_GCC_VERSION := $(shell cat /proc/version | sed -n 's/.*gcc version \([[:digit:]]\.[[:digit:]]\.[[:digit:]]\).*/\1/p')
	CCVERSION = $(shell $(CC) -dumpversion)

.PHONY: all
all: print_vars clean modules

print_vars:
	@echo
	@echo "CC: " $(CC)
	@echo "CCVERSION: " $(CCVERSION)
	@echo "BASEDIR: " $(BASEDIR)
	@echo "DRIVERDIR: " $(DRIVERDIR)
	@echo "PWD: " $(PWD)
	@echo

.PHONY:modules
modules:
	$(MAKE) -C $(KERNELDIR) M=$(PWD) modules

.PHONY:clean
clean:
	$(MAKE) -C $(KERNELDIR) M=$(PWD) clean

endif
``

If you're compiling for a different DSM version, you need to update `BASEDIR` in the makefile:

```
	BASEDIR := /usr/local/x86_64-pc-linux-gnu/x86_64-pc-linux-gnu/sys-root/usr/lib/modules/DSM-6.2.4
```

6. Compile module:

```bash
$ make
```

I end up with some build errors but `r8217.ko` is compiled in the current directory.

## Install module

1. Copy `r8215.ko` to a location available from your Synology device, like a shared folder or using SCP.

2. Connect using SSH and run the following commands:

```bash
sudo -i
cp r8217.ko /lib/modules
insmod /lib/modules/r8127.ko
```

3. Confirm the module is loaded:

```
root@synology:~# lspci -v | grep r8127
	Kernel driver in use: r8127
```

4. Set up new interface:

```bash
$ ip link set up eth4 # replace 'eth4' with your interface
```

At this point, you should be able to see interface details using `ifconfig eth4`. (or eth2 if you have a dual port model)

5. Update modules so `r8127` is loaded automatically at next boot:

Open Control Panel > Task Scheduler > Create Trigger Task

A new task at boot with the following command

```bash
insmod /lib/modules/r8127.ko
ip link set up eth4
/usr/syno/sbin/synonet --manual eth4 192.168.1.225 255.255.255.0
/usr/syno/sbin/synonet --set_gateway 192.168.1.1
/usr/syno/sbin/synonet --set_dns 192.168.1.1

```
Replace the IP, gateway & DNS, subnet mask with your own network setting. 
---

Viola!

## Changelog

- 2023-07-08: Update driver version to 9.009.02.
- 2023-07-08: Initial commit, support for driver version 9.006.04.
- 2025-11-03: Adoption for R8127 and DSM6.2.4
