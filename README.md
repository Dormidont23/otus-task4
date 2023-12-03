##Задание №4.##
На имеющемся образе centos/7 - v. 1804.2
1. Уменьшить том под / до 8G
2. Выделить том под /home
3. Выделить том под /var - сделать в mirror
4. /home - сделать том для снапшотов
5. Прописать монтирование в fstab. Попробовать с разными опциями и разными файловыми системами (на выбор)
Работа со снапшотами:
- сгенерить файлы в /home/
- снять снапшот
- удалить часть файлов
- восстановится со снапшота
- залоггировать работу можно с помощью утилиты script

Подготовим временный том для корневого раздела /:\
[root@otus-task4 ~]# **pvcreate /dev/sdb**\
&emsp;Physical volume "/dev/sdb" successfully created.\
[root@otus-task4 ~]# **vgcreate vg_root /dev/sdb**\
&emsp;Volume group "vg_root" successfully created\
[root@otus-task4 ~]# **lvcreate -n lv_root -l +100%FREE /dev/vg_root**\
&emsp;Logical volume "lv_root" created.

Создадим на нем файловую систему и смонтируем его, чтобы перенести туда данные:
[root@otus-task4 ~]# mkfs.xfs /dev/vg_root/lv_root
meta-data=/dev/vg_root/lv_root   isize=512    agcount=4, agsize=655104 blks
         =                       sectsz=512   attr=2, projid32bit=1
         =                       crc=1        finobt=0, sparse=0
data     =                       bsize=4096   blocks=2620416, imaxpct=25
         =                       sunit=0      swidth=0 blks
naming   =version 2              bsize=4096   ascii-ci=0 ftype=1
log      =internal log           bsize=4096   blocks=2560, version=2
         =                       sectsz=512   sunit=0 blks, lazy-count=1
realtime =none                   extsz=4096   blocks=0, rtextents=0\
mount /dev/vg_root/lv_root /mnt

Этой командой скопируем все данные с / раздела в /mnt:\
[root@otus-task4 ~]# xfsdump -J - /dev/VolGroup00/LogVol00 | xfsrestore -J - /mnt

Затем переконфигурируем grub для того, чтобы при старте перейти в новый /. Сымитируем текущий root - сделаем в него chroot и обновим grub:\
[root@otus-task4 ~]# for i in /proc/ /sys/ /dev/ /run/ /boot/; do mount --bind $i /mnt/$i; done
[root@otus-task4 ~]# chroot /mnt/
[root@otus-task4 /]# grub2-mkconfig -o /boot/grub2/grub.cfg
Generating grub configuration file ...
Found linux image: /boot/vmlinuz-3.10.0-862.2.3.el7.x86_64
Found initrd image: /boot/initramfs-3.10.0-862.2.3.el7.x86_64.img
done

Обновим образ initrd.\
[root@otus-task4 boot]# cd /boot ; for i in `ls initramfs-*img`; do dracut -v $i `echo $i|sed "s/initramfs-//g; s/.img//g"` --force; done

