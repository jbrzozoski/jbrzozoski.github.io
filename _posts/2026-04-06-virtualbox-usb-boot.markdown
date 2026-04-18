---
layout: post
title:  How to boot from a USB device in Virtualbox w/o excessive priviliges
date:   2026-04-06 00:01:00 -0500
categories: computers
---
I wanted to setup a Windows 11 machine in VirtualBox using the tweaks that Rufus provides when it generates the USB installer. I had Rufus make a USB drive, then proceeded to try and figure out how to get VirtualBox to use it.

I found that there are ways to point VirtualBox at real physical devices, but I found that actually using those drives in virtualbox required running at the same permissions as needed for raw access to the drive. That makes sense, but I didn't feel like running virtualbox as root nor did I feel like mangling the USB drive permissions temporarily. 

After digging a bit more, I found a way to convert a raw disk image into a virtualbox VDI. The first time I did this I hadn't zeroed out the USB first, and the image was the full size of the USB drive, which is much bigger than it needed to be.

To reduce the size, I zeroed out the whole USB drive, then remade it into a Win11 installer with Rufus, then ran the following command to make it into a VDI:

```
sudo vboxmanage convertfromraw /dev/sda win11_rufus.vdi --format=VDI
```

This time, the image was much smaller (only the size of the actual data and not the whole USB drive)

Then I just changed the ownership of the VDI file so that my normal user could access it, and attached it to a new VM.

After installing Win11, I detached the VDI from the VM and all is well.
