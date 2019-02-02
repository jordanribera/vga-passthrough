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
$ lspci -nn | grep ATI
43:00.0 VGA compatible controller: Advanced Micro Devices, Inc. [AMD/ATI] Ellesmere [Radeon Pro WX 5100]
43:00.1 Audio device: Advanced Micro Devices, Inc. [AMD/ATI] Ellesmere [Radeon RX 580]
```

For GeForce cards:
```
$ lspci -nn | grep NVIDIA
0b:00.0 VGA compatible controller: NVIDIA Corporation GP106 [GeForce GTX 1060 6GB] (rev a1)
0b:00.1 Audio device: NVIDIA Corporation GP106 High Definition Audio Controller (rev a1)
```

Make a note of the IDs and bus designations for both virtual devices (VGA controller and audio device) associated with your guest GPU:
```
VGAPT_VGA_ID='10de:1401'
VGAPT_VGA_AUDIO_ID='10de:0fba'
VGAPT_VGA_BUS=01:00.0
VGAPT_VGA_AUDIO_BUS=01:00.1
```

## Banish the guest GPU
In order to make use of the guest GPU in our virtual machine, we need to ensure that the host operating system doesn't latch onto it. To avoid this, we bind the vfio driver to the guest gpu at boot (using the IDs noted above):
```
$ echo options vfio-pci ids=$VGAPT_VGA_ID,$VGAPT_VGA_AUDIO_ID > /etc/modprobe.d/vfio.conf
$ printf "vfio\nvfio_pci\n" > /etc/modules-load.d/vfio.conf
$ update-initramfs -u
```

Reboot the host machine and then verify that the vfio driver is binding properly. Using `lspci -v` and finding the guest GPU in the output will tell us which driver is in use:
```
$ lspci -v
...
0b:00.0 VGA compatible controller: NVIDIA Corporation GP106 [GeForce GTX 1060 6GB] (rev a1) (prog-if 00 [VGA controller])
	Subsystem: eVga.com. Corp. GP106 [GeForce GTX 1060 6GB]
	Flags: fast devsel, IRQ 145, NUMA node 0
	Memory at fa000000 (32-bit, non-prefetchable) [size=16M]
	Memory at 86a0000000 (64-bit, prefetchable) [size=256M]
	Memory at 86b0000000 (64-bit, prefetchable) [size=32M]
	I/O ports at 1000 [size=128]
	Expansion ROM at fb000000 [disabled] [size=512K]
	Capabilities: <access denied>
	Kernel driver in use: vfio-pci
	Kernel modules: nvidiafb, nouveau

0b:00.1 Audio device: NVIDIA Corporation GP106 High Definition Audio Controller (rev a1)
	Subsystem: eVga.com. Corp. GP106 High Definition Audio Controller
	Flags: fast devsel, IRQ 146, NUMA node 0
	Memory at fb080000 (32-bit, non-prefetchable) [size=16K]
	Capabilities: <access denied>
	Kernel driver in use: vfio-pci
	Kernel modules: snd_hda_intel
...
```

The presence of `Kernel driver in use: vfio-pci` on both devices means the driver is binding as intended.

## Create a VM
Many of the guides available when I explored this process recommended manually invoking qemu via a script which holds a complex set of flags and variables. I was pleased to find that the GUI application virt-manager was perfectly capable of handling most of the configuration.

### Basic setup
Create a virtual machine as usual. I may return to expand this later.

### Adding PCI devices
Adding PCI devices is fairly straightforward. While viewing the virtual machine's properties:
* Click "Add Hardware"
* Select "PCI Host Device"
* Scroll and select the appropriate device (in my case `0000:0b:00.0` and `0000:0b:00.1`)

### Input and Output
With this configuration, the guest GPU will output to an attached monitor just like a separate system. The easiest way to manage input to the virtual machine is to plug in a second mouse and keyboard and pass them directly to the guest. Another approach would be to attach your monitor and peripherals to a KVM, with the guest and host systems on different registers.

A more advanced approach is to use evdev to route input data to the virtual machine. This works especially well in conjunction with [Looking Glass](https://looking-glass.hostfission.com/). With an HDMI or DisplayPort [dummy plug](https://www.amazon.com/gp/product/B077CZ6JC3/) attached, the guest GPU will render frames to its frame buffer as normal. The shared memory mapping and client binary provided by Looking Glass allows you to directly read that frame buffer and interact with your guest inside of a window.

## More Later...
* First boot & Code 43
