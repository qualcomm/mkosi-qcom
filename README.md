# Introduction

[mkosi](https://mkosi.systemd.io/) is a fancy wrapper to facilitate creating
customized OS disk images. This repository contains configuration files and
documentation helpful for utilizing *mkosi* to create disk images to assist in
development and testing of platform-software on Qualcomm-based development
boards (with UEFI boot).

The repository basic contains configuration for **Arch Linux Arm**, **Debian**,
**Fedora**, and **Ubuntu**.

# Installing mkosi

A recent version of *mkosi* is required, install this by issuing:

```
pipx install -f git+https://github.com/systemd/mkosi.git
```

Ensure your host OS is configured per the details in the [Frequently asked
questions](#Frequently-asked-questions) below.

# Baking an image

Invoke *mkosi* with the **build** verb in order to build an image.

```
mkosi -d <distro> [--devicetree <dtb>] [--profile <profile>] [-f] build
```

The **-d** argument can be used to select the distribution you want to build.
The provided configuration supports **arch**, **debian**, **fedora**, **ubuntu**.

Many Qualcomm boards does not come with upstream-compatible DeviceTree blobs
loaded by UEFI, the **--devicetree** property can be provided, the DeviceTree
blob will be expected to be installed by selected packages for the
distribution.

Some boards require custom settings, for this **--profile** can be used to
select a relevant profile. See below section about *mkosi.profiles*.

The **-f** option tells *mkosi* to overwrite a previously built disk image.

*mkosi* will produce an output image names **image.raw**, containing the full
disk image containing EFI system partition and root file system.

**Note:** many targets lacks runtime firmware in the *[linux-firmware](https://web.git.kernel.org/pub/scm/linux/kernel/git/firmware/linux-firmware.git/)*
project, see note below on where in *mkosi.extra* to provide this.

## Example
```
mkosi -d debian --devicetree qcom/qcs6490-rb3gen2.dtb --profile qcs6490-rb3gen2 -f build
```
*-f option will overwrite any previously baked image, instead of failing*

## Flashing the baked image

The resulting image will be called *image.raw*, this can be flashed onto most
boards with existing non-Linux firmware (boot etc), using e.g. [QDL](https://github.com/linux-msm/qdl)
and the included **rawprogram-ufs.xml** and **rawprogram-nvme.xml**.

For UFS-absed devices use:
```
qdl prog_firehose_ddr.elf rawprogram-ufs.xml
```

For NVMe-based devices:

```
qdl prog_firehose_ddr.elf rawprogram-nvme.xml
```

*On some targets xbl_s_devprg_ns.melf should be used instead of
prog_firehose_ddr.elf, neither is provided through this project*

If your device has multiple storage devices, such as UFS and NVME, use
*--storage nvme* or *--storage ufs* to direct QDL to the appropriate
destination.

## Booting the image

Once flashed, power on the board and you should boot into the newly installed
disk image. Some distributions will ask for timezone settings etc during boot,
answer as desired.

Depending on the hardware support for your board, you may either use an
attached display and keyboard for this, or access the board over UART.

### Logging in

Per the configuration log into the system using:

- user: **root**
- password: **14**

## linux-firmware

Not all boards have firmware available in *[linux-firmware](https://web.git.kernel.org/pub/scm/linux/kernel/git/firmware/linux-firmware.git/)*
and/or the related packages of your Linux distribution. These files can be
added as necessary under **mkosi.extra/usr/lib/firmware** and will then be
baked into the image.

# Structure of the project

## mkosi.conf

Base configuration, common for all boards and distributions. In particular it
defines that the we're building a disk image, of the *arm64* architecture, that
it's *bootable* using *systemd-boot*, with a common kernel command line.

Read more about these options in the official [mkosi man page](https://github.com/systemd/mkosi/blob/main/mkosi/resources/man/mkosi.1.md)

## mkosi.conf.d

Contains extensions to the base *mkosi.conf*, matched on the selected
distribution. Among a few other things, this is where the default list of
packages to be installed for each distribution is defined.

## mkosi.extra

Provides an overlay of a few configuration files to improve the out-of-the-box
experience of the built image.

| file | Comments |
| --- | --- |
| 50-root.conf | Automatically extend the root partition (but not the file system) to cover the remainder of the disk |
| 10-network-manager.preset | Make the NetworkManager systemd service launch automatically on boot (if present) |

### Runtime firmware

A placeholder directory **mkosi.extra/usr/lib/firmware** exists as a
convenience for users to place any runtime firmware which is not provided
through *[linux-firmware](https://web.git.kernel.org/pub/scm/linux/kernel/git/firmware/linux-firmware.git/)*.

## mkosi.packages

Empty directory in which you can add any custom built packages for the
distribution your building.

## mkosi.profiles

Some boards requires some further customizations, these are captured in as
*mkosi profiles* in the *mkosi.profiles* directory. One concrete example is the
need to specify the 4k sector size for UFS-based devices.

Currently provided profiles are:
| profile | Comments |
| --- | --- |
| ufs | General purpose profile to override the SectorSize config, as needed on UFS-based devices |
| qcs6490-rb3gen2 | Override the SectorSize config and increase deferred_probe_timeout on kernel command line |

Not all boards require a profile to be selected, e.g. SC8280XP and X1 Elite
devices with NVMe storage works without this.

# Build your own kernel

The upstream Linux kernel community aims to maintain *defconfig* to the point
where necessary hardware drivers for running the kernel on supported Qualcomm
hardware are enabled by default.

Kernel packages can then be built for the various distributions using below
commands and copied into **mkosi.packages**. Note that the *Packages* list in
relevant *mkosi.conf.d* config might need to be updated for the new package to
be selected.

## Arch Linux
```
LLVM=1 ARCH=arm64 make -j99 pacman-pkg
```

Copy the resulting **linux-upstream** files into *mkosi.packages* and make
sure that **linux-upstream** is in the *Packages=* directive in
*mkosi.conf.d/arch.conf*.

## Debian or Ubuntu
```
LLVM=1 ARCH=arm64 make -j99 O=debian bindeb-pkg
```

Copy the resulting **\*.deb** files into *mkosi.packages*. Use **dpkg-deb
--into mkosi.packages ...** to acquire the **Package:** specifier of the kernel
packet, then plug this into the *Packages=* directive in
*mkosi.conf.d/debian.conf* or *mkosi.conf.d/ubuntu.conf*.

## Fedora
```
LLVM=1 ARCH=arm64 make -j99 binrpm-pkg
```

Copy **rpmbuild/RPMS/*.rpm** files into *mkosi.packages*.

### Note on compression and UEFI

The provided ***mkosi.conf*** does enable the packaging of the kernel and
related parts into a UKI. If you disable this, you need to make sure that the
kernel is uncompressed, or built with the **CONFIG_EFI_ZBOOT** option, as UEFI
does not decompress the kernel.

## Managing DeviceTree for your kernel

In the event that your board's UEFI implementation does not provide an upstream
compliant Devicetree blob, you need to ensure the board's dtb file is loaded
during boot - e.g. by systemd-boot.

When baking the image, the *--devicetree* option provides the means for
selecting a DeviceTree blob from the installed packages. When installing the
generated package into an existing installation, make sure that the generated
*systemd-boot* loader entry contains the correct **devicetree** entry, or that
you build an *UKI* with the relevant DeviceTree blob included.

# Frequently asked questions

## Does this work on a device with Android Boot Loader (abl)

No. *mkosi* generates disk images of standard Linux distributions, that expects
to be loaded by an UEFI bootloader.

## mkosi system dependencies

Install the dependencies on your host system, for Ubuntu 22.04 this is:

```
sudo apt install binfmt-support dnf mkosi uidmap pipx rpm systemd-repart qemu-user-static qemu-system-arm
```

Make sure pipx-installed binaries are in your $PATH:

```
pipx ensurepath
```

Restart your terminal/shell.

Set up sub-UIDs/GIDs for containerization

```
echo "$USER:200000:65536" | sudo tee -a /etc/subuid
echo "$USER:200000:65536" | sudo tee -a /etc/subgid
```

Enable the Aarch64 emulation using QEMU.

```
sudo update-binfmts --enable qemu-aarch64
```

### umask

If you system is configured to a umask other than 0022, subsequent runs of
*mkosi* will always fail with various permission issues. So make you set the
umask in the terminal you're about to run *mkosi*:

```
umask 0022
```

## systemd-* is too old

*mkosi* depends on a variety of fairly recent features in *systemd*. To
faciliate this *mkosi* includes a feature which first builds an image with
tools, and then builds the image in that container.

Enable this by setting **ToolsTree=default** in the *Build* section of the
config.

## $HOME is a network mount

*mkosi* uses namespaces to set up Linux distribution to be built into the
image, this does not play well with network mounted, such as NFS, home
directories. Instead clone the *mkosi-qcom* project into some local storage;
such as */mnt/local/mkosi-qcom* (subject to your system configuration).

At runtime *mkosi* will use the user's cache directory, create a directory for
this on local storage and update the *XDG_CACHE_HOME* to redirect these
accesses like:

```
export XDG_CACHE_HOME=/mnt/local/tmp
```

# Contribute

With the goal of providing a convenient development environment for upstream
work, please do contribute to documentation and configuration by opening a Pull
Request. Issues can be used to track issues with this content, or specifics
that needs to be corrected in the various Linux distributions.

See [CONTRIBUTING](CONTRIBUTING.md) for more information.

## License

Licensed under [BSD 3-Clause Clear License](LICENSE.txt).
