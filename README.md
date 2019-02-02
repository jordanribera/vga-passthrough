# vga-passthrough

## Hardware Used
Here is a list of the parts used in the system at time of writing:

Thing | Part
----- | ----
Motherboard | ASUS X399 Zenith Extreme
CPU | AMD Ryzen Threadripper 1900X (8 core, 16 thread)
RAM | 64GB (4x16) G.SKILL TridentZ DDR4 (3000MHz)
GPU (host) | AMD Radeon Pro WX 5100
GPU (guest) | EVGA GeForce GTX 1060 SC 6GB


## Host Preparation
This section details steps needed to prepare the host machine.

### BIOS settings
bios settings here

### Install Host OS
It's important to pick a linux distro+version that will get you a new kernel.
For my own build I picked Ubuntu 18.10, which is using kernel 4.18. You can
check the version of your linux kernel like so:

```
$ uname -a
Linux hostname 4.18.0-10-generic #11-Ubuntu SMP Thu Oct 11 15:13:55 UTC 2018 x86_64 x86_64 x86_64 GNU/Linux
```

### Install OVMF
You will need to provide qemu with OVMF, an open source UEFI firmware. I saw
several guides which detailed how to compile this or download it from an
external source. I was able to install it from existing/defautl repos using
apt:

```
$ sudo apt-get install ovmf
```

### Install qemu
The actual virtualization will be handled by qemu. This is another easy apt
install:

```
$ sudo apt-get install qemu-system-x86 qemu-utils
```

* Installing libvirt + virt-manager

## Exploration
Here are some commands used to inspect your system's IOMMU groupings.

### Check for IOMMU support
If nothing matches this grep, you've got a problem.
```
$ dmesg | grep -e IOMMU
[    0.614052] AMD-Vi: IOMMU performance counters supported
[    0.614100] AMD-Vi: IOMMU performance counters supported
[    0.641624] AMD-Vi: Found IOMMU at 0000:00:00.2 cap 0x40
[    0.641627] AMD-Vi: Found IOMMU at 0000:40:00.2 cap 0x40
[    0.643070] perf/amd_iommu: Detected AMD IOMMU #0 (2 banks, 4 counters/bank).
[    0.643083] perf/amd_iommu: Detected AMD IOMMU #1 (2 banks, 4 counters/bank).
[    6.356055] AMD IOMMUv2 driver by Joerg Roedel <jroedel@suse.de>
```

### Enumerate PCI-E devices
List all PCI devices:
```
$ lspci
```

From this list you will need to make note of the device IDs for the GPUs you are working with.

For Radeon cards:
```
$lspci -nn | grep ATI
```

For GeForce cards:
```
$lspci -nn | grep NVIDIA

## Banish the guest GPU
* Use values noted above
* Bind the vfio driver to guest gpu
* Reboot and check

## Create a VM
* Basic setup
* Adding PCI devices
* What about input?
* First boot & Code 43
*
