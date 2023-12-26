## Задание №4. Работа с LVM ##
На имеющемся образе centos/7 - v1804.2
1. Уменьшить том под / до 8G
2. Выделить том под /var - сделать в mirror
3. Выделить том под /home
4. /home - сделать том для снапшотов
5. Прописать монтирование в fstab.
6. Работа со снапшотами:\
&nbsp;- сгенерить файлы в /home/\
&nbsp;- снять снапшот\
&nbsp;- удалить часть файлов\
&nbsp;- восстановиться со снапшота
### Уменьшить том под / до 8G ###
Подготовка временного тома для корневого раздела /:\
[root@otus-task4 ~]# **pvcreate /dev/sdb**\
&emsp;Physical volume "/dev/sdb" successfully created.\
[root@otus-task4 ~]# **vgcreate vg_root /dev/sdb**\
&emsp;Volume group "vg_root" successfully created\
[root@otus-task4 ~]# **lvcreate -n lv_root -l +100%FREE /dev/vg_root**\
&emsp;Logical volume "lv_root" created.

Создать на нём файловую систему и смонтировать его, чтобы перенести туда данные:\
[root@otus-task4 ~]# **mkfs.xfs /dev/vg_root/lv_root**
```
meta-data=/dev/vg_root/lv_root   isize=512    agcount=4, agsize=655104 blks
         =                       sectsz=512   attr=2, projid32bit=1
         =                       crc=1        finobt=0, sparse=0
data     =                       bsize=4096   blocks=2620416, imaxpct=25
         =                       sunit=0      swidth=0 blks
naming   =version 2              bsize=4096   ascii-ci=0 ftype=1
log      =internal log           bsize=4096   blocks=2560, version=2
         =                       sectsz=512   sunit=0 blks, lazy-count=1
realtime =none                   extsz=4096   blocks=0, rtextents=0
```
[root@otus-task4 ~]# **mount /dev/vg_root/lv_root /mnt**\
Копировать все данные с / раздела в /mnt:\
[root@otus-task4 ~]# **xfsdump -J - /dev/VolGroup00/LogVol00 | xfsrestore -J - /mnt**\
xfsdump: using file dump (drive_simple) strategy\
xfsdump: version 3.1.7 (dump format 3.0)\
xfsrestore: using file dump (drive_simple) strategy\
xfsrestore: version 3.1.7 (dump format 3.0)\
xfsdump: level 0 dump of otus-task4:/\
xfsdump: dump date: Mon Dec  4 06:32:50 2023\
xfsdump: session id: 2ce7bdd3-35e1-4bab-b136-1e9f776ec918\
xfsdump: session label: ""\
xfsdump: ino map phase 1: constructing initial dump list\
xfsrestore: searching media for dump\
xfsdump: ino map phase 2: skipping (no pruning necessary)\
xfsdump: ino map phase 3: skipping (only one dump stream)\
xfsdump: ino map construction complete\
xfsdump: estimated dump size: 881017024 bytes\
xfsdump: creating dump session media file 0 (media 0, file 0)\
xfsdump: dumping ino map\
xfsdump: dumping directories\
xfsrestore: examining media file 0\
xfsrestore: dump description:\
xfsrestore: hostname: otus-task4\
xfsrestore: mount point: /\
xfsrestore: volume: /dev/mapper/VolGroup00-LogVol00\
xfsrestore: session time: Mon Dec  4 06:32:50 2023\
xfsrestore: level: 0\
xfsrestore: session label: ""\
xfsrestore: media label: ""\
xfsrestore: file system id: b60e9498-0baa-4d9f-90aa-069048217fee\
xfsrestore: session id: 2ce7bdd3-35e1-4bab-b136-1e9f776ec918\
xfsrestore: media id: b0e6ca4b-ab8e-4d06-b3d6-3f75d20d0a39\
xfsrestore: searching media for directory dump\
xfsrestore: reading directories\
xfsdump: dumping non-directory files\
xfsrestore: 2697 directories and 23603 entries processed\
xfsrestore: directory post-processing\
xfsrestore: restoring non-directory files\
xfsdump: ending media file\
xfsdump: media file size 858121032 bytes\
xfsdump: dump size (non-dir files) : 844967608 bytes\
xfsdump: dump complete: 27 seconds elapsed\
xfsdump: Dump Status: SUCCESS\
xfsrestore: restore complete: 27 seconds elapsed\
xfsrestore: Restore Status: SUCCESS

Затем переконфигурируем grub для того, чтобы при старте перейти в новый /. Сымитируем текущий root - сделаем в него chroot и обновим grub:\
[root@otus-task4 ~]# **for i in /proc/ /sys/ /dev/ /run/ /boot/; do mount --bind $i /mnt/$i; done**\
[root@otus-task4 ~]# **chroot /mnt/**\
[root@otus-task4 /]# **grub2-mkconfig -o /boot/grub2/grub.cfg**\
Generating grub configuration file ...\
Found linux image: /boot/vmlinuz-3.10.0-862.2.3.el7.x86_64\
Found initrd image: /boot/initramfs-3.10.0-862.2.3.el7.x86_64.img\
done

Обновить образ initrd.\
[root@otus-task4 boot]# **cd /boot ; for i in \`ls initramfs-\*img\`; do dracut -v $i \`echo $i|sed "s/initramfs-//g; s/.img//g"\` --force; done**\
...\
*** Hardlinking files done ***\
*** Stripping files ***\
*** Stripping files done ***\
*** Generating early-microcode cpio image contents ***\
*** Constructing AuthenticAMD.bin ****\
*** No early-microcode cpio image needed ***\
*** Store current command line parameters ***\
*** Creating image file ***\
*** Creating image file done ***\
*** Creating initramfs image file '/boot/initramfs-3.10.0-862.2.3.el7.x86_64.img' done ***

Ну и для того, чтобы при загрузке был смонтирован нужный root нужно в файле **/etc/default/grub** заменить **rd.lvm.lv=VolGroup00/LogVol00** на **rd.lvm.lv=vg_root/lv_root** и выполнить **grub2-mkconfig -o /boot/grub2/grub.cfg**\
Перезагружаемся успешно с новым рут томом. Убедиться в этом можно посмотрев вывод lsblk:\
[root@otus-task4 ~]# **lsblk**\
NAME                    MAJ:MIN RM  SIZE RO TYPE MOUNTPOINT\
sda                       8:0    0   40G  0 disk\
├─sda1                    8:1    0    1M  0 part\
├─sda2                    8:2    0    1G  0 part /boot\
└─sda3                    8:3    0   39G  0 part\
&emsp;&emsp;├─VolGroup00-LogVol01 253:1    0  1.5G  0 lvm  [SWAP]\
&emsp;&emsp;└─VolGroup00-LogVol00 253:2    0 37.5G  0 lvm\
**sdb                       8:16   0   10G  0 disk\
└─vg_root-lv_root       253:0    0   10G  0 lvm  /**\
sdc                       8:32   0    2G  0 disk\
sdd                       8:48   0    1G  0 disk\
sde                       8:64   0    1G  0 disk

Теперь нужно изменить размер старой VG и вернуть на неё рут. Для этого удаляем старый LV размером в 40G и создаем новый на 8G:\
[root@otus-task4 ~]# **lvremove /dev/VolGroup00/LogVol00**\
Do you really want to remove active logical volume VolGroup00/LogVol00? [y/n]: y\
  Logical volume "LogVol00" successfully removed\
[root@otus-task4 ~]# **lvcreate -n VolGroup00/LogVol00 -L 8G /dev/VolGroup00**\
WARNING: xfs signature detected on /dev/VolGroup00/LogVol00 at offset 0. Wipe it? [y/n]: y\
  Wiping xfs signature on /dev/VolGroup00/LogVol00.\
  Logical volume "LogVol00" created.

Проделываем на нем те же операции, что и в первый раз:\
[root@otus-task4 ~]# **mkfs.xfs /dev/VolGroup00/LogVol00**
```
meta-data=/dev/VolGroup00/LogVol00 isize=512    agcount=4, agsize=524288 blks
         =                       sectsz=512   attr=2, projid32bit=1
         =                       crc=1        finobt=0, sparse=0
data     =                       bsize=4096   blocks=2097152, imaxpct=25
         =                       sunit=0      swidth=0 blks
naming   =version 2              bsize=4096   ascii-ci=0 ftype=1
log      =internal log           bsize=4096   blocks=2560, version=2
         =                       sectsz=512   sunit=0 blks, lazy-count=1
realtime =none                   extsz=4096   blocks=0, rtextents=0
```
[root@otus-task4 ~]# **mount /dev/VolGroup00/LogVol00 /mnt**\
[root@otus-task4 ~]# **xfsdump -J - /dev/vg_root/lv_root | xfsrestore -J - /mnt**\
xfsdump: using file dump (drive_simple) strategy\
xfsrestore: using file dump (drive_simple) strategy\
xfsdump: version 3.1.7 (dump format 3.0)\
xfsrestore: version 3.1.7 (dump format 3.0)\
xfsdump: level 0 dump of otus-task4:/\
xfsdump: dump date: Mon Dec  4 07:24:27 2023\
xfsdump: session id: c9767a08-a563-4297-b43d-556b35b95bb9\
xfsdump: session label: ""\
xfsdump: ino map phase 1: constructing initial dump list\
xfsrestore: searching media for dump\
xfsdump: ino map phase 2: skipping (no pruning necessary)\
xfsdump: ino map phase 3: skipping (only one dump stream)\
xfsdump: ino map construction complete\
xfsdump: estimated dump size: 934619840 bytes\
xfsdump: creating dump session media file 0 (media 0, file 0)\
xfsdump: dumping ino map\
xfsdump: dumping directories\
xfsrestore: examining media file 0\
xfsrestore: dump description:\
xfsrestore: hostname: otus-task4\
xfsrestore: mount point: /\
xfsrestore: volume: /dev/mapper/vg_root-lv_root\
xfsrestore: session time: Mon Dec  4 07:24:27 2023\
xfsrestore: level: 0\
xfsrestore: session label: ""\
xfsrestore: media label: ""\
xfsrestore: file system id: df503f2c-e5ee-4ee5-aab4-272e5331fce3\
xfsrestore: session id: c9767a08-a563-4297-b43d-556b35b95bb9\
xfsrestore: media id: cc2fc868-34b0-40ad-9b4f-684d13199c36\
xfsrestore: searching media for directory dump\
xfsrestore: reading directories\
xfsdump: dumping non-directory files\
xfsrestore: 3125 directories and 26982 entries processed\
xfsrestore: directory post-processing\
xfsrestore: restoring non-directory files\
xfsdump: ending media file\
xfsdump: media file size 906734080 bytes\
xfsdump: dump size (non-dir files) : 891533400 bytes\
xfsdump: dump complete: 33 seconds elapsed\
xfsdump: Dump Status: SUCCESS\
xfsrestore: restore complete: 33 seconds elapsed\
xfsrestore: Restore Status: SUCCESS

Так же как в первый раз переконфигурируем grub, за исключением правки /etc/default/grub.\
[root@otus-task4 ~]# **for i in /proc/ /sys/ /dev/ /run/ /boot/; do mount --bind $i /mnt/$i; done**\
[root@otus-task4 ~]# **chroot /mnt/**\
[root@otus-task4 /]# **grub2-mkconfig -o /boot/grub2/grub.cfg**\
Generating grub configuration file ...\
Found linux image: /boot/vmlinuz-3.10.0-862.2.3.el7.x86_64\
Found initrd image: /boot/initramfs-3.10.0-862.2.3.el7.x86_64.img\
done\
[root@otus-task4 boot]# **cd /boot ; for i in \`ls initramfs-\*img\`; do dracut -v $i \`echo $i|sed "s/initramfs-//g; s/.img//g"\` --force; done**\
...\
*** Installing kernel module dependencies and firmware ***\
*** Installing kernel module dependencies and firmware done ***\
*** Resolving executable dependencies ***\
*** Resolving executable dependencies done ***\
*** Hardlinking files ***\
*** Hardlinking files done ***\
*** Stripping files ***\
*** Stripping files done ***\
*** Generating early-microcode cpio image contents ***\
*** Constructing AuthenticAMD.bin ****\
*** No early-microcode cpio image needed ***\
*** Store current command line parameters ***\
*** Creating image file ***\
*** Creating image file done ***\
*** Creating initramfs image file '/boot/initramfs-3.10.0-862.2.3.el7.x86_64.img' done ***

Пока не перезагружаемся и не выходим из-под chroot - мы можем заодно перенести /var
### Выделить том под /var в зеркало ###
На свободных дисках создаем зеркало:\
[root@otus-task4 boot]# **pvcreate /dev/sdc /dev/sdd**\
  Physical volume "/dev/sdc" successfully created.\
  Physical volume "/dev/sdd" successfully created.\
[root@otus-task4 boot]# **vgcreate vg_var /dev/sdc /dev/sdd**\
  Volume group "vg_var" successfully created\
[root@otus-task4 boot]# **lvcreate -L 950M -m1 -n lv_var vg_var**\
  Rounding up size to full physical extent 952.00 MiB\
  Logical volume "lv_var" created.

Создаем на нём ФС и перемещаем туда /var:\
[root@otus-task4 boot]# **mkfs.ext4 /dev/vg_var/lv_var**\
mke2fs 1.42.9 (28-Dec-2013)\
Filesystem label=\
OS type: Linux\
Block size=4096 (log=2)\
Fragment size=4096 (log=2)\
Stride=0 blocks, Stripe width=0 blocks\
60928 inodes, 243712 blocks\
12185 blocks (5.00%) reserved for the super user\
First data block=0\
Maximum filesystem blocks=249561088\
8 block groups\
32768 blocks per group, 32768 fragments per group\
7616 inodes per group\
Superblock backups stored on blocks:\
        32768, 98304, 163840, 229376

Allocating group tables: done\
Writing inode tables: done\
Creating journal (4096 blocks): done\
Writing superblocks and filesystem accounting information: done

[root@otus-task4 boot]# **mount /dev/vg_var/lv_var /mnt**\
[root@otus-task4 boot]# **cp -aR /var/\* /mnt/**\
[root@otus-task4 boot]# **rsync -avHPSAX /var/ /mnt/**\
sending incremental file list\
./\
.updated\
            163 100%    0.00kB/s    0:00:00 (xfr#1, ir-chk=1025/1027)\

sent 143,116 bytes  received 609 bytes  287,450.00 bytes/sec\
total size is 255,916,346  speedup is 1,780.60

На всякий случай сохраняем содержимое старого var:\
[root@otus-task4 boot]# **mkdir /tmp/oldvar && mv /var/\* /tmp/oldvar**\
Ну и монтируем новый var в каталог /var:\
[root@otus-task4 boot]# **umount /mnt**\
[root@otus-task4 boot]# **mount /dev/vg_var/lv_var /var**

Правим fstab для автоматического монтирования /var:\
[root@otus-task4 boot]# **echo "\`blkid | grep var: | awk '{print $2}'\` /var ext4 defaults 0 0" >> /etc/fstab**

После чего можно успешно перезагружаться в новый (уменьшенный root) и удалять временную Volume Group:\
[root@otus-task4 ~]# **lvremove /dev/vg_root/lv_root**\
Do you really want to remove active logical volume vg_root/lv_root? [y/n]: y\
  Logical volume "lv_root" successfully removed\
[root@otus-task4 ~]# **vgremove /dev/vg_root**\
  Volume group "vg_root" successfully removed\
[root@otus-task4 ~]# **pvremove /dev/sdb**\
  Labels on physical volume "/dev/sdb" successfully wiped.
### Выделить том под /home ###
**lvcreate -n LogVol_Home -L 2G /dev/VolGroup00**
