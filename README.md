# Домашнее задание к занятию "3.5. Файловые системы"

## Узнайте о sparse (разряженных) файлах.

Разреженные файлы - специальные файлы, которые хранят пустую информация в виде нулей в блоке метаданных ФС, это позволяет экономить физическое место диска.

## Могут ли файлы, являющиеся жесткой ссылкой на один объект, иметь разные права доступа и владельца? Почему?

Нет немогут, т.к. жесткая ссылка это по факту ссылка на один и тот же файл, а права доступа и владелец хранятся в inode

## Сделайте vagrant destroy на имеющийся инстанс Ubuntu. Замените содержимое Vagrantfile следующим

После приминения настроек появились дополнительно два неразмеченных диска

     vagrant@sysadm-fs:~$ lsblk
     NAME                      MAJ:MIN RM  SIZE RO TYPE MOUNTPOINT
     loop0                       7:0    0 67.2M  1 loop /snap/lxd/21835
     loop1                       7:1    0 43.6M  1 loop /snap/snapd/14978
     loop2                       7:2    0 61.9M  1 loop /snap/core20/1328
     sda                         8:0    0   64G  0 disk
     ├─sda1                      8:1    0    1M  0 part
     ├─sda2                      8:2    0  1.5G  0 part /boot
     └─sda3                      8:3    0 62.5G  0 part
       └─ubuntu--vg-ubuntu--lv 253:0    0 31.3G  0 lvm  /
     sdb                         8:16   0  2.5G  0 disk
     sdc                         8:32   0  2.5G  0 disk

## Используя fdisk, разбейте первый диск на 2 раздела: 2 Гб, оставшееся пространство.

     vagrant@sysadm-fs:~$ sudo fdisk /dev/sdb
     
     Welcome to fdisk (util-linux 2.34).
     Changes will remain in memory only, until you decide to write them.
     Be careful before using the write command.
     
     Device does not contain a recognized partition table.
     Created a new DOS disklabel with disk identifier 0x14d5d317.
     
     Command (m for help): n
     Partition type
        p   primary (0 primary, 0 extended, 4 free)
        e   extended (container for logical partitions)
     Select (default p):
     
     Using default response p.
     Partition number (1-4, default 1):
     First sector (2048-5242879, default 2048):
     Last sector, +/-sectors or +/-size{K,M,G,T,P} (2048-5242879, default 5242879): +2G
     
     Created a new partition 1 of type 'Linux' and of size 2 GiB.

     Command (m for help): n
     Partition type
        p   primary (1 primary, 0 extended, 3 free)
        e   extended (container for logical partitions)
     Select (default p):
     
     Using default response p.
     Partition number (2-4, default 2):
     First sector (4196352-5242879, default 4196352):
     Last sector, +/-sectors or +/-size{K,M,G,T,P} (4196352-5242879, default 5242879):
     
     Created a new partition 2 of type 'Linux' and of size 511 MiB.

     Command (m for help): w
     The partition table has been altered.
     Calling ioctl() to re-read partition table.
     Syncing disks.

## Используя sfdisk, перенесите данную таблицу разделов на второй диск.

     vagrant@sysadm-fs:~$ sudo sfdisk -d /dev/sdb > ~/sdb_part.dump
     vagrant@sysadm-fs:~$ ls
     sdb_part.dump
     vagrant@sysadm-fs:~$ sudo sfdisk /dev/sdc < ~/sdb_part.dump
     Checking that no-one is using this disk right now ... OK
     
     Disk /dev/sdc: 2.51 GiB, 2684354560 bytes, 5242880 sectors
     Disk model: VBOX HARDDISK
     Units: sectors of 1 * 512 = 512 bytes
     Sector size (logical/physical): 512 bytes / 512 bytes
     I/O size (minimum/optimal): 512 bytes / 512 bytes
     
     >>> Script header accepted.
     >>> Script header accepted.
     >>> Script header accepted.
     >>> Script header accepted.
     >>> Created a new DOS disklabel with disk identifier 0x14d5d317.
     /dev/sdc1: Created a new partition 1 of type 'Linux' and of size 2 GiB.
     /dev/sdc2: Created a new partition 2 of type 'Linux' and of size 511 MiB.
     /dev/sdc3: Done.
     
     New situation:
     Disklabel type: dos
     Disk identifier: 0x14d5d317
     
     Device     Boot   Start     End Sectors  Size Id Type
     /dev/sdc1          2048 4196351 4194304    2G 83 Linux
     /dev/sdc2       4196352 5242879 1046528  511M 83 Linux
     
     The partition table has been altered.
     Calling ioctl() to re-read partition table.
     Syncing disks.

## Соберите mdadm RAID1 на паре разделов 2 Гб.

    vagrant@sysadm-fs:~$ sudo mdadm --create /dev/md0 --level=1 --raid-devices=2 /dev/sd[bc]1
    mdadm: Note: this array has metadata at the start and
        may not be suitable as a boot device.  If you plan to
        store '/boot' on this device please ensure that
        your boot-loader understands md/v1.x metadata, or use
        --metadata=0.90
    Continue creating array? yes
    mdadm: Defaulting to version 1.2 metadata
    mdadm: array /dev/md0 started.

## Соберите mdadm RAID0 на второй паре маленьких разделов.

    vagrant@sysadm-fs:~$ sudo mdadm --create /dev/md1 --level=0 --raid-devices=2 /dev/sd[bc]2
    mdadm: Defaulting to version 1.2 metadata
    mdadm: array /dev/md1 started.

## Создайте 2 независимых PV на получившихся md-устройствах.

    vagrant@sysadm-fs:~$ sudo pvcreate /dev/md0
      Physical volume "/dev/md0" successfully created.
    vagrant@sysadm-fs:~$ sudo pvcreate /dev/md1
      Physical volume "/dev/md1" successfully created.

## Создайте общую volume-group на этих двух PV.

    vagrant@sysadm-fs:~$ sudo vgcreate newvgraid /dev/md0 /dev/md1
      Volume group "newvgraid" successfully created

## Создайте LV размером 100 Мб, указав его расположение на PV с RAID0.

    vagrant@sysadm-fs:~$ sudo lvcreate -L 100m -n lvnewmd1 newvgraid /dev/md1
      Logical volume "lvnewmd1" created.

## Создайте mkfs.ext4 ФС на получившемся LV.

     vagrant@sysadm-fs:~$ sudo mkfs.ext4 -L fsext4 -m 1 /dev/mapper/newvgraid-lvnewmd1
     mke2fs 1.45.5 (07-Jan-2020)
     Creating filesystem with 25600 4k blocks and 25600 inodes
     
     Allocating group tables: done
     Writing inode tables: done
     Creating journal (1024 blocks): done
     Writing superblocks and filesystem accounting information: done

## Смонтируйте этот раздел в любую директорию, например, /tmp/new.

     vagrant@sysadm-fs:~$ mkdir /tmp/new
     vagrant@sysadm-fs:~$ sudo mount /dev/mapper/newvgraid-lvnewmd1 /tmp/new/
     vagrant@sysadm-fs:~$ mount | grep /tmp/new
     /dev/mapper/newvgraid-lvnewmd1 on /tmp/new type ext4 (rw,relatime,stripe=256)

## Поместите туда тестовый файл, например wget https://mirror.yandex.ru/ubuntu/ls-lR.gz -O /tmp/new/test.gz

     vagrant@sysadm-fs:~$ sudo wget https://mirror.yandex.ru/ubuntu/ls-lR.gz -O /tmp/new/test.gz
     --2023-01-22 17:56:56--  https://mirror.yandex.ru/ubuntu/ls-lR.gz
     Resolving mirror.yandex.ru (mirror.yandex.ru)... 213.180.204.183, 2a02:6b8::183
     Connecting to mirror.yandex.ru (mirror.yandex.ru)|213.180.204.183|:443... connected.
     HTTP request sent, awaiting response... 200 OK
     Length: 24414066 (23M) [application/octet-stream]
     Saving to: ‘/tmp/new/test.gz’
     
     /tmp/new/test.gz              100%[=================================================>]  23.28M  4.69MB/s    in 5.2s
     
     2023-01-22 17:57:01 (4.50 MB/s) - ‘/tmp/new/test.gz’ saved [24414066/24414066]

## Прикрепите вывод lsblk

    vagrant@sysadm-fs:~$ lsblk
    NAME                      MAJ:MIN RM  SIZE RO TYPE  MOUNTPOINT
    loop0                       7:0    0 67.2M  1 loop  /snap/lxd/21835
    loop2                       7:2    0 61.9M  1 loop  /snap/core20/1328
    loop3                       7:3    0 49.8M  1 loop  /snap/snapd/17950
    loop4                       7:4    0 63.3M  1 loop  /snap/core20/1778
    loop5                       7:5    0 91.9M  1 loop  /snap/lxd/24061
    sda                         8:0    0   64G  0 disk
    ├─sda1                      8:1    0    1M  0 part
    ├─sda2                      8:2    0  1.5G  0 part  /boot
    └─sda3                      8:3    0 62.5G  0 part
      └─ubuntu--vg-ubuntu--lv 253:0    0 31.3G  0 lvm   /
    sdb                         8:16   0  2.5G  0 disk
    ├─sdb1                      8:17   0    2G  0 part
    │ └─md0                     9:0    0    2G  0 raid1
    └─sdb2                      8:18   0  511M  0 part
      └─md1                     9:1    0 1018M  0 raid0
        └─newvgraid-lvnewmd1  253:1    0  100M  0 lvm   /tmp/new
    sdc                         8:32   0  2.5G  0 disk
    ├─sdc1                      8:33   0    2G  0 part
    │ └─md0                     9:0    0    2G  0 raid1
    └─sdc2                      8:34   0  511M  0 part
      └─md1                     9:1    0 1018M  0 raid0
        └─newvgraid-lvnewmd1  253:1    0  100M  0 lvm   /tmp/new

## Протестируйте целостность файла

    vagrant@sysadm-fs:~$ sudo gzip -t /tmp/new/test.gz
    vagrant@sysadm-fs:~$ echo $?
    0

## Используя pvmove, переместите содержимое PV с RAID0 на RAID1

    vagrant@sysadm-fs:~$ sudo pvmove -n lvnewmd1 /dev/md1 /dev/md0
      /dev/md1: Moved: 8.00%
      /dev/md1: Moved: 100.00%

## Сделайте --fail на устройство в вашем RAID1 md.

    vagrant@sysadm-fs:~$ sudo mdadm --fail /dev/md0 /dev/sdb1
    mdadm: set /dev/sdb1 faulty in /dev/md0

## Подтвердите выводом dmesg, что RAID1 работает в деградированном состоянии.

    vagrant@sysadm-fs:~$ dmesg | tail
    [ 5153.088613] md/raid1:md0: not clean -- starting background reconstruction
    [ 5153.088615] md/raid1:md0: active with 2 out of 2 mirrors
    [ 5153.088626] md0: detected capacity change from 0 to 2144337920
    [ 5153.089299] md: resync of RAID array md0
    [ 5163.462733] md: md0: resync done.
    [ 5697.098879] md1: detected capacity change from 0 to 1067450368
    [ 6889.546143] EXT4-fs (dm-1): mounted filesystem with ordered data mode. Opts: (null)
    [ 6889.546149] ext4 filesystem being mounted at /tmp/new supports timestamps until 2038 (0x7fffffff)
    [ 7602.902879] md/raid1:md0: Disk failure on sdb1, disabling device.
                   md/raid1:md0: Operation continuing on 1 devices.

## Протестируйте целостность файла, несмотря на "сбойный" диск он должен продолжать быть доступен

    vagrant@sysadm-fs:~$ sudo gzip -t /tmp/new/test.gz
    vagrant@sysadm-fs:~$ echo $?
    0
