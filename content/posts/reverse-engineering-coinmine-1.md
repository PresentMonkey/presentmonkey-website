---
title: "Reverse Engineering Coinmine One Part 1"
date: 2022-03-17
draft: false
categories: ["Hardware", "Reverse Engineering"]
---

So I received a broken Coinmine One and decided it would be a good project to try to reverse engineer how it works. Opening it up, everything was surprisingly standardized with very little proprietary. I don't have images of my own, so here is one from [cryptobrief.com](https://cryptobriefing.com/coinmine-one-waste-of-money-or-bitcoin-mining-innovation/). 

![coinmine internals!](/images/reverse-engineering-coinmine/internals.jpg) 

As you can see, it's just a small normal PC in a fancy case.

## Investigating

Using a SATA <-> USB adapter, I connected the hard drive, and to my surprise it was unencrypted. It consisted of 4 partitions as seen with `fdisk`

```none
Disk /dev/sdd: 931.53 GiB, 1000204886016 bytes, 1953525168 sectors
Disk model: 048-2E7172      
Units: sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes
Disklabel type: gpt
Disk identifier: 9F944706-4C90-43CC-A5BD-F1AFEE12D9C7

Device        Start      End  Sectors  Size Type
/dev/sdd1      2048    22527    20480   10M BIOS boot
/dev/sdd2     22528  1046527  1024000  500M EFI System
/dev/sdd3   1046528 27353087 26306560 12.6G Linux filesystem
/dev/sdd4  27353088 81233919 53880832 25.7G Linux filesystem
```

Mounting the 2 `ext4` partitions we can what the contents are.

The first one consists of your normal linux install with the exception of the `coinmine` directory which is empty, we'll get to that later.

```none
$ ls
bin  boot  coinmine  dev  etc  home  lib  lib64  lost+found  mnt  opt  proc  root  run  sbin  srv  sys  systemd  tmp  usr  var
```

The second one consisted of just 2 folders

```none
$ ls
Docker_Volumes  lost+found
```

Using `tree` in `Docker_Volumes revealed what it was for, holding data about the blockchain.

```none
$ tree . -d
.
└── Lightning
    ├── Mainnet
    │   └── bitcoind-mainnet-data
    │       ├── blocks
    │       ├── chainstate
    │       ├── database
    │       └── indexes
    │           └── txindex
    └── Simnet
```

This isn't too important for me, and it just takes a bunch of storage, so I won't be saving this partition to the disk image I'll be creating later.

Now, about the `/coinmine` directory, checking the `fstab` reveals that the 4th partition just gets mounted to that directory. Mystery solved.

```none
# <file system> <dir> <type> <options> <dump> <pass>
# UUID=b743e635-a3ef-4935-a1ba-bb84f90b16b1
UUID=b743e635-a3ef-4935-a1ba-bb84f90b16b1               /               ext4            rw,relatime     0 1

# UUID=6276-E51F
UUID=6276-E51F                  /boot           vfat            rw,relatime,fmask=0022,dmask=0022,codepage=437,iocharset=iso8859-1,shortname=mixed,utf8,errors=remount-ro     0 2

# UUID=86a03524-90b1-47d2-8759-57316eccf473
UUID=86a03524-90b1-47d2-8759-57316eccf473               /coinmine       ext4            rw,relatime     0 2
```

Going back to the main partition, I decided to check on the `home` folder.

```none
$ ls /home
builduser  satoshi
```

`builduser` is a user I assume is used to build something needed for the OS to work. I don't know exactly what it's for, it seems to build `yay`.

`satoshi` is more interesting:

```none
$ ls /home/satoshi/
config.yaml  docker-compose.prod.yml  docker-compose.yml  programs
```

Looking at the compose files, it's used to run their software, including mining, as the gpu gets passed into the container. It consists of 2 services `miner_updater` and `minary_api_server`.  They also seem to use ngrok for debugging, but this is the production docker-compose, weird. Looking at the `docker-compose` docs the `prod` file is used to say what changes should be made from the original compose file. This makes a lot more sense. Bellow are the 2 compose files linked as a github gist (they are pretty long):

* [docker-compose.prod.yml](https://gist.github.com/PresentMonkey/daf00d93ea02e55614d12a4d212d8d0e)

* [docker-compose.yml](https://gist.github.com/PresentMonkey/f8fe1af368db19d3ce4187f61f2d57d1)

Next I wanted to investigate what's inside the containers, but I found no easy way to do so, so I instead decided to just use a virtual machine and boot from a hard drive image.

## Creating disk image

The hard drive itself is 1TB in size so creating a disk image directly would be waste of space. What I decided to do, was to create images of the first 3 partitions and the combine them together to create a disk image. I did not include the last partition since it was large and just contained blockchain data. In addition to save space, I shrunk the partitions to make it easier to work with.

To start I used gnome disk utility, which allowed to me to create partition images easily from the gui. With that done, I was left with 3 images:

```none
coinmine_part_1.img
coinmine_part_2.img
coinmine_part_3.img
```

Now I needed to combine those into one image. I found [this](https://serverfault.com/questions/281628/combine-partitions-to-one-disk-image) StackExchange post which was exactly what I needed.

To start I created a sparse image with the appropriate size:

```none
$ dd if=/dev/zero of=coinmine.img bs=1 count=0 seek=16G
```

Next, created partitions using `fdisk`, with the appropriate size of partitions corresponding to the partition images we created with gnome disk utility, which resulted in:

```none
Disk coinmine.img: 16 GiB, 17179869184 bytes, 33554432 sectors
Units: sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes
Disklabel type: gpt
Disk identifier: 66406CAA-4B3D-764B-92BA-48BAE3A83022

Device          Start      End  Sectors  Size Type
coinmine.img1    2048    24575    22528   11M Linux filesystem
coinmine.img2   24576  1099775  1075200  525M Linux filesystem
coinmine.img3 1099776 30459903 29360128   14G Linux filesystem
```

Now, this is the magic part. Using `kpartx` we can create devices from the partitions, to which we can write to with `dd`.

```none
kpartx -av coinmine.img
```

```none
dd if=coinmine_part_1.img of=/dev/mapper/loop0p1 bs=1M
dd if=coinmine_part_2.img of=/dev/mapper/loop0p2 bs=1M
dd if=coinmine_part_3.img of=/dev/mapper/loop0p3 bs=1M
```

```none
kpartx -dv coinmine.img
```

And that's it. The disk image is created.

## Changing root password

With the disk image, there comes another issue. I don't know what the root password is on the image, so it needs to be changed. This can be done relatively easily however.

Using `kpartx` again, I can easily mount the 3rd partition. Next `chroot` can be used to change the root password:

```none
$ sudo chroot ./
# passwd
New password: 
Retype new password: 
passwd: password updated successfully
```

The reasoning behind how this works is a little complicated, but basically what it does is change what the apparent root location is. In the terminal now instead of `/` aka root being the root of my OS, instead it's changed to the image. Then the `passwd` command, it's referencing the image's `passwd` program, not my computer's, if that makes any sense.

## Make virtual machine

For the VM I'm using Virt Manager, a GUI frontend to QEMU/KVM. Setup is just like normal, it needs 4GB of memory and be in UEFI mode not legacy BIOS mode. When the VM starts though, we'll run into a bit of an issue. Because we never moved over the 4th partition, it's trying to mount it and failing. Therefore it drops us in emergency mode, where we can login as root and make the edits needed to `fstab`. All that's needed is adding `nofail` to the mount, so now it looks like this: 

```none
```none
# <file system> <dir> <type> <options> <dump> <pass>
# UUID=b743e635-a3ef-4935-a1ba-bb84f90b16b1
UUID=b743e635-a3ef-4935-a1ba-bb84f90b16b1               /               ext4            rw,relatime     0 1

# UUID=6276-E51F
UUID=6276-E51F                  /boot           vfat            rw,relatime,fmask=0022,dmask=0022,codepage=437,iocharset=iso8859-1,shortname=mixed,utf8,errors=remount-ro     0 2

# UUID=86a03524-90b1-47d2-8759-57316eccf473
UUID=86a03524-90b1-47d2-8759-57316eccf473               /coinmine       ext4            rw,relatime,nofail     0 2
```

And that's it for part 1. I'll go into a deeper dive into seeing how the software works in part 2. Thanks for reading.
