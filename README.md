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
* BIOS settings
* Install host OS
* Install OVMF
* Install qemu
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
