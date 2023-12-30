---
title: "Szyfrowanie całego dysku z Porteusem (no, prawie całego)"
date: 2023-12-30
publishDate: <[FillPublishDate]>
draft: false
tags: ['poradnik', 'porteus', 'szyfrowanie', 'luks', 'grub2', 'linux']
translationKey: PorteusFullDiskEncryption
---

## Disclaimer
Wszystkie dane na docelowym dysku będą usunięte! Nie zapomnij o zrobieniu kopii ważnych danych na inny dysk, jeśli takowe posiadasz. Nie odpowiadam za jakiekolwiek uszkodzenia ani utratę danych na pozostałych dyskach wynikające z nieuważnego używania wymienionych niżej programów bądź wybrania niewłaściwego dysku lub partycji.

## Co będzie potrzebne
- docelowy pendrive lub zewnętrzny dysk
- obraz płyty Porteusa
- dowolna dystrybucja Linuksa
- GRUB2 (syslinuxy, grub4dosy, itp. nie nadają się)
- cryptsetup
- fdisk lub inne narzędzie do partycjonowania
- jakiś edytor tekstu
- cpio i xz
- narzędzia do formatowania (fat32 i do wyboru ext4 lub xfs)
- dostęp do konta root
- trochę czasu i wiedzy

W Porteusie można zrobić moduł GRUBa wpisując ``pmod -m grub`` i aktywować go. Poza tym wszystkie w/w programy są w nim obecne.

## Przygotowywanie dysku
Dla jasności na tym przykładzie system będzie "instalowany" na pendrivie Sandisk Ultra USB 3.0 o pojemności 16 GB. Oczywiście nie namawiam nikogo do kupowania akurat tego pendriva, szczególnie o tej pojemności, bo potrafi mocno zamulać przy zapisie (nawet, gdy nie jest szyfrowany).

### Wybieranie dysku
Sprawdzamy, jak się przedstawia dysk, na którym chcemy postawić zaszyfrowanego Porteusa
```
root@porteus:/home/guest# fdisk -l /dev/sd?
Disk /dev/sda: 238.47 GiB, 256060514304 bytes, 500118192 sectors
Disk model: SSDPR-CX400-256-
Units: sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes
Disklabel type: dos
Disk identifier: 0x26dec7f0

Device     Boot  Start       End   Sectors   Size Id Type
/dev/sda1  *      2048    206847    204800   100M  7 HPFS/NTFS/exFAT
/dev/sda2       206848 500115455 499908608 238.4G  7 HPFS/NTFS/exFAT


Disk /dev/sdb: 3.95 GiB, 4237295616 bytes, 8275968 sectors
Disk model: Flash Disk      
Units: sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes
Disklabel type: dos
Disk identifier: 0x10140062

Device     Boot Start     End Sectors  Size Id Type
/dev/sdb1  *     2048 8275967 8273920  3.9G  c W95 FAT32 (LBA)


Disk /dev/sdc: 14.32 GiB, 15376000000 bytes, 30031250 sectors
Disk model: Ultra           
Units: sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes
Disklabel type: dos
Disk identifier: 0x00043561

Device     Boot Start      End  Sectors  Size Id Type
/dev/sdc1  *     2048 30031249 30029202 14.3G  c W95 FAT32 (LBA)
root@porteus:/home/guest# 
```

Jeśli nadal nie wiesz, jak się przedstawia, odłącz ten dysk, wpisz
```
root@porteus:/home/guest# dmesg -w
```
i podłącz go ponownie.

Po podłączeniu dysku powinno wyskoczyć coś takiego:
```
[*ciach*]
[  105.277225] usb 2-1: new high-speed USB device number 2 using ehci-pci
[  105.638838] usb 2-1: New USB device found, idVendor=0781, idProduct=5581, bcdDevice= 1.00
[  105.638850] usb 2-1: New USB device strings: Mfr=1, Product=2, SerialNumber=3
[  105.638854] usb 2-1: Product: Ultra
[  105.638856] usb 2-1: Manufacturer: SanDisk
[  105.638859] usb 2-1: SerialNumber: [*ciach*]
[  105.641655] usb-storage 2-1:1.0: USB Mass Storage device detected
[  105.644078] scsi host2: usb-storage 2-1:1.0
[  106.713581] scsi 2:0:0:0: Direct-Access     SanDisk  Ultra            1.00 PQ: 0 ANSI: 6
[  106.726900] sd 2:0:0:0: [sdc] 30031250 512-byte logical blocks: (15.4 GB/14.3 GiB)
[  106.739061] sd 2:0:0:0: [sdc] Write Protect is off
[  106.739069] sd 2:0:0:0: [sdc] Mode Sense: 43 00 00 00
[  106.757230] sd 2:0:0:0: [sdc] Write cache: disabled, read cache: enabled, doesn't support DPO or FUA
[  106.849243]  sdc: sdc1
[  106.849420] sd 2:0:0:0: [sdc] Attached SCSI removable disk
```
Widzimy, że przedstawia się jako ``/dev/sdc``. Oczywiście w poleceniach wskazujemy właściwy dysk, jeśli przedstawia się inaczej.

Sprawdzamy, czy partycje dysku ``/dev/sdc`` są zamontowane
```
root@porteus:/home/guest# mount | grep sdc
```
Jeśli nic nie wypisze, to znaczy, że nie są zamontowane, a więc możemy przejść dalej.

### Nadpisywanie poprzedniej zawartości dysku
UWAGA! JESZCZE RAZ, PRZED WYKONANIEM PONIŻSZEGO POLECENIA, SPRAWDŹ, CZY NA PEWNO WYBRAŁEŚ WŁAŚCIWY DYSK. WSZYSTKIE DANE NA WSKAZANYM DYSKU BĘDĄ NIEODWRACALNIE USUNIĘTE!!!
Można nadpisać cały dysk losowymi danymi poleceniem
```
root@porteus:/home/guest# dd if=/dev/urandom of=/dev/sdc bs=4M status=progress
```
Może to potrwać od kilku minut do kilku godzin.

### Zakładanie partycji
Odpalamy fdiska na tym dysku
```
root@porteus:/home/guest# fdisk /dev/sdc

Welcome to fdisk (util-linux 2.37.4).
Changes will remain in memory only, until you decide to write them.
Be careful before using the write command.


Command (m for help):
```

Zakładamy nową tablicę partycji
```
Command (m for help): o
Created a new DOS disklabel with disk identifier 0x703b1b52.
```

Zakładamy partycję na Porteusa, jednocześnie zostawiamy 32 MB za nią na partycję EFI (albo jak kto woli, ESP)
```
Command (m for help): n
Partition type
   p   primary (0 primary, 0 extended, 4 free)
   e   extended (container for logical partitions)
Select (default p): 

Using default response p.
Partition number (1-4, default 1): 
First sector (2048-30031249, default 2048): 
Last sector, +/-sectors or +/-size{K,M,G,T,P} (2048-30031249, default 30031249): -32M

Created a new partition 1 of type 'Linux' and of size 14.3 GiB.
```

Jeśli wyskoczy nam to
```
Partition #1 contains a vfat signature.

Do you want to remove the signature? [Y]es/[N]o: y

The signature will be removed by a write command.
```
to potwierdzamy.

Ustawiamy tą partycję jako bootowalną
```
Command (m for help): a
Selected partition 1
The bootable flag on partition 1 is enabled now.
```

Zakładamy partycję na bootloader dla UEFI
```
Command (m for help): n
Partition type
   p   primary (1 primary, 0 extended, 3 free)
   e   extended (container for logical partitions)
Select (default p): 

Using default response p.
Partition number (2-4, default 2): 
First sector (29966336-30031249, default 29966336): 
Last sector, +/-sectors or +/-size{K,M,G,T,P} (29966336-30031249, default 30031249): 

Created a new partition 2 of type 'Linux' and of size 31.7 MiB.

```

Zmieniamy jej typ na 'EFI (FAT-12/16/32)' (ef)
```
Command (m for help): t
Partition number (1,2, default 2): 
Hex code or alias (type L to list all): ef

Changed type of partition 'Linux' to 'EFI (FAT-12/16/32)'.
```

Sprawdzamy, czy układ partycji nam odpowiada, oraz - co najważniejsze - czy robimy to na właściwym dysku
```
Disk /dev/sdc: 14.32 GiB, 15376000000 bytes, 30031250 sectors
Disk model: Ultra           
Units: sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes
Disklabel type: dos
Disk identifier: 0x703b1b52

Device     Boot    Start      End  Sectors  Size Id Type
/dev/sdc1  *        2048 29966335 29964288 14.3G 83 Linux
/dev/sdc2       29966336 30031249    64914 31.7M ef EFI (FAT-12/16/32)

Filesystem/RAID signature on partition 1 will be wiped.
```

Jeśli wszystko nam pasuje, to zapisujemy zmiany
```
Command (m for help): w
The partition table has been altered.
Calling ioctl() to re-read partition table.
Syncing disks.

root@porteus:/home/guest#
```

### Formatowanie partycji
Formatujemy partycję EFI
```
root@porteus:/home/guest# mkfs.vfat /dev/sdc2 
mkfs.fat 4.2 (2021-01-31)
root@porteus:/home/guest#
```

i partycję z Porteusem (LUKS1 dlatego, że GRUB w BIOSie nie potrafi bootować z partycji LUKS2 mimo że powinien ją obsługiwać)
```
root@porteus:/home/guest# cryptsetup -y -s 512 --type=luks1 luksFormat /dev/sdc1

WARNING!
========
This will overwrite data on /dev/sdc1 irrevocably.

Are you sure? (Type 'yes' in capital letters): YES
Enter passphrase for /dev/sdc1: 
Verify passphrase: 
root@porteus:/home/guest#
```

Otwieramy zaszyfrowany kontener na partycji
```
root@porteus:/home/guest# cryptsetup luksOpen /dev/sdc1 crypt
Enter passphrase for /dev/sdc1: 
root@porteus:/home/guest#
```

Formatujemy wewnątrz zaszyfrowanego kontenera, w tym przypadku wybieramy system plików ext4
``` 
root@porteus:/home/guest# mkfs.ext4 /dev/mapper/crypt 
mke2fs 1.46.5 (30-Dec-2021)
64-bit filesystem support is not enabled.  The larger fields afforded by this feature enable full-strength checksumming.  Pass -O 64bit to rectify.
Creating filesystem with 3745024 4k blocks and 936560 inodes
Filesystem UUID: f13f291e-3184-425b-95bc-e9a202a5bdb1
Superblock backups stored on blocks: 
	32768, 98304, 163840, 229376, 294912, 819200, 884736, 1605632, 2654208

Allocating group tables: done                            
Writing inode tables: done                            
Creating journal (16384 blocks): done
Writing superblocks and filesystem accounting information: done   

root@porteus:/home/guest#
```

## Instalacja systemu i GRUBa
Montujemy partycje
```
root@porteus:/home/guest# mount /dev/mapper/crypt /mnt/dm-0
root@porteus:/home/guest# mount /dev/sdc2 /mnt/sdc2
```

Montujemy obraz Porteusa
```
root@porteus:/home/guest# mount Porteus-XFCE-v5.01-x86_64.iso /mnt/loop
mount: /mnt/loop: WARNING: source write-protected, mounted read-only.
```

i kopiujemy z niego wszystko na dysk
```
root@porteus:/home/guest# cp -r /mnt/loop/* /mnt/dm-0/
```

Obraz nam już nie będzie potrzebny, więc go odmontujemy
```
root@porteus:/home/guest# umount /mnt/loop/
```

Instalujemy GRUBa
```
root@porteus:/home/guest# export GRUB_ENABLE_CRYPTODISK=y
root@porteus:/home/guest# grub-install /dev/sdc --target=i386-pc --boot-directory=/mnt/dm-0/boot/
Installing for i386-pc platform.
Installation finished. No error reported.
root@porteus:/home/guest# grub-install --target=i386-efi --boot-directory=/mnt/dm-0/boot/ --efi-directory=/mnt/sdc2/ --removable
Installing for i386-efi platform.
Installation finished. No error reported.
root@porteus:/home/guest# grub-install --target=x86_64-efi --boot-directory=/mnt/dm-0/boot/ --efi-directory=/mnt/sdc2/ --removable
Installing for x86_64-efi platform.
Installation finished. No error reported.
root@porteus:/home/guest#
```

Nie zapominamy też o (co prawda bardzo skromnym, ale jednak) configu GRUBa
```
menuentry "Porteus" {
	linux /boot/syslinux/vmlinuz from=UUID:Tu-WsTaW-UuId-ZaSzYfRoWaNeGo-KoNtEnErA
	initrd /boot/syslinux/initrd.xz
}
```

UUID kontenera można wyciągnąć wpisując
```
root@porteus:/home/guest# blkid /dev/sdc1
/dev/sdc1: UUID="5b9b4bcd-cb8e-4858-a966-8dea23ec676e" TYPE="crypto_LUKS" PARTUUID="703b1b52-01"
```
NIE PODAJEMY UUIDA WEWNĄTRZ ZASZYFROWANEGO KONTENERA!!!
```
root@porteus:/home/guest# blkid /dev/mapper/crypt
/dev/mapper/crypt: UUID="f13f291e-3184-425b-95bc-e9a202a5bdb1" BLOCK_SIZE="4096" TYPE="ext4"
```
Zastępujemy nim ciąg ``Tu-WsTaW-UuId-ZaSzYfRoWaNeGo-KoNtEnErA`` i zapisujemy config w ``/mnt/dm-0/boot/grub/grub.cfg``.

## Modyfikacja ramdysku
GRUB teraz potrafi ładować jądro i obraz ramdysku z zaszyfrowanego kontenera na partycji (najpierw pytając o hasło do niego), ale Porteus nie potrafi go (jeszcze) rozpoznać, a co dopiero go otworzyć.

Tworzymy tymczasowy katalog i od razu do niego przechodzimy
```
root@porteus:/home/guest# cd $(mktemp -d)
root@porteus:/tmp/tmp.CVEW0IeOtV#
```

Rozpakowujemy tu obraz ramdysku
```
root@porteus:/tmp/tmp.CVEW0IeOtV# xz -dc /mnt/dm-0/boot/syslinux/initrd.xz | cpio -i
4827 blocks
root@porteus:/tmp/tmp.CVEW0IeOtV#
```

Zapisujemy tego patcha gdziekolwiek, np. w ``/tmp/patch``
```diff
diff --no-dereference -ruN 1/cleanup 2/cleanup
--- 1/cleanup	2023-12-23 15:47:29.297743669 +0100
+++ 2/cleanup	2023-12-23 16:49:16.302471492 +0100
@@ -72,17 +72,26 @@
 fi
 
 echo "unmounting everything else"
+# Close encrypted container file:
+if [ -b /dev/mapper/crypt ]; then
+    umount /dev/mapper/crypt
+    cryptsetup luksClose crypt
+    losetup -d /dev/loop2
+fi
 # RSC: workaround for "umount -a" that fails on recent kernels
-ALL=`cat /proc/mounts | cut -d" " -f2 | egrep -v "(tmpfs|/dev)" | tr "\n" " "`
+ALL=`cat /proc/mounts | cut -d" " -f2 | egrep -v "(tmpfs|/dev|/proc|/sys)" | tr "\n" " "`
 umount $ALL 2>/dev/null
-if [ $? -ne 0 ]; then
-    # Close encrypted container:
-    if [ -b /dev/mapper/crypt ]; then
-        cryptsetup luksClose crypt
-        losetup -d /dev/loop2
-    fi
-    umount -ar 2>/dev/null
+if [ $? != 0 ]; then
+    x=3
+    while [ $x -gt 0 ]; do
+        umount -art $(grep -v nodev /proc/filesystems | cut -f2 | tr '\n' ',') 2> /dev/null && break
+        sleep 0.5
+        let x=x-1
+    done
 fi
+# Close encrypted container partitions:
+[ -b /dev/mapper/changes ] && cryptsetup luksClose /dev/mapper/changes
+[ -b /dev/mapper/porteus ] && cryptsetup luksClose /dev/mapper/porteus
 
 # Eject cdrom device:
 if [ -z "`egrep -qo " noeject( |\$)" /proc/cmdline 2>/dev/null`" -a -b "$CD" ]; then
diff --no-dereference -ruN 1/etc/porteus/initrd.ver 2/etc/porteus/initrd.ver
--- 1/etc/porteus/initrd.ver	2023-12-23 15:47:29.297743669 +0100
+++ 2/etc/porteus/initrd.ver	2023-12-23 15:42:16.488611446 +0100
@@ -1 +1 @@
-initrd.xz:20230923
+initrd.xz:20231223-luks
diff --no-dereference -ruN 1/fatal 2/fatal
--- 1/fatal	2023-12-23 15:47:29.285742979 +0100
+++ 2/fatal	2023-12-09 14:21:16.959591379 +0100
@@ -1,5 +1,7 @@
 #!/bin/sh
 
+rm -rf /keys
+
 echo && echo
 echo "[1;36m""Porteus data not found.
 You are maybe using an unsupported boot device (eg. SCSI or old PCMCIA).
@@ -8,6 +10,9 @@
 Make sure that your boot parameters (cheatcodes) are correct.
 In case of booting over network - check if the driver for your NIC
 is included in initrd image.
+In case of boot disk encryption - make sure that encrypted disk is plugged in
+and the passphrase which you enter is correct. Check also if cryptsetup
+program and kernel modules such as dm-crypt and dm-mod exists in initrd image.
 
 Press space/enter to unmount all devices and reboot or any other key
 to drop to the debug shell.""[0m"
diff --no-dereference -ruN 1/finit 2/finit
--- 1/finit	2023-12-23 15:47:29.237740219 +0100
+++ 2/finit	2023-12-08 19:38:48.590680563 +0100
@@ -98,7 +98,7 @@
     [ -d /mnt/$DEV ] || mount_device $DEV
     [ $1 /mnt/$DEV/$LPTH ]
 elif [ $LPATH = UUID: -o $LPATH = LABEL ]; then
-    ID=`echo $2 | cut -d: -f2 | cut -d/ -f1`; LPTH=`echo $2 | cut -d/ -f2-`; DEV=`blkid | grep $ID | cut -d: -f1 | cut -d/ -f3`; SLEEP=6
+    ID=`echo $2 | cut -d: -f2 | cut -d/ -f1`; LPTH=`echo $2 | cut -d/ -f2-`; [ $LPATH$ID == $LPTH ] && LPTH=""; DEV=`blkid | grep $ID | cut -d: -f1 | cut -d/ -f3`; SLEEP=6
     while [ $SLEEP -gt 0 -a "$DEV" = "" ]; do nap; let SLEEP=SLEEP-1; fstab; DEV=`blkid | grep $ID | cut -d: -f1 | cut -d/ -f3`; done
     [ -d /mnt/$DEV ] || mount_device $DEV
     [ $1 /mnt/$DEV/$LPTH ]
@@ -109,6 +109,12 @@
 # Check if a location is writable
 is_writable(){ touch $1/.test 2>/dev/null; [ -e $1/.test ] && rm $1/.test; }
 
+# Check if partition is encrypted with LUKS
+is_luks(){ blkid $1 2> /dev/null | cut -d" " -f3- | grep -q crypto_LUKS; }
+
+# Load modules required for opening encrypted partition
+load_dm_modules(){ if [ ! $DMLOADED ]; then for x in dm_crypt cryptd cbc sha256_generic aes_generic aes_x86_64; do modprobe $x 2>/dev/null; done; DMLOADED=1; fi }
+
 # Booting failed. Failed to find porteus files. 
 fail() { echo -e $i"couldn't find $1. Correct your from= cheatcode.\n Documentation in /usr/doc/porteus. Press 'enter' to continue booting."; read -s; }
 
diff --no-dereference -ruN 1/linuxrc 2/linuxrc
--- 1/linuxrc	2023-12-23 15:47:29.217739070 +0100
+++ 2/linuxrc	2023-12-21 18:16:29.999777917 +0100
@@ -80,6 +80,22 @@
 		locate -e $FROM/porteus/$CFG
 		if [ $? -eq 0 ]; then
 			DIR=`echo $LPTH | rev | cut -d/ -f3- | rev`; [ $DIR ] && FOLDER=$DIR/porteus
+		elif is_luks /dev/$DEV; then
+			load_dm_modules
+			echo $i"found encrypted Porteus partition on /dev/$DEV"
+			if [ -f /keys/porteus ]; then
+				cryptsetup luksOpen /dev/$DEV porteus --key-file=/keys/porteus || cryptsetup luksOpen /dev/$DEV porteus
+			else
+				cryptsetup luksOpen /dev/$DEV porteus
+			fi
+			[ $? -ne 0 ] && { echo -e "${BOLD}${RED}Cannot open encrypted partition${RST}" && . fatal; }
+			LUKSUUID=`blkid /dev/mapper/porteus | cut -d '"' -f2`
+			LUKSDEV=`blkid /dev/dm-* | grep $LUKSUUID | cut -d: -f1 | cut -d/ -f3`
+			TRUEDEV=$DEV
+			param fsck && fsck_dat /dev/mapper/porteus
+			mount_device $LUKSDEV
+			DIR=`echo $LPTH | rev | cut -d/ -f3- | rev`; [ $DIR ] && FOLDER=$DIR/porteus
+			locate -e /mnt/$LUKSDEV/$DIR/porteus/$CFG || { echo -e "${BOLD}${RED}Porteus config not found on encrypted partition${RST}" && . fatal; }
 		else
 			echo -e "${YELLOW}from= cheatcode is incorrect, press enter to search through all devices${RST}"
 			read -s; search -e porteus/$CFG
@@ -90,6 +106,7 @@
 	CFGDEV=/mnt/$DEV
 fi
 
+rm -f /keys/porteus
 [ -e $CFGDEV/$FOLDER/$CFG ] && PTH=$CFGDEV/$FOLDER || . fatal
 
 # Set some variables to export as environment variables
@@ -119,17 +136,62 @@
 if [ $CHANGES ]; then
 	echo $i"setting up directory for changes"
 	CHNEXIT=`echo $CHANGES | cut -d: -f1`; [ $CHNEXIT = EXIT ] && CHANGES=`echo $CHANGES | cut -d: -f2-`
-	[ -r $CFGDEV/$CHANGES ] && { DEV=`echo $CFGDEV | sed s@/mnt/@@`; LPTH=$CHANGES; } || locate -r $CHANGES
+	[ -r $CFGDEV/$CHANGES ] && { DEV=`echo $CFGDEV | sed s@/mnt/@@`; LPTH=$CHANGES; } || locate -r $CHANGES || is_luks /dev/$DEV
 	if [ $? -eq 0 ]; then
 		if [ -d /mnt/$DEV/$LPTH ]; then
 			mkdir -p /mnt/$DEV/$LPTH/changes 2>/dev/null && \
 			mount -o bind /mnt/$DEV/$LPTH/changes /memory/changes && touch /memory/changes/._test1 2>/dev/null
+		elif is_luks /dev/$DEV; then
+			load_dm_modules
+			echo $i"found encrypted changes partition on /dev/$DEV"
+			if [ $DEV != "$TRUEDEV" ]; then
+				if [ -f /keys/changes ]; then
+					/opt/000-kernel/sbin/cryptsetup luksOpen /dev/$DEV changes --key-file=/keys/changes || /opt/000-kernel/sbin/cryptsetup luksOpen /dev/$DEV changes
+				else
+					/opt/000-kernel/sbin/cryptsetup luksOpen /dev/$DEV changes
+				fi
+				if [ $? -eq 0 ]; then
+					CHGLUKSUUID=`blkid /dev/mapper/changes | cut -d '"' -f2`
+					CHGLUKSDEV=`blkid /dev/dm-* | grep $CHGLUKSUUID | cut -d: -f1 | cut -d/ -f3`
+					CHGTRUEDEV=$DEV
+					param fsck && fsck_dat /dev/mapper/changes
+					mount_device $CHGLUKSDEV
+				fi
+			else
+				echo $i"encrypted changes partition and Porteus partition are the same"
+				CHGLUKSDEV=$LUKSDEV
+			fi
+			if [ "$CHGLUKSDEV" -a -d /mnt/$CHGLUKSDEV/$LPTH ]; then
+				mkdir -p /mnt/$CHGLUKSDEV/$LPTH/changes 2>/dev/null && \
+				mount -o bind /mnt/$CHGLUKSDEV/$LPTH/changes /memory/changes && touch /memory/changes/._test1 2>/dev/null
+			elif [ "$CHGLUKSDEV" -a -f /mnt/$CHGLUKSDEV/$LPTH ]; then
+				if is_luks /mnt/$CHGLUKSDEV/$LPTH; then
+					losetup /dev/loop2 /mnt/$CHGLUKSDEV/$LPTH
+					echo $i"found encrypted .dat container"
+					if [ -f /keys/changesdat ]; then
+						/opt/000-kernel/sbin/cryptsetup luksOpen /dev/loop2 crypt --key-file=/keys/changesdat || /opt/000-kernel/sbin/cryptsetup luksOpen /dev/loop2 crypt
+					else
+						/opt/000-kernel/sbin/cryptsetup luksOpen /dev/loop2 crypt
+					fi
+					fsck_dat /dev/mapper/crypt
+					mount /dev/mapper/crypt /memory/changes 2>/dev/null && touch /memory/changes/._test1 2>/dev/null
+				else
+					fsck_dat /mnt/$CHGLUKSDEV/$LPTH
+					mount -o loop /mnt/$CHGLUKSDEV/$LPTH /memory/changes 2>/dev/null && touch /memory/changes/._test1 2>/dev/null
+				fi
+			else
+				CHGERR=3; fail $CHANGES; fail_chn; false
+			fi
 		else
-			if blkid /mnt/$DEV/$LPTH 2>/dev/null | cut -d" " -f3- | grep -q _LUKS; then
-				for x in dm_crypt cryptd cbc sha256_generic aes_generic aes_x86_64; do modprobe $x 2>/dev/null; done
+			if is_luks /mnt/$DEV/$LPTH; then
+				load_dm_modules
 				losetup /dev/loop2 /mnt/$DEV/$LPTH
 				echo $i"found encrypted .dat container"
-				/opt/000-kernel/sbin/cryptsetup luksOpen /dev/loop2 crypt
+				if [ -f /keys/changesdat ]; then
+					/opt/000-kernel/sbin/cryptsetup luksOpen /dev/loop2 crypt --key-file=/keys/changesdat || /opt/000-kernel/sbin/cryptsetup luksOpen /dev/loop2 crypt
+				else
+					/opt/000-kernel/sbin/cryptsetup luksOpen /dev/loop2 crypt
+				fi
 				fsck_dat /dev/mapper/crypt
 				mount /dev/mapper/crypt /memory/changes 2>/dev/null && touch /memory/changes/._test1 2>/dev/null
 			else
@@ -146,7 +208,7 @@
 				echo "boot will continue in '[1;36mAlways Fresh[0m' mode for this session"
 				sleep 10; CHGERR=1; rmdir /mnt/$DEV/$LPTH/changes; fail_chn
 			else
-				echo $i"filesystem is posix compatible"; CHNDEV=/mnt/$DEV
+				echo $i"filesystem is posix compatible"; [ $CHGLUKSDEV ] && CHNDEV=/mnt/$CHGLUKSDEV || CHNDEV=/mnt/$DEV
 				rmdir /memory/changes/mnt/* 2>/dev/null
 				rm -rf /memory/changes/var/lock/* /var/run/laptop-mode-tools/* /var/spool/cron/cron.??????
 				for x in `find /memory/changes/var/run -name "*pid" 2>/dev/null`; do rm $x; done
@@ -160,7 +222,7 @@
 					#chown -R guest /memory/changes/home/guest 2>/dev/null
 				fi
 			fi
-		else
+		elif [ ! $CHGERR ]; then
 			echo $i"changes not writable, using memory instead"; CHGERR=2; umount /memory/changes 2>/dev/null; fail_chn
 		fi
 	else
@@ -170,6 +232,7 @@
 	 echo $i"changes cheatcode not found, using memory only"; fail_chn
 fi
 
+rm -rf /keys
 mkdir -p /memory/changes/mnt/live
 
 debug

```

i nakładamy go na rozpakowany ramdysk
```
root@porteus:/tmp/tmp.CVEW0IeOtV# patch -p1 < /tmp/patch 
patching file etc/porteus/initrd.ver
patching file fatal
patching file finit
patching file linuxrc
root@porteus:/tmp/tmp.CVEW0IeOtV#
```

Nadpisujemy stary obraz ramdysku nowym
```
root@porteus:/tmp/tmp.CVEW0IeOtV# find . | cpio -H newc -o | xz --check=crc32 --x86 --lzma2 > /mnt/dm-0/boot/syslinux/initrd.xz
4833 blocks
root@porteus:/tmp/tmp.CVEW0IeOtV#
```

## Tworzenie drugiego obrazu ramdysku (na wymagane moduły jądra i program do otwierania zaszyfrowanego kontenera)
Co prawda Porteus już umie rozpoznać zaszyfrowany kontener na partycji, ale jeszcze nie potrafi go otworzyć, bo nie ma programu ``cryptsetup``, który by to mógł zrobić. Znajduje się on w module ``000-kernel`` i jest wykorzystywany do otwierania zaszyfrowanego obrazu kontenera ze zmianami na nieszyfrowanej partycji. W tym przypadku by dostać się do ``000-kernel``, w którym jest program ``cryptsetup``, trzeba najpierw otworzyć kontener, a bez tego programu nie możemy tego zrobić. Takie błędne koło.

A więc tworzymy kolejny tymczasowy katalog i znowu od razu do niego przechodzimy
```
root@porteus:/tmp/tmp.CVEW0IeOtV# cd $(mktemp -d)
root@porteus:/tmp/tmp.Xk1JROw4CT#
```

Montujemy obraz ``000-kernel.xzm``
```
root@porteus:/tmp/tmp.Xk1JROw4CT# mkdir kernel
root@porteus:/tmp/tmp.Xk1JROw4CT# mount /mnt/dm-0/porteus/base/000-kernel.xzm kernel
```

Ustawiamy wersję jądra w zmiennej ``kver``, będziemy się do niej odwoływać kilka razy
```
root@porteus:/tmp/tmp.Xk1JROw4CT# kver=$(basename $(find kernel/lib/modules/ -type d -name "*-porteus"))
```

Kopiujemy ``cryptsetup``
```
root@porteus:/tmp/tmp.Xk1JROw4CT# mkdir bin
root@porteus:/tmp/tmp.Xk1JROw4CT# cp kernel/sbin/cryptsetup bin
```

Zapisujemy listę modułów jądra potrzebnych do otwarcia kontenera do pliku
```
root@porteus:/tmp/tmp.Xk1JROw4CT# for mod in `find kernel/lib/modules/$kver/kernel/{crypto,arch/x86/crypto,drivers/md} -name "*.ko"`; do grep `basename $mod`: kernel/lib/modules/$kver/modules.dep | sed s/:// | tr " " "\n" >> mod; done
```

Kopiujemy moduły z listy do tymczasowego katalogu, potem przenosimy je we właściwe miejsce
```
root@porteus:/tmp/tmp.Xk1JROw4CT# mkdir out
root@porteus:/tmp/tmp.Xk1JROw4CT# for x in `sort -u mod`; do cp -a --parents kernel/lib/modules/$kver/$x out; done
root@porteus:/tmp/tmp.Xk1JROw4CT# mv out/kernel/lib .
```

Odmontowujemy ``000-kernel.xzm``
```
root@porteus:/tmp/tmp.Xk1JROw4CT# umount kernel
```

Sprzątamy
```
root@porteus:/tmp/tmp.Xk1JROw4CT# rmdir out/kernel out kernel
root@porteus:/tmp/tmp.Xk1JROw4CT# rm mod
```

Tworzymy listę zależności modułów jądra
```
root@porteus:/tmp/tmp.Xk1JROw4CT# depmod -b . $kver
```

Te ostrzeżenia można zignorować
```
depmod: WARNING: could not open modules.order at /tmp/tmp.Xk1JROw4CT/./lib/modules/6.5.5-porteus: No such file or directory
depmod: WARNING: could not open modules.builtin at /tmp/tmp.Xk1JROw4CT/./lib/modules/6.5.5-porteus: No such file or directory
```

Tworzymy drugi obraz ramdysku
```
root@porteus:/tmp/tmp.Xk1JROw4CT# find . | cpio -H newc -o | xz --check=crc32 --x86 --lzma2 > /mnt/dm-0/boot/syslinux/initrd-crypt.xz
17020 blocks
root@porteus:/tmp/tmp.Xk1JROw4CT#
```

Nie zapominamy o dopisaniu nowego obrazu do linijki initrd w configu gruba
```
menuentry "Porteus" {
	linux /boot/syslinux/vmlinuz from=UUID:Tu-WsTaW-UuId-ZaSzYfRoWaNeGo-KoNtEnErA
	initrd /boot/syslinux/initrd.xz /boot/syslinux/initrd-crypt.xz
}
```

## Fajrant
Restartujemy komputer i sprawdzamy czy się nasz zaszyfrowany system uruchomi.

Na tym można by było zakończyć ten tutorial, ale...

## Ale to jeszcze nie wszystko
### Gdy się nie chce pierdyliard razy klepać haseł do kontenerów
Tworzymy jeszcze jeden tymczasowy katalog i jeszcze raz od razu do niego przechodzimy
```
root@porteus:/home/guest# cd $(mktemp -d)
root@porteus:/tmp/tmp.EFiZQAcvZ9#
```

Tworzymy tu katalog ``keys`` i do niego wchodzimy
```
root@porteus:/tmp/tmp.EFiZQAcvZ9# mkdir keys
root@porteus:/tmp/tmp.EFiZQAcvZ9# cd keys
root@porteus:/tmp/tmp.EFiZQAcvZ9/keys#
```

Tworzymy tu nowe pliki o określonych nazwach, które będą służyć do otwarcia kontenerów:
- porteus - partycja z Porteusem
- changes - partycja ze zmianami (nie musimy tworzyć, jeśli będą na tej samej partycji co Porteus albo będzie nieszyfrowana)
- changesdat - szyfrowany plik ze zmianami (nie musimy tworzyć, jeśli ma nie być szyfrowany)

Generujemy plik z losowymi danymi
```
root@porteus:/tmp/tmp.EFiZQAcvZ9/keys# dd if=/dev/random of=porteus bs=1K count=1
1+0 records in
1+0 records out
1024 bytes (1.0 kB, 1.0 KiB) copied, 0.000816108 s, 1.3 MB/s
root@porteus:/tmp/tmp.EFiZQAcvZ9/keys# 
```

Ale zanim dodamy nowy klucz, sprawdzimy, ile kluczy mamy dodanych do kontenera
```
root@porteus:/tmp/tmp.EFiZQAcvZ9/keys# cryptsetup luksDump /dev/sdc1
LUKS header information for /dev/sdc1

Version:       	1
Cipher name:   	aes
Cipher mode:   	xts-plain64
Hash spec:     	sha256
Payload offset:	4096
MK bits:       	512
MK digest:     	da 10 74 58 fe 70 88 32 e0 9a a6 8b ad 1a a1 66 55 05 a4 5c 
MK salt:       	74 17 8c d5 53 2c 2d bf da 67 9a c3 ff bd 22 42 
               	e9 3e 3a b3 97 fb 05 34 ae b2 15 16 fb c4 79 b7 
MK iterations: 	31326
UUID:          	5b9b4bcd-cb8e-4858-a966-8dea23ec676e

Key Slot 0: ENABLED
	Iterations:         	501230
	Salt:               	98 f7 f1 11 06 7e fa cc 5c 17 ff 98 26 7b b0 46 
	                      	13 1b 4b 4b 01 08 4a ac 32 1b 4f 9f 4a 03 9e 20 
	Key material offset:	8
	AF stripes:            	4000
Key Slot 1: DISABLED
Key Slot 2: DISABLED
Key Slot 3: DISABLED
Key Slot 4: DISABLED
Key Slot 5: DISABLED
Key Slot 6: DISABLED
Key Slot 7: DISABLED
root@porteus:/tmp/tmp.EFiZQAcvZ9/keys#
```
Jak widać, mamy dodany tylko jeden klucz.

Dodajemy drugi klucz do kontenera
```
root@porteus:/tmp/tmp.EFiZQAcvZ9/keys# cryptsetup luksAddKey /dev/sdc1 porteus 
Enter any existing passphrase: 
root@porteus:/tmp/tmp.EFiZQAcvZ9/keys#
```

i jeszcze raz sprawdzamy
```
root@porteus:/tmp/tmp.EFiZQAcvZ9/keys# cryptsetup luksDump /dev/sdc1
LUKS header information for /dev/sdc1

Version:       	1
Cipher name:   	aes
Cipher mode:   	xts-plain64
Hash spec:     	sha256
Payload offset:	4096
MK bits:       	512
MK digest:     	da 10 74 58 fe 70 88 32 e0 9a a6 8b ad 1a a1 66 55 05 a4 5c 
MK salt:       	74 17 8c d5 53 2c 2d bf da 67 9a c3 ff bd 22 42 
               	e9 3e 3a b3 97 fb 05 34 ae b2 15 16 fb c4 79 b7 
MK iterations: 	31326
UUID:          	5b9b4bcd-cb8e-4858-a966-8dea23ec676e

Key Slot 0: ENABLED
	Iterations:         	501230
	Salt:               	98 f7 f1 11 06 7e fa cc 5c 17 ff 98 26 7b b0 46 
	                      	13 1b 4b 4b 01 08 4a ac 32 1b 4f 9f 4a 03 9e 20 
	Key material offset:	8
	AF stripes:            	4000
Key Slot 1: ENABLED
	Iterations:         	499320
	Salt:               	d1 74 5a a8 f8 e4 aa d6 f3 03 cf 14 ca 80 87 5b 
	                      	27 28 ea 75 60 40 a7 4b c9 83 de 7e e0 07 5b 3e 
	Key material offset:	512
	AF stripes:            	4000
Key Slot 2: DISABLED
Key Slot 3: DISABLED
Key Slot 4: DISABLED
Key Slot 5: DISABLED
Key Slot 6: DISABLED
Key Slot 7: DISABLED
root@porteus:/tmp/tmp.EFiZQAcvZ9/keys#
```
No i teraz już mamy dodane dwa klucze, którymi można otworzyć zaszyfrowany kontener (tym pierwszym jest oczywiście hasło).

Postępujemy analogicznie w przypadku pozostałych kontenerów, które chcemy mieć zaszyfrowane, ale nie chce nam się wpisywać do nich haseł.

Wychodzimy z tego katalogu i tworzymy kolejny (ostatni już) obraz ramdysku
```
root@porteus:/tmp/tmp.EFiZQAcvZ9/keys# cd ..
root@porteus:/tmp/tmp.EFiZQAcvZ9# find . | cpio -H newc -o | xz --check=crc32 --x86 --lzma2 > /mnt/dm-0/boot/syslinux/initrd-keys.xz
3 blocks
root@porteus:/tmp/tmp.EFiZQAcvZ9#
```

Nie zapominamy o uprawnieniach do tego obrazu - to bardzo ważne, by inni użyszkodnicy poza rootem nie mieli dostępu do kluczy
```
root@porteus:/tmp/tmp.EFiZQAcvZ9# chmod 400 /mnt/dm-0/boot/syslinux/initrd-keys.xz 
```

I jeszcze dopiszemy nowy obraz do linijki initrd w configu gruba
```
menuentry "Porteus" {
	linux /boot/syslinux/vmlinuz from=UUID:Tu-WsTaW-UuId-ZaSzYfRoWaNeGo-KoNtEnErA
	initrd /boot/syslinux/initrd.xz /boot/syslinux/initrd-crypt.xz /boot/syslinux/initrd-keys.xz
}
```

Gotowe, teraz już wystarczy tylko raz w GRUBie wpisać hasło.

Uwagi:
- jeśli do ramdysku został dodany nieprawidłowy klucz, cryptsetup zapyta o hasło
- pliki z kluczami będą kasowane z ramdysku po momentach, w których mogły być użyte, niezależnie od ich poprawności i tego, czy zostały użyte, oraz po nieznalezieniu danych Porteusa
- tylko trzy na twardo wklepane w/w zaszyfrowane kontenery mogą być otwarte podczas startu, przed załadowaniem właściwego systemu

### Gdy trzeba zaktualizować jądro
Przed restartem należy uruchomić poniższy skrypt, w którym jako pierwszy argument podajemy ścieżkę do ``000-kernel.xzm``, a jako drugi miejsce, gdzie obraz ramdysku będzie zapisywany, np. ``/mnt/dm-0/boot/syslinux/initrd-crypt.xz`` (UWAGA: zastępuje wskazany plik jeśli istnieje).
```bash
#!/bin/bash

if [ -z $1 ] || [ -z $2 ]; then echo "usage: $0 path/to/000-kernel.xzm path/to/output/initrd-crypt.xz"; exit 1; fi

kernelxzm="$(realpath $1)"
output="$(realpath $2)"

if [ $(basename "$kernelxzm") != "000-kernel.xzm" ]; then echo "invalid file selected"; exit 3; fi
if [ ! -f "$kernelxzm" ]; then echo "000-kernel.xzm not found"; exit 2; fi

set -e

temp=$(mktemp -d)
echo "tmp folder is $temp"
cd $temp

mkdir kernel
mount "$kernelxzm" kernel
kver=$(basename $(find kernel/lib/modules/ -type d -name "*-porteus"))

mkdir bin
cp kernel/sbin/cryptsetup bin

for mod in `find kernel/lib/modules/$kver/kernel/{crypto,arch/x86/crypto,drivers/md} -name "*.ko"`; do
    grep `basename $mod`: kernel/lib/modules/$kver/modules.dep | sed s/:// | tr " " "\n" >> mod
done

mkdir out
for x in `sort -u mod`; do cp -a --parents kernel/lib/modules/$kver/$x out; done
rm mod

umount kernel
rmdir kernel

mv out/kernel/lib .
rmdir out/kernel out
depmod -b . $kver

find . | cpio -H newc -o | xz --check=crc32 --x86 --lzma2 > "$output"

cd ..
echo "cleanup tmp"
rm -rf $temp
```

Tak samo, jak wcześniej, te ostrzeżenia możemy zignorować
```
depmod: WARNING: could not open modules.order at /tmp/tmp.75ZjuSk7Nj/./lib/modules/6.6.3-porteus: No such file or directory
depmod: WARNING: could not open modules.builtin at /tmp/tmp.75ZjuSk7Nj/./lib/modules/6.6.3-porteus: No such file or directory
```

### Jeśli chodzi o parametr fsck
Jeśli parametr ``fsck`` został ustawiony, sprawdzana jest również spójność systemów plików w otwartych zaszyfrowanych kontenerach. 

### Jeśli chodzi o parametry extramod i rootcopy
Na chwilę obecną ``extramod`` działa tylko w obrębie zaszyfrowanego kontenera na partycji Porteusa (``extramod=/coś/tam``) oraz niezaszyfrowanych partycji. Podobnie chyba jest z ``rootcopy``, choć nie sprawdzałem tego.
