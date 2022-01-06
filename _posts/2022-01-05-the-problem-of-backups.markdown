---
layout: post
title:  "The problem of backups..."
date:   2022-01-05 00:01:00 -0500
categories: proxmox-backup-server proxmox-ve computers
---
# What's the problem?

One of the first issues I wanted to tackle as part of extracting myself from "The Cloud" was having reasonable backups of my files. Running a copy of NextCloud to backup my phone is easy, and making manual local backups from NextCloud is easy, but that's not enough.

Manual backups *will* fail to happen on occassion. It's just the way things are.

On-site backups will not help if my house burns down or some other similarly catastrophic event takes out all copies.

I'm a bit on the data hoarder side of things and have about 3TB of files to backup, so paying an online backup servive gets expensive in monthly costs.

# Enter Proxmox Backup Server (PBS)

I've been a fan of ProxmoxVE (PVE) and using it for my personal servers for a while now. When I saw that Proxmox were coming out with a highly integrated backup solution, I thought that could possibly fit all of my needs. It hit several checkboxes that I really liked:

- I can run it on my own hardware at a relative's house or under my desk at work (offsite and no monthly cost!)
- Encrypted backups (key only stored on PVE side)
- Incremental backups (only needs to send changed data over the wire)
- Smart integration with PVE such that VMs and containers are backed up atomically with configs and drives as a unit

I picked up a used PC with reasonable specs off eBay, loaded it up with PBS, and started experimenting. I know I'm going to have to play with VPNs a little bit to get my PVE and PBS servers to see each other once they're at different locations, but that's a problem for future me.

The basics of PBS worked fantastically! (I hope to make a blog post walking through the basic steps soon...) PBS can make backups of any containers (LXC) or VM with almost no interruption, and it really does a fantastic job on disk-based VMs or smaller file-based LXCs.

However, I discovered that while the amount of data sent over the wire really is just the changed files from file-based backups, it can still take a very long time. When PVE goes to backup a file-based store it connects to PBS, gets a manifest from PBS of all the hashes of the files as they existed at the last backup, recalculates the hashes of all files on it's local store, then sends just files with changed hashes to PBS.

The probem is that "recalculates the hashes of all files" part. When it hits my 4TB of data on rotating media HDD, it takes many hours to read and hash everything.

This long scan time fails for me since I was hoping to make PBS only have to be online briefly each day, so as to keep it more secure (offline and powered down) and in case I ended up putting it at my relative's house it wouldn't be making noise or taking up power all day long. Reading all that data is also excessive load and wear on the PVE server.

# Working around the slow backup

You can mark individual file-stores within PVE as "do not backup". So, my current plan is to mark the really big file stores in that manner, and back them up separately using a more efficient method. I'll still use the built-in PVE/PBS integration for most LXC and VM, including all block-stores and smaller file-stores.

Thankfully, I have my larger file-stores on ZFS, and my PBS server will be using ZFS as well. My new plan is to script a ZFS-based snapshot and incremental send between the two servers to keep the big file-stores in sync. They will require more manual setup and monitoring, but should hopefully be just as automatic once working.

This does have the downside you can't encrypt just the remote backups. You need to encrypt the local ZFS also.

# Encrypting specific datasets in ZFS under PVE

This is [semi-documented at the PVE wiki](https://pve.proxmox.com/wiki/ZFS_on_Linux#zfs_encryption), but that site doesn't get into some of the issues you will bump into and how to solve them.

First up, encrypted datasets can't always be moved around between PVE nodes using the normal methods. You may have to do some manual shutdown and migration from the command line, possibly even manual ZFS send/receive commands. Not an issue for my normal usage since I mostly live on one PVE node.

Second, the docs don't provide guidance on how to load keys at boot. I decided to just use a local key file since I wanted PVE to be able to boot when I wasn't around, and was more concerned with the remote backups being encrypted.

Here's how I set it up:

Create a new key file (just 32 bytes of random data):

```
dd if=/dev/urandom of=/root/edata.key bs=32 count=1
chmod 400 /root/edata.key
```

Create a new encrypted dataset using that key:

```
zfs create -o encryption=on -o keyformat=raw -o keylocation=file:///root/edata.key dpool/edata
```

Create a new service to load the key at boot (you can use multiple ExecStart lines if you need more calls to load-key):

```
cat > /etc/systemd/system/zfs-load-key.service <<EOF
[Unit]
Description=Load encryption keys
DefaultDependencies=false
Before=zfs-mount.service
After=zfs-import.target

[Service]
Type=oneshot
RemainAfterExit=yes
ExecStart=/usr/sbin/zfs load-key dpool/edata

[Install]
WantedBy=zfs-mount.service
EOF

systemctl enable zfs-load-key.service
```

Tell PVE's storage subsystem about the new datastore area:

```
pvesm add zfspool enc-zfs --pool dpool/edata --content images,rootdir --mountpoint /dpool/edata --nodes om --sparse 1
```

At this point I can use the usual methods in PVE to create or move file-stores or block-stores on the new `enc-zfs` area.

