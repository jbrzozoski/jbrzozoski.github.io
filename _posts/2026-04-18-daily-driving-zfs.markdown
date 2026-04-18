---
layout: post
title:  Daily driving a Linux laptop with ZFS root
date:   2026-04-18 00:01:00 -0500
categories: computers
---
I use ZFS for the root partition of my Linux laptop. It adds friction simply because it's more complex to setup and ZFS users are in the minority for non-server systems, so there's an increased risk of being on the "fringe".

However, for my usage, these are the pros of ZFS:

- I can snapshot my whole system before doing something "unsafe" and roll back to that snapshot from a rescue boot
- I can do incremental backups very efficiently and easily using snapshots and a remote ZFS file server
- The local drive can be encrypted such that the backups are unreadable on the backup server, without breaking incremental backup functionality
- I get extra ECC on all files to have higher confidence against bit rot
- I can "scrub" the drive which test reads everything

Last time I setup a laptop using ZFS, it was on an older version of Linux Mint which included optional tools in the installer to help make this happen. Those tools have been removed since they were used too little to be worth maintaining. I also didn't like their setup since it included zsys which is an overambitious attempt to allow a system to hold multiple full Linux installs within a ZFS heirarchy, auto-snapshot on every APT change, and more. I never used the multi-distro functions of zsys, and the auto-snapshot just tended to overflow my drives with unnecessary historical snapshots.

But I'm now at the point of wanting to reformat my laptop to a fresh install of Mint, and still want to keep ZFS. I'm fine with abandoning zsys, but still need to find some more help in setting up ZFS boot.

A currently popular tool in the ZFS space is (https://zfsbootmenu.org). It's nominal form is a Linux kernel and initrd packaged into a single UEFI file that you can install on the UEFI boot partition and your BIOS should be able to launch directly with no other understanding of file systems. The minimal ZfsBootMenu image will use the full Linux kernel capabilities to scan all ZFS pools it can find for things that match a certain expected pattern or boot/root filesystems. It then presents the user with a menu of found systems to let them pick one to attach and run. Finally, the ZfsBootMenu kernel will reattach the proper boot/root filesystems that the user selected, and reimage into the kernel provided in that boot image.

This is a relatively neat utility as it will let you have boot and root on arbitrarily complex zpools, including zraid and encryption. The problem is that it doesn't provide any easy installers. There are lengthy instructions to install it with more recent versions of Debian and Ubuntu, but those use `debootstrap` which is not workable with Mint from what I can tell.

I came up with an alternate method which can be summarize roughly as follows:

1. Run a complete Mint install using vanilla filesystem types and no encryption on a laptop
2. Before rebooting out of the installer, `tar` the full install you just made (boot and root) into on bit tarball on a USB drive
3. Wipe the laptop drive again
4. Reboot freshly into the Mint installer again
5. Follow most of the ZfsBootMenu instructions for prepping the drives for an [Ubuntu install](https://docs.zfsbootmenu.org/en/v3.1.x/guides/ubuntu/uefi.html), but stop before any steps that copy content onto the newly partitioned and mounted drives
6. Extract the tarball from the USB drive onto the newly partitioned and mounted drives
7. Conditionally follow steps to tweak the installed drives and ensure the boot menu and UEFI files are setup properly

This is not an "easy" process and requires evaluating quite a few steps from the ZfsBootMenu instructions for whether they apply or not, but it can work. I have made a Linux Mint install onto an encrypted ZFS boot drive using it. I intend to practice and refine the process a few more times before doing my main laptop, but it is planned...

