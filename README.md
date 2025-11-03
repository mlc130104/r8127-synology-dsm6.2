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
$ cd /toolkit/pkgscripts-ng
$ git checkout DSM6.2.4 # replace 6.2.4 with your system platform
$ ./EnvDeploy -v 6.2.4 -p v1000 -t /home/ubuntu/Downloads # replace 'avoton' with your platform
$ cp /etc/ssl/certs/ca-certificates.crt /toolkit/build_env/ds.v1000-6.2.4/etc/ssl/certs/ca-certificates.crt # replace 'ds.v1000-6.2.4' with your platform and version
```

4. Chroot into environment:

```bash
$ chroot /toolkit/build_env/ds.avoton-7.2
```

5. Download code:

```bash
$ mkdir -p /usr/src
$ cd /usr/src
$ git clone https://github.com/tabrezm/r8125-synology
$ cd r8125-synology/src
```

If you're compiling for a different DSM version, you need to update `BASEDIR` in the makefile:

```
	BASEDIR := /usr/local/x86_64-pc-linux-gnu/x86_64-pc-linux-gnu/sys-root/usr/lib/modules/DSM-7.2
```

6. Compile module:

```bash
$ make
```

You should have no build errors and `r8215.ko` in the current directory.

## Install module

1. Copy `r8215.ko` to a location available from your Synology device, like a shared folder or using SCP.

2. Connect using SSH and run the following commands:

```bash
$ sudo -i
$ cp r8215.ko /lib/modules
$ insmod /lib/modules/r8125.ko
```

3. Confirm the module is loaded:

```
root@synology:~# lspci -v | grep r8125
	Kernel driver in use: r8125
```

4. Set up new interface:

```bash
$ ip link set up eth4 # replace 'eth4' with your interface
```

At this point, you should be able to see interface details using `ifconfig eth4`.

5. Update modules so `r8125` is loaded automatically at next boot:

```bash
$ ln -s /bin/kmod /sbin/depmod
$ depmod -a # warnings are safe to ignore
```

---

## Change log

- 2023-07-08: Update driver version to 9.009.02.
- 2023-07-08: Initial commit, support for driver version 9.006.04.
