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
  Physical volume "/dev/sdb" successfully created.\
[root@otus-task4 ~]# **vgcreate vg_root /dev/sdb**\
  Volume group "vg_root" successfully created\
[root@otus-task4 ~]# **lvcreate -n lv_root -l +100%FREE /dev/vg_root**\
  Logical volume "lv_root" created.
