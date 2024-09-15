---
title: "Separating a Dual Boot Computer"
description: "Steps to take if you had installed dual boot Linux + Windows on the same drive, but now you want to separate them without clean install"
date: 2023-08-16T08:00:00
draft: False
tags: ["dev", "tutorial"]
---


This article will help you if:
- You did dual boot Linux + Windows in the same drive (different partition) and you are selecting them via GRUB
- You upgrade and bought a new storage. Now you want to separate them to different drives.
- You don't want to do clean install of each OS.

My specific case
- Both my old drive and new drive is an NVME SSD. Both are connected during the whole process.
- I'm using modern hardware ([UEFI with GPT ](https://www.howtogeek.com/193669/whats-the-difference-between-gpt-and-mbr-when-partitioning-a-drive/)instead of the old BIOS with MBR)
- I use dual boot Windows 10 and Ubuntu 20.04. I wish to keep Windows in the old SSD, and Ubuntu in the new SSD. This guide was written using this intention, so the end goal is that the old drive still contains Windows, and the new drive will contain Ubuntu.

Overview:
1. Clone your OS using [CloneZilla](https://clonezilla.org/downloads.php), [SystemRescue](https://www.system-rescue.org/) (used in this tutorial for [moving linux partition](https://help.ubuntu.com/community/MovingLinuxPartition)), or other means.
2. Now you'll have exact duplicates of your two OSs, with similar UUID. We generate unique ID for disks and partition
3. Using the unique UUID, we configure GRUB to show Windows partition from 1 drive and Ubuntu partition in the other drive.
4. After confirming correct OS when selected in GRUB, we delete the unwanted duplicates.


# Detailed Steps
I've written the detailed steps that I took (and briefly mentioning failed experiments). I do not guarantee this is the best approach for your system as you might have customized configuration or slightly different hardware.
### Identification
- You can view device assignments using `lsblk`
- If you have a harddisk or SSD with SATA connection, the device name is `/dev/sdXY` where `X` is the device index (first device is `a` next device is `b` and so on) and `Y` is the partition index. e.g. my 4TB hdd has the path `/dev/sdb` while the two partitions (each 2TB) that I made in that hdd are called `/dev/sdb1` and `/dev/sdb2`.
- If you have an NVME SSD the device name will start with  `nvme`. Mine says `/dev/nvmeXn1pY`, where `X` is the device index and `Y` is the partition index. 
- To view complete details of connected storage device, run `sudo lshw -C disk`.

### Ingredients
- A flash drive. Burn the CloneZilla OS using rufus and do step 1. next, burn a live Ubuntu OS
- time and patience (if it's your first time doing such)
## 1. Clone your OS
I used CloneZilla to do device-device clone of all partitions. There are many tutorials such as [this video](https://www.youtube.com/watch?v=2VwpmEA9pYQ) or [this article](https://www.tecmint.com/linux-centos-ubuntu-disk-cloning-backup-using-clonezilla/). The main parts are:
- Mode: `device-device`
- Choose `beginner` menu
- Choose `disk_to_local_disk` to clone the entire disk to another. In this menu you can probably do `part_to_local_part` which is a partition clone to reduce cloning time. 
- MOST IMPORTANT PART!! Do not mess this up. Choose the source which is your old drive.
- Then choose your destination which is the new drive. If you do the opposite, you'll end up wiping out your old drive.
- I chose to `Skip checking/repairing source file system` and to `Use the partition table from the source disk`
Cloning takes a while depending on how large your drive is. After that, there should be identical partitions on both drives. But when you restart and GRUB shows, you'll still see the same 2 options. This is because the clone disk have identical UUID, making the motherboard and GRUB confused. If you boot Windows and go to Disk Management, you'll see one of the disk to be offline with the info: `Offline (The disk is offline because it has a signature collision with another disk that is online)`. To view disk UUID and partition UUID in Ubuntu:
```
lsblk -o name,size,type,ptuuid,partuuid
```

![lsblk command for same uuid](/blog/dual-boot-in-different-drives/lsblk-same-uuid.jpg)
Figure 1. Similar UUID on both drives (nvme0n1 - old drive, nvme1n1 - new drive with the cloned partitions)
# 2. Generate unique UUID
Universally Unique Identifier (UUID) is now the [preferred method](https://help.ubuntu.com/community/UsingUUID) to identify drives because device assignments can change between system boot. To list all UUIDs of attached devices (mounted or not) use `sudo blkid`.

Our goal is:
- Create unique disk UUID, this will allow the motherboard to list both drives
- Create unique partition UUID (called PARTUUID) for EFI partition duplicates (one for windows in old drive, one for ubuntu in new drive).

To learn about UUID and PARTUUID in GPT-disks, refer to [this discussion](https://unix.stackexchange.com/questions/375548/what-is-uuid-partuuid-and-ptuuid/463410#463410).

You can do this manually, with each partition's different filesystem, e.g. for ext4 use `tune2fs`, for ntfs use `ntfslabel` and so on as [discussed here](https://unix.stackexchange.com/questions/12858/how-to-change-filesystem-uuid-2-same-uuid). Or, since our configuration is GPT, we can use `gdisk`. I recommend reading the manual using `man gdisk` to understand required partitions used by windows or ubuntu. Make note of the your new device's name (in my case it's `nvme0n1`, note: the disk name not the partition). Better to do this in Live Ubuntu as we'll follow it by some config changes. Use the following commands in the `gdisk` menu:
```
sudo gdisk /dev/nvme0n1

Partition table scan:
  MBR: protective
  BSD: not present
  APM: not present
  GPT: present

Found valid GPT with protective MBR; using GPT.

# enter expert command
Command (? for help): x

# randomize disk and partition unique GUIDs
Expert command (? for help): f

# go back to main menu
Expert command (? for help): m 

# write changes, will prompt "About to write GPT data, THIS WILL OVERWRITE EXISTING PARTITIONS!!"
Command (? for help): w
```

If we check again using `lsblk`, the disk and partition UUID are now different

![lsblk showing different uuid](/blog/dual-boot-in-different-drives/lsblk-diff-uuid.jpg)

Figure 2. Changed UUID on new drive

Actually, the suitable way of checking is using `sudo blkid` as we are using mixture of filesystem for our partitions. E.g. for the EFI partition, the UUID is shorter since it uses the vfat filesystem.

Now we need to edit the `fstab` file, which manages the mounting of partitions at boot time. If you're on Live Ubuntu, mount your new ubuntu partition:
```
sudo mount /dev/nvme0n1p4 /mnt
sudo nano /mnt/etc/fstab
```
Note, you should be editing `fstab` file of the Ubuntu on the new drive, on not `/etc/fstab`.

Now, inside there should be at least 2 UUIDs, one that mounts to root or `/`, and another that mounts to `/boot/efi` (again, assuming the system is UEFI based). These UUIDs should still point to the old partitions, and you simply need to replace them with the UUID of your new partitions (in the drive that will contain only Ubuntu). In the terminal, you can copy and paste using by right click, or `CTRL+Shift+C` and `CTRL+Shift+V`. After modifying, save the file using `CTRL+X`.


		Add Figure or text message of your fstab file

Figure Snippet of the changed `/etc/fstab` file. Note that the `efi` partition is `vfat` file system, so it has a shorter UUID.

# 3. Configure GRUB
Now we have unique UUIDs for all partitions, we need to edit GRUB, to list the Ubuntu from the new drive, and the Windows in the old drive. Only modifying `fstab` won't make GRUB recognize the correct devices. We will modify the `grub.cfg` file, which lists the boot options you see when you dual boot. Currently, they still point to the old partition UUIDs.
#### Reinstalling GRUB
Here's a tutorial for [reinstalling GRUB](https://help.ubuntu.com/community/Grub2/Installing#Reinstalling_GRUB_2), also [this answer](https://askubuntu.com/a/88432) gives step by step in case for dual boot. The main parts are:
- From a Live Ubuntu, mount the new drive's **partition** containing Ubuntu
```
sudo mount /dev/nvme0n1p4 /mnt
```
- For UEFI system, you also need to mount the EFI partition, otherwise it will return [the error](https://unix.stackexchange.com/questions/405472/cannot-find-efi-directory-issue-with-grub-install) `cannot find EFI directory`.
```
sudo mount /dev/nvme0n1p1 /mnt/boot/efi
```
- Now fix GRUB. Note the 2nd argument here is device name for the **drive**, not **partition**.
```
sudo grub-install --boot-directory=/mnt/boot /dev/nvme0n1
```
#### Alternative: Update manually
P.S. I tried this method first, but it didn't work. Leaving it here as a side note.
While still in your Live Ubuntu with the new Ubuntu partition still mounted:
```
sudo nano /mnt/boot/grub/grub.cfg
```

In `nano` press `CTRL+\`, paste (right click in terminal) your old UUID, hit enter, then paste new UUID, enter to start replace process (press `a` to replace all). There's also a UUID for the EFI partition under `menuentry` for Windows. We want GRUB to point to Windows in the old drive, so we'll leave this unchanged.

#### Understanding boot priority
From BIOS/UEFI, you can select boot priority. Now that you've create unique UUID and configured GRUB from Ubuntu in your new drive, you could boot to all 4 OSs. There'll be 2 `grub.cfg`, 1 from your Ubuntu partition in old drive, and the other from the Ubuntu partition in the new drive. You can switch to OS in old drive if you set the Ubuntu from old drive as boot priority, and vice-versa.

However, since our goal is to boot Windows from the old drive, we need to make one final modification.
- Edit the `grub.cfg` in the Ubuntu partition of new drive (refer to "Update manually section above") using `nano`
- Locate the `menuentry` to Windows (usually has the string "Boot to windows"). After reinstalling grub, this section should point to UUID of the EFI partition on your new drive.
- Simply change this to the UUID of EFI partition from your old drive.

#### Verify GRUB boots to intended drive
See if the GRUB menu options boot to correct drives:
- use `lsblk` and see where `/` and `/boot/efi` is for ubuntu
- open Disk Management for windows

# 4. Clean up
Now that we have GRUB listing the intended OS, we can remove the rest. Remember, we now have an EFI partition in each drive. You can use Live Ubuntu to clean up unwanted OS partitions from both drive. See the "Which partition does Ubuntu use" section below to identify correct partitions to delete.

After deleting, you will want to extend your system partition to take the unallocated space, however there's a little risk in moving your partition to the start of your drive, as if an error occurs, you might lost your data (not if you backup, but all this process itself). I took the risk and it went smoothly. You can read more about moving space from [gparted docs](https://gparted.org/display-doc.php%3Fname%3Dmoving-space-between-partitions).

![cleaning up your partitions](/blog/dual-boot-in-different-drives/cleaning-up.jpg)

Figure 3. Resizing/moving your system partition to occupy the whole space.
# What I learned
- if using UEFI, there'll be a partition containing EFI Partition System (EPS) that your motherboard will read.
- when dual boot, both windows and ubuntu will point to EFI in same partition. but they have different subfolders
- if you have mixed filesystem types, it's called "partition table". since you're using UEFI, it uses GUID Partition Table (GPT)

# Tangent

### What are the partition table for Ubuntu
If you have a new disk and not sure how to make partitions or what file systems are used for Ubuntu, [this answer](https://askubuntu.com/a/843649) gives detail step by step (assuming you're using UEFI system which uses GPT or GUID Partition Table). You can use `gparted` for this:
1. 1st partition / EFI. Select your empty drive and select "Make New Partition Table"
	- size: 500 MB
	- type for the new partition: Primary
	- use as: EFI
2. 2nd partition / Root. Select the remaining space and select "+"
	- size: at least 10 GB
	- type for the new partition: Primary
	- use as: ext4
	- mount point: choose "/"
There's also an option to add "swap" partition (if you're using the hibernate feature in Ubuntu) and adding another partition to hold your home directory. However, these [Discrete Partitions](https://wiki.archlinux.org/title/Partitioning#Discrete_partitions) are usually not needed unless you want to share them between multiple OS.

Other `gparted` configs:
- What is [LABEL](https://help.ubuntu.com/community/UsingUUID#:~:text=8134%2D0b2429c4c02c%20%2D%3E%20../../sda5-,Using%20LABEL,-Labels%20can%20be)? It is used to identify a partition, typically for flash drives or external HDD. If a device has a label, when mounted it will show at `/media/<label>`

### Which partition does Ubuntu use?
Run `lsblk`, the mounted partitions are used by Ubuntu, the rest are Windows'. Note that if you're using UEFI system, the partition with `/boot/efi` is used by both Ubuntu and Windows
```
nvme0n1     259:0    0 953,9G  0 disk
├─nvme0n1p1 259:1    0   100M  0 part /boot/efi
├─nvme0n1p2 259:2    0    16M  0 part
├─nvme0n1p3 259:3    0   326G  0 part
├─nvme0n1p4 259:5    0 627,3G  0 part /
└─nvme0n1p5 259:4    0   509M  0 part
```