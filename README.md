# lvm_linux

## Prerequireties

Vagrantfile берем из этого [стенда](https://gitlab.com/otus_linux/stands-03-lvm) 

Перед началом работы поставим пакет xfsdump - он будет необходим для снятия копии / тома.  

```console
[root@lvm vagrant]# yum install xfsdump
```

Текущее состояние :

```console
[root@lvm vagrant]# lsblk
NAME                    MAJ:MIN RM  SIZE RO TYPE MOUNTPOINT
sda                       8:0    0   40G  0 disk 
├─sda1                    8:1    0    1M  0 part 
├─sda2                    8:2    0    1G  0 part /boot
└─sda3                    8:3    0   39G  0 part 
  ├─VolGroup00-LogVol00 253:0    0 37.5G  0 lvm  /
  └─VolGroup00-LogVol01 253:1    0  1.5G  0 lvm  [SWAP]
sdb                       8:16   0   10G  0 disk 
sdc                       8:32   0    2G  0 disk 
sdd                       8:48   0    1G  0 disk 
sde                       8:64   0    1G  0 disk 
```

## Уменьшить том под / до 8G

Подготовим временный том для / раздела на диске sdb.  

Создадим  physical volume на диске sdb :

```console
[root@lvm vagrant]# pvcreate /dev/sdb
  Physical volume "/dev/sdb" successfully created.
```

Создадим volume group   :

```console
[root@lvm vagrant]# vgcreate vg_root /dev/sdb
  Volume group "vg_root" successfully created
```

Создаем logical volume с именем `lv_root` и размером 100% от созданной volume group  :

```console
[root@lvm vagrant]# lvcreate -n lv_root -l +100%FREE /dev/vg_root
  Logical volume "lv_root" created.
```

Создадим на logical volume `lv_root`  файловую систему xfs :

```console
[root@lvm vagrant]# mkfs.xfs /dev/vg_root/lv_root
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

Смонтируем logical volume :

```console
[root@lvm vagrant]# mount /dev/vg_root/lv_root /mnt
```

Cкопируем все данные с `/` раздела в `/mnt:` :

```console
[root@lvm vagrant]# xfsdump -J - /dev/VolGroup00/LogVol00 | xfsrestore -J - /mnt
...
xfsrestore: Restore Status: SUCCESS
[root@lvm vagrant]#
```

Затем переконфигурируем grub для того, чтобы при старте перейти в новый `/`  

Сымитируем текущий root -> сделаем в него chroot :

```console
[root@lvm vagrant]# for i in /proc/ /sys/ /dev/ /run/ /boot/; do mount --bind $i /mnt/$i; done
```

Обновим grub :

```console 
[root@lvm vagrant]# grub2-mkconfig -o /boot/grub2/grub.cfg
Generating grub configuration file ...
Found linux image: /boot/vmlinuz-3.10.0-862.2.3.el7.x86_64
Found initrd image: /boot/initramfs-3.10.0-862.2.3.el7.x86_64.img
Found CentOS Linux release 7.5.1804 (Core)  on /dev/mapper/vg_root-lv_root
done
```

Обновим образ initrd (initial RAM disk) :  

```console
[root@lvm vagrant]# cd /boot ; for i in `ls initramfs-*img`; do dracut -v $i `echo $i|sed "s/initramfs-//g; \
> s/.img//g"` --force; done
```

Ну и для того, чтобы при загрузке был смонтирован нужный `root` нужно в файле
`/boot/grub2/grub.cfg` заменить `rd.lvm.lv=VolGroup00/LogVol00` на `rd.lvm.lv=vg_root/lv_root`

```console
[root@lvm boot]# sed -i -e 's/rd.lvm.lv=VolGroup00\/LogVol00/rd.lvm.lv=vg_root\/lv_root/' /boot/grub2/grub.cfg
```

Перезагружаемся с новым рут томом. 

