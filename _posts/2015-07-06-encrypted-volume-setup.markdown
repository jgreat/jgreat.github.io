---
layout: post
title:  "Encrypted Volume Setup"
date:   2015-07-06 13:33:17 -0600
categories: linux ubuntu btrfs luks
---

Based on blog post:
<a href="http://thesimplecomputer.info/full-disk-encryption-with-ubuntu">http://thesimplecomputer.info/full-disk-encryption-with-ubuntu</a>

### Notes
Remember when it cost hundreds of thousands of dollars for a few TB of storage? Well I just bought a pair of 4TB disk for about $250, WE LIVE IN THE FUTURE!!!

Okay, I want to encrypt the data I'm storing. I currently have my home encrypted using ecryptfs (i.e. "Encrypt my Home" in Ubuntu Setup). This is fine for home, but I have services that I want to run while not logged in. So I'm going to use btrfs on top of a pair of LUKS devices.

## Encryption

### Prep the Devices? - Nope.
There is a lot of advice as a best practice to prep the devices by writing noise with /dev/urandom or AES ciphertext (as suggested above), but on big devices like my 4TB drive it would take 11+ hours to complete. I'm not paranoid enough about this data to wait.

The down side is that someone could figure out the bounds of the data, That I have 1.6TB used instead of the full 4TB.

Anyways I can always fill the rest of the drive with noise after the fact with /dev/zero. <a href="http://security.stackexchange.com/questions/29682/remedy-for-not-having-filled-the-disk-with-random-data">http://security.stackexchange.com/questions/29682/remedy-for-not-having-filled-the-disk-with-random-data</a>

### Benchmark.
Which encryption option are best:

```bash
root@caesar:/# cryptsetup benchmark
# Tests are approximate using memory only (no storage IO).
PBKDF2-sha1       789590 iterations per second
PBKDF2-sha256     451972 iterations per second
PBKDF2-sha512     362077 iterations per second
PBKDF2-ripemd160  562540 iterations per second
PBKDF2-whirlpool  167183 iterations per second
#  Algorithm | Key |  Encryption |  Decryption
     aes-cbc   128b   166.0 MiB/s   186.3 MiB/s
 serpent-cbc   128b    81.7 MiB/s   215.1 MiB/s
 twofish-cbc   128b   196.9 MiB/s   242.1 MiB/s
     aes-cbc   256b   129.7 MiB/s   135.9 MiB/s
 serpent-cbc   256b    88.0 MiB/s   215.8 MiB/s
 twofish-cbc   256b   189.4 MiB/s   230.1 MiB/s
     aes-xts   256b   178.0 MiB/s   181.4 MiB/s
 serpent-xts   256b   204.0 MiB/s   200.6 MiB/s
 twofish-xts   256b   224.4 MiB/s   213.0 MiB/s
     aes-xts   512b   133.9 MiB/s   142.0 MiB/s
 serpent-xts   512b   189.2 MiB/s   204.3 MiB/s
 twofish-xts   512b   221.1 MiB/s   224.1 MiB/s
```

**Hash**

Its my understanding that sha1 as a hash method is not really recommended anymore. sha512 isn't that much less efficient then sha256 on my CPU so sha512 it is.

**Algorithm**

twofish-xts performs the highest, but I was a but surprised that the 256 vs 512 bit performance pretty much negligible. twofish-xts-plain64 512bit is the choice.

### Create the partitions.
/dev/sdb and /dev/sdc are my devices.

Use gparted to create a gpt partition table and create a primary partition using the whole disk.

### Create the encrypted device.

Repeat for each device.

```
root@caesar:/# cryptsetup luksFormat --cipher twofish-xts-plain64 --key-size 512 --hash sha512 --iter-time 1000 /dev/sdc1

WARNING!
========
This will overwrite data on /dev/sdc1 irrevocably.

Are you sure? (Type uppercase yes): YES
Enter passphrase:
Verify passphrase:
```

### Open the encrypted devices

Make them available as /dev/mapper devices.

```
root@caesar:/# cryptsetup luksOpen /dev/sdc1 data-01
Enter passphrase for /dev/sdc1:
root@caesar:/# cryptsetup luksOpen /dev/sdd1 data-02
Enter passphrase for /dev/sdd1:
```

### Get UUIDs for devices

We need this to setup crypttab to mount at boot.

```
root@caesar:~# cryptsetup luksUUID /dev/sdc1
fafc39b8-204e-4bb4-b7d8-808abb8ba53f
root@caesar:~# cryptsetup luksUUID /dev/sdd1
f08efefc-2882-445a-969f-b4e0c6c99c70
```

### /etc/crypttab

The first field is the mapper device name. Second is the UUID.

```
data-01 UUID=fafc39b8-204e-4bb4-b7d8-808abb8ba53f
data-02 UUID=f08efefc-2882-445a-969f-b4e0c6c99c70
```

### Back up the LUKS headers
Backup the headers for the devices to a file somewhere encrypted. If your header gets corrupted this is the only way you may be able to recover the data.

```
cryptsetup luksHeaderBackup /dev/sdc1 --header-backup-file /home/jgreat/sdc1-data-01.img
cryptsetup luksHeaderBackup /dev/sdd1 --header-backup-file /home/jgreat/sdd1-data-01.img
```

## Create your file system
You can now use the /dev/mapper devices to make the filesystem/volume management of your choice. I'm creating a set of btrfs mirrors.

### btrfs mirror

**Create the mirror**

```
root@caesar:/# mkfs.btrfs -d raid1 -m raid1 /dev/mapper/data-01 /dev/mapper/data-02
Btrfs v3.17
See http://btrfs.wiki.kernel.org for more information.

Turning ON incompat feature 'extref': increased hardlink limit per file to 65536
adding device /dev/mapper/data-02 id 2
fs created label (null) on /dev/mapper/data-01
    nodesize 16384 leafsize 16384 sectorsize 4096 size 7.28TiB
```

**Show the results**

```
root@caesar:/# btrfs fi show
Label: none  uuid: 6adadbe1-9194-4cda-abe0-5779b305aa76
    Total devices 2 FS bytes used 640.00KiB
    devid    1 size 3.64TiB used 2.03GiB path /dev/mapper/data-01
    devid    2 size 3.64TiB used 2.01GiB path /dev/mapper/data-02

Btrfs v3.17
```

**Mount the devices**

Use the UUID or one of the /dev paths.

```
root@caesar:/# mount UUID=6adadbe1-9194-4cda-abe0-5779b305aa76 /data
```

**Balance btrfs**
You may want to balance the devices since they will show a slight mismatch.

```
root@caesar:/# btrfs balance /data
Done, had to relocate 6 out of 6 chunks
```

**Create a subvolume**

I'm going to create a subvolume.

```
root@caesar:/# btrfs sub create /data/media
Create subvolume '/data/media'
```

**/etc/fstab**

Add an entry to fstab so we can mount on boot

```
UUID=6adadbe1-9194-4cda-abe0-5779b305aa76 /data  btrfs   defaults 0 2
```
