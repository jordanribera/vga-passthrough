# vga-passthrough

## Hardware used
Here is a list of the parts used in the system at time of writing:

Thing | Part
----- | ----
Motherboard | ASUS X399 Zenith Extreme
CPU | AMD Ryzen Threadripper 2950X (16 core, 32 thread)
RAM | 64GB (4x16) G.SKILL TridentZ DDR4 (3000MHz)
GPU (host) | AMD Radeon Pro WX 5100
GPU (guest) | RTX 2080ti Founders Edition


## Host preparation
This section details steps needed to prepare the host machine.

### BIOS settings
Before you start, go into your system's BIOS and ensure that IOMMU and AMD-v
are enabled. Note that IOMMU should be **"enabled"**, not **"auto"**. This does
make a difference.

### Install host OS
It's important to pick a linux distro+version that will get you a new kernel.
For this guide I used Ubuntu 19.04, which is using version 5.0.x of the Linux
kernel. These same steps worked just as well for me with Ubuntu 18.10 with
kernel version 4.19. You can check your kernel version like so:

```
$ lsb_release -dc
Description:	Ubuntu 19.04
Codename:	disco

$ uname -r
5.0.20-050020-generic
```

### Install OVMF
You will need to provide qemu with OVMF, an open source UEFI firmware. I saw
several guides which detailed how to compile this or download it from an
external source. I was able to install it from existing/default repos using
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

I also installed libvirt and virt-manager, which is a graphical tool for
managing virtual machines.

```
sudo apt-get install libvirt virt-manager
```

## Exploration
Here are some commands used to inspect your system's pci-e devices.

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
We can sift through the system's pci-e devices using the `lspci` command. We
start off by finding both of our GPUs with the following command:
```
$ lspci | grep VGA
0a:00.0 VGA compatible controller: NVIDIA Corporation GV102 (rev a1)
43:00.0 VGA compatible controller: Advanced Micro Devices, Inc. [AMD/ATI] Ellesmere [Radeon Pro WX 5100]
```

The first value on each of those lines represents the pci-e bus and slot for
that device. In addition to the bus/slot, we will also need the device IDs for
the cards in those slots. Use `-s` to filter by each of the two slots, and also
`-nn` to display the device ID:
```
$ lspci -s 0a:00 -nn
0a:00.0 VGA compatible controller [0300]: NVIDIA Corporation GV102 [10de:1e07] (rev a1)
0a:00.1 Audio device [0403]: NVIDIA Corporation TU102 High Definition Audio Controller [10de:10f7] (rev a1)
0a:00.2 USB controller [0c03]: NVIDIA Corporation TU102 USB 3.1 Controller [10de:1ad6] (rev a1)
0a:00.3 Serial bus controller [0c80]: NVIDIA Corporation TU102 USCI Controller [10de:1ad7] (rev a1)

$ lspci -s 43:00 -nn
43:00.0 VGA compatible controller [0300]: Advanced Micro Devices, Inc. [AMD/ATI] Ellesmere [Radeon Pro WX 5100] [1002:67c7]
43:00.1 Audio device [0403]: Advanced Micro Devices, Inc. [AMD/ATI] Ellesmere [Radeon RX 580] [1002:aaf0]
```

There are a few things to take note of in this output. The first is the fact
that there are multiple devices present on each slot. Any GPU with HDMI or
DisplayPort outputs will also have an audio device since those connections
carry audio in addition to video. Additionally RTX cards have a USB controller
for the USB type C port (intended for VR). The second thing is the device IDs,
which appear in brackets at the end of each line. Take note of the IDs for the
devices from your guest GPU and stash them into environment
variables:
```
VGAPT_VGA_ID='10de:1401'
VGAPT_VGA_AUDIO_ID='10de:0fba'
```

## Banish the guest GPU
In order to make use of the guest GPU in our virtual machine, we need to ensure
that the host operating system doesn't latch onto it. To avoid this, we bind the
vfio driver to the guest gpu at boot (using the IDs noted above):
```
$ echo options vfio-pci ids=$VGAPT_VGA_ID,$VGAPT_VGA_AUDIO_ID > /etc/modprobe.d/vfio.conf
$ printf "vfio\nvfio_pci\n" > /etc/modules-load.d/vfio.conf
$ update-initramfs -u
```

Reboot the host machine and then verify that the vfio driver is binding
properly. Using `lspci -v` aimed at the guest GPU will tell us which
driver is in use:
```
$ lspci -s 0a:00 -v
0a:00.0 VGA compatible controller: NVIDIA Corporation GP106 [GeForce GTX 1060 6GB] (rev a1) (prog-if 00 [VGA controller])
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

0a:00.1 Audio device: NVIDIA Corporation GP106 High Definition Audio Controller (rev a1)
	Subsystem: eVga.com. Corp. GP106 High Definition Audio Controller
	Flags: fast devsel, IRQ 146, NUMA node 0
	Memory at fb080000 (32-bit, non-prefetchable) [size=16K]
	Capabilities: <access denied>
	Kernel driver in use: vfio-pci
	Kernel modules: snd_hda_intel
```

The presence of `Kernel driver in use: vfio-pci` on both devices means the vfio
driver is binding as intended.

## Create a VM
Many of the guides available when I explored this process recommended manually
invoking qemu via a script which holds a complex set of flags and variables. I
was pleased to find that the GUI application virt-manager was perfectly capable
of handling most of the configuration.

### Basic setup
Create a virtual machine as usual. I may return to expand this later.

### Adding PCI devices
Adding PCI devices is fairly straightforward. While viewing the virtual
machine's properties:
* Click "Add Hardware"
* Select "PCI Host Device"
* Scroll and select the appropriate device (in my case `0000:0a:00.0` and `0000:0a:00.1`)

### Input and Output
With this configuration, the guest GPU will output to an attached monitor just
like a separate system. The easiest way to manage input to the virtual machine
is to plug in a second mouse and keyboard and pass them directly to the guest.

A more advanced approach is to use [evdev](https://passthroughpo.st/using-evdev-passthrough-seamless-vm-input/)
to route input data to the virtual machine. This works especially well in
conjunction with [Looking Glass](https://looking-glass.hostfission.com/). With
an HDMI or DisplayPort [dummy plug](https://www.amazon.com/gp/product/B077CZ6JC3/)
attached, the guest GPU will render frames to its frame buffer as normal. The
shared memory mapping and client binary provided by Looking Glass allows you to
directly read that frame buffer and interact with your guest inside of a window.

Finally the most robust method and the one that I eventually settled on would
be to use a hardware KVMP switch. I added a pci-e USB 3.0 card and passed it to
the virtual machine (binding the vfio driver to it and adding it in the
virt-manager GUI just as with the GPU). Any device plugged into that card or a
connected hub will hotplug and be detected by the vm. The KVMP switch is
connected to both GPUs, and to one USB port on the motherboard and one on the
VM's USB card. In terms of switching, the VM and host work as two separate
systems.

Audio output has its own complications. I decided right away that the virtual
sound device was unacceptable. It stuttered and glitched and was absolute
garbage. I highly recommend using a little USB
[sound device](https://www.amazon.com/Sabrent-External-Adapter-Windows-AU-MMSA/dp/B00IRVQ0F8/).

If you are using a KVMP switch and an HDMI or DisplayPort monitor with an audio
output, that can also be a convenient option.

### Code 43
Using an Nvidia card for the guest can involve an additional complication. In
order to protect the perceived value of their enterprise compute cards, their
consumer card drivers will panic if they detect that they are being used by a
virtual system. The result is a GPU driver crash with "code 43".

Fortunately the driver doesn't need to know (shhhhh). By editing the VM's
configuration XML directly and adding two options, I was able to convince the
Nvidia card that it was not living inside of a simulation.
```
$ virsh edit <vm name>
```

This will open the XML configuration for your VM in your default editor. the
two things you need to add fall within the XML topology like so:
```
<domain type='kvm' xmlns:qemu='http://libvirt.org/schemas/domain/qemu/1.0'>
...
  <features>
    <hyperv>
      <vendor_id state='on' value='123456789ab'/>
    </hyperv>
    <kvm>
      <hidden state='on'/>
    </kvm>
  </features>
...
</domain>
```

You may find that `<features>`. `<hyperv>`, and/or `<kvm>` already exist.
That's fine. Just be sure you have a `vendor_id` and `hidden`.

## More Later...
* First boot & Code 43
