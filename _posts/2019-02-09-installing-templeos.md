---
title: How to install TempleOS on a PC without optical drive
layout: post
category: blog
---

Anyone that wanted to try TempleOS on actual hardware might have discovered
that the official image will only work when booted from a real ATA-attached
CD-ROM drive, which not many machines, especially notebooks have these days.

If your machine still has BIOS-boot with an ATA-compatible harddisk controller,
there is still a way to achieve this.

I'm writing up a step-by-step guide on how i achieved that on a Lenoveo Ideapad S205,
and follow up with some background information i found while reading up on this.

## Prerequisites

Not every PC is adequate for booting TempleOS.
The PC must have some "oldschool" compatibilty layers in place.
These include:

- Ability for BIOS-based, e.g. non-UEFI boot
- IDE-Emulation option for SATA drives

Also you need a bootable USB-Stick with a Live Linux Distro.

## Booting up Linux

So fire up your target machine with livestick. In my case i used the
Debian Live image.

We need qemu-kvm installed, and obviously also have the TempleOS.ISO ready somewhere.

But in the first step, we have to find the I/O Port Numbers of our harddisk controller.

To You might have to install "pciutils" to get lspci.
When run as root, it should show an entry like this:

```
# lspci -v
[...]
00:11.0 SATA controller: Advanced Micro Devices, Inc. [AMD/ATI] SB7x0/SB8x0/SB9x0 SATA Controller [AHCI mode] (prog-if 01 [AHCI 1.0])
	Subsystem: Lenovo SB7x0/SB8x0/SB9x0 SATA Controller [AHCI mode]
	Flags: bus master, 66MHz, medium devsel, latency 64, IRQ 19
	I/O ports at 2118 [size=8]
	I/O ports at 2124 [size=4]
	I/O ports at 2110 [size=8]
	I/O ports at 2120 [size=4]
	I/O ports at 2100 [size=16]
	Memory at f024a000 (32-bit, non-prefetchable) [size=1K]
	Capabilities: [70] SATA HBA v1.0
	Capabilities: [a4] PCI Advanced Features
	Kernel driver in use: ahci
	Kernel modules: ahci
```

There are several I/O-Ports listed, of which i'm not sure what they do,
but i was successfull by just using the first two, 0x2118 and 0x2124.

## Install TempleOS

After installing qemu-kvm, we need to boot it from the TempleOS.ISO, passing in the real harddisk and enough RAM:

kvm -hda /dev/sda -cdrom TempleOS.ISO -boot d -m 512

It should boot right into the installer. When it asks if you're running
inside a VM, you just hit yes and it will automatically install to the harddrive.
It will automatically install it, and set up the bootloader to use those port numbers inside the VM.

So after a few minutes the installation is done, and the installer asks to reboot.
Quit the VM and remove the "-boot d" part, so it boots from the harddrive.

It should boot right up and start "Uncompressing dictionaries", which again will take a moment.
Kindly deny the tour, and find yourself in the TempleOS command prompt.

What we need to do know, is run the kernel installation routine, to fix up
those port number.
Type 'BootHDIns;', go through the prompts until it asks for those I/O port numbers.
Use those we found above, in hexadecimal form, in our case 0x2118 and 0x2124.
Everything else can be kept at default.

After that, you can 'Reboot;', but that won't work inside the VM anymore.
Now it's time to reboot the machine and try it on the metal.

Your computer should now either be booting TempleOS, or it doesn't. Good luck!


# Background

## ATA Ports

TempleOS' creator Terry went for the lowest common denominator in
implementing harddrive support: The ATA PIO compatibility layer.
This involves poking at certain I/O-Registers, like 0x01f0 for the primary ATA controller. [1]

The installer will probe only probe certain fixed locations of these ports,
that happen to match those provided by default in Qemu or other VM software.

Without relying on these legacy methods, driver complexity increases quickly.
[WhyNotMore.DD](https://templeos.holyc.xyz/Wb/Doc/WhyNotMore.html) goes into more detail about the driver limitations.


## BIOS Boot

The usual boot process on a x86 PC works like this:
The BIOS reads the first sector (512 bytes, a.K.a. MBR) of the boot device and loads it to a specific address.
This sector contains code that has the responsibility to load futher sectors,
and load subsequent parts of the bootloader, until that in itself is capable of booting the OS.

Remember that in this early stage, the boot code can't really keep an eye on every possible
media and the according drivers, that might be very complex.
For this reason, the BIOS implements simple "drivers" for all supported boot methods,
and makes simple access method available through the infamous [Interrupt 0x13](https://en.wikipedia.org/wiki/INT_13H)
This might include drivers for ATA controllers, SATA's AHCI, a complete USB-Stack etc.
If you're interested in the details, you can take a look behind the curtain in the open (SeaBIOS)[https://github.com/coreboot/seabios]

## Real-Mode

So if the BIOS has all these drivers, why can't TempleOS make use of that?
The reason is, the utilities provided by the BIOS are only available in real mode.

After leaving real mode for protected mode or long mode (TempleOS is a 64-bit only OS as you might know),
the OS or Bootloader usually provides the necessary drivers.
it would require temporarily switching back to real mode in some way or another.
OSDev even dedicated [a full article]() to advise against relying on BIOS methods.

Another option might be to load the full USB Image to RAM, and present it in TempleOS as a Ramdisk, which is already supported.
The issue here is, real-mode can only address 640k of memory (20 Bit Address == 1 MiB in theory).
TempleOS already needs to take care that it can fit the first stage into those 640k.

## Proposal

In theory, it's possible to reach higher addresses in real-mode with some trickery (the so-called (Unreal Mode)[https://wiki.osdev.org/Unreal_Mode]).

Considering that there are lots of modern-but-not-too-recent machines around
with lots of RAM and no optical drive, it might be possible to modify the bootloader
so it loads the whole image into the B:/ Ramdisk through Unreal Mode,
and then continue booting from there.

This way we could get a USB-bootable TempleOS image, without actually implementing
a whole USB Mass Storage stack.


<!--
installed from Linux Live Stick in KVM

- Same boot error in DrvChk
- SATA controller is not available on 0x01f0
    - Setting SATA controller to "IDE" in BIOS doesn't help
- Different I/O-Ports in lspci -v:
  On ThinkPad:
        I/O ports at 3080 [size=8]
        I/O ports at 3090 [size=4]
        I/O ports at 3088 [size=8]
        I/O ports at 3094 [size=4]
        I/O ports at 3060 [size=32]

    Different offsets, but same port sizes!

- Booting KVM with AHCI device:
    -drive id=disk,format=raw,file=/dev/sda,if=none -device ahci,id=ahci -device ide-drive,drive=disk,bus=ahci.0

Can't install TempleOS on different machine with different I/O-Ports!
How to change IO-Ports (base0, base1) in installed image?

? ATAProbe? -> not called

BlkDevAdd(offset=0, bd->base0=0x01f0)
BlkDevInit? -> 0
ATAInit?
ATAReadNativeMax?

kernel_cfg->add_dev ??
    CKCfg
!-->


BootHDIns -> I/O port   2118    2124


# TL;DR

1) Install TempleOS on a HD in a VM
2) Find your ATA I/O ports with lspci -v
3) Boot the VM from the installed drive
4) Run `BootHDIns;`, and type in your native I/O ports when prompted


[1](https://wiki.osdev.org/I/O_Ports)
