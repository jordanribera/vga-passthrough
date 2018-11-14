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
* Check dmesg for IOMMU support
* Enumerate devices by IOMMU groups
* Make note of guest GPU's IDs etc

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
