# homework-03

## linux

1) Уменьшит том под / до 8G
2) Выделить том под /home
3) Выделить том под /var - сделать в mirror
4) /home - сделать том для снапшотов
5) Прописать монтирование в fstab. Попробовать с разными опциями и разными
файловыми системами ( на выбор)


## Уменьшит том под / до 8G

Подготовим временный том для / раздела:
sudo pvcreate /dev/sdb
  Physical volume "/dev/sdb" successfully created
sudo vgcreate vg_root /dev/sdb
  Volume group "vg_root" successfully created
sudo lvcreate -n lv_root -l +100%FREE /dev/vg_root
  Logical volume "lv_root" created.

# Создадим на нем файловую систему и смонтируем его, чтобы перенести туда данные:
sudo mkfs.xfs /dev/vg_root/lv_root

Этой командой скопируем все данные с / раздела в /mnt:
mount /dev/vg_root/lv_root /mnt

sudo xfsdump -J - /dev/VolGroup00/LogVol00 | xfsrestore -J - /mnt
Проверяем  ls /mnt

# Переконфигурируем grub для того, чтобы при старте перейти в новый /

Сымитируем текущий root -> сделаем в него chroot и обновим grub:
  for i in /proc/ /sys/ /dev/ /run/ /boot/; do mount --bind $i /mnt/$i; done
  chroot /mnt/
  grub2-mkconfig -o /boot/grub2/grub.cfg
Generating grub configuration file ...
Found linux image: /boot/vmlinuz-3.10.0-862.2.3.el7.x86_64
Found initrd image: /boot/initramfs-3.10.0-862.2.3.el7.x86_64.img
done

# Обновим образ initrd
  cd /boot ; for i in `ls initramfs-*img`; do dracut -v $i `echo $i|sed "s/initramfs-//g; s/.img//g"` --force; done
*** Creating image file ***
*** Creating image file done ***
*** Creating initramfs image file '/boot/initramfs-3.10.0-862.2.3.el7.x86_64.img' done ***

# Чтобы при загрузке был смонтирован нужный root в файле
/boot/grub2/grub.cfg заменить rd.lvm.lv=VolGroup00/LogVol00 на rd.lvm.lv=vg_root/lv_root

Нужно изменить размер старой VG и вернуть на него рут. Для этого удаляем
старый LV размером в 40G и создаем новый на 8G:

lvremove /dev/VolGroup00/LogVol00
    Do you really want to remove active logical volume VolGroup00/LogVol00? [y/n]: y
   Logical volume "LogVol00" successfully removed

 lvcreate -n VolGroup00/LogVol00 -L 8G /dev/VolGroup00
    WARNING: xfs signature detected on /dev/VolGroup00/LogVol00 at offset 0. Wipe it? [y/n]: y
   Wiping xfs signature on /dev/VolGroup00/LogVol00.
   Logical volume "LogVol00" created.

mkfs.xfs /dev/VolGroup00/LogVol00
mount /dev/VolGroup00/LogVol00 /mnt
xfsdump -J - /dev/vg_root/lv_root | xfsrestore -J - /mnt
    xfsdump: Dump Status: SUCCESS
    xfsrestore: Restore Status: SUCCESS

Переконфигурируем grub, за исключением правки /etc/grub2/grub.cfg
for i in /proc/ /sys/ /dev/ /run/ /boot/; do mount --bind $i /mnt/$i; done
chroot /mnt/
grub2-mkconfig -o /boot/grub2/grub.cfg
    Generating grub configuration file ...
    Found linux image: /boot/vmlinuz-3.10.0-862.2.3.el7.x86_64
    Found initrd image: /boot/initramfs-3.10.0-862.2.3.el7.x86_64.img
    done
cd /boot ; for i in `ls initramfs-*img`; do dracut -v $i `echo $i|sed "s/initramfs-//g; s/.img//g"` --force; done
    *** Creating image file ***
    *** Creating image file done ***
    *** Creating initramfs image file '/boot/initramfs-3.10.0-862.2.3.el7.x86_64.img' done ***

# ├─VolGroup00-LogVol00    253:0    0    8G  0 lvm  /

### Выделить том под /var в зеркало

создаем зеркало:
pvcreate /dev/sdd /dev/sde

vgcreate vg_var /dev/sde /dev/sdd
   Volume group "vg_var" successfully created
lvcreate -L 950M -m1 -n lv_var vg_var
    Rounding up size to full physical extent 952.00 MiB
   Logical volume "lv_var" created.

Создаем на нем ФС и перемещаем туда /var:
mkfs.ext4 /dev/vg_var/lv_var

mount /dev/vg_var/lv_var /mnt
rsync -avHPSAX /var/ /mnt/

На всякий случай сохраняем содержимое старого var
mkdir /tmp/oldvar && mv /var/* /tmp/oldvar

Монтируем новый var в каталог /var
umount /mnt
mount /dev/vg_var/lv_var /var

Правим fstab для автоматического монтирования /var:
echo "`blkid | grep var: | awk '{print $2}'` /var ext4 defaults 0 0" >> /etc/fstab

После чего можно перезагружаться в новый (уменьшеннвй root) и удалять
временную Volume Group
lvremove /dev/vg_root/lv_root
vgremove /dev/vg_root
pvremove /dev/sdb


sdd                          8:48   0    1G  0 disk 
├─vg_var-lv_var_rmeta_0    253:3    0    4M  0 lvm  
│ └─vg_var-lv_var          253:7    0  952M  0 lvm  /var
└─vg_var-lv_var_rimage_0   253:4    0  952M  0 lvm  
  └─vg_var-lv_var          253:7    0  952M  0 lvm  /var
sde                          8:64   0    1G  0 disk 
├─vg_var-lv_var_rmeta_1    253:5    0    4M  0 lvm  
│ └─vg_var-lv_var          253:7    0  952M  0 lvm  /var
└─vg_var-lv_var_rimage_1   253:6    0  952M  0 lvm  
  └─vg_var-lv_var          253:7    0  952M  0 lvm  /var



# Выделzем том под /home по тому же принципу что делали для /var

vcreate -n LogVol_Home -L 2G /dev/VolGroup00
   Logical volume "LogVol_Home" created.
mount /dev/VolGroup00/LogVol_Home /mnt/
cp -aR /home/* /mnt/ 
rm -rf /home/*
umount /mnt
mount /dev/VolGroup00/LogVol_Home /home/

Правим fstab для автоматического монтирования /home
echo "`blkid | grep Home | awk '{print $2}'` /home xfs defaults 0 0" >> /etc/fstab

└─sda3                       8:3    0   39G  0 part 
  ├─VolGroup00-LogVol00    253:0    0    8G  0 lvm  /
  ├─VolGroup00-LogVol01    253:1    0  1.5G  0 lvm  [SWAP]
  └─VolGroup00-LogVol_Home 253:2    0    2G  0 lvm  /home


# /home - сделать том для снапшотов

Сгенерируем файлы в /home/:
touch /home/file{1..20}

Снять снапшот
 lvcreate -L 100MB -s -n home_snap /dev/VolGroup00/LogVol_Home
 
Удалить часть файлов
rm -f /home/file{11..20}

Процесс восстановления со снапшота
umount /home
lvconvert --merge /dev/VolGroup00/home_snap
mount /home


# на нашей куче дисков попробовать поставить btrfs/zfs:

с кешем и снэпшотами
разметить здесь каталог /opt

pvcreate /dev/sdb
vgcreate vg_opt /dev/sdb
lvcreate -L 950M -n lv_opt vg_opt
mkfs.btrfs /dev/vg_opt/lv_opt
mount /dev/vg_opt/lv_opt /mnt
rsync -avHPSAX /opt/ /mnt/
umount /mnt
mount /dev/vg_opt/lv_opt /opt
blkid
mount /dev/vg_opt/lv_opt /opt
lsblk

btrfs subvolume create /opt/
btrfs subvolume create /opt/sub_folder
btrfs subvolume show /opt/sub_folder/
touch 1
touch 2
touch 3
touch 4
btrfs subvolume create /opt/sub_snap
btrfs subvolume snapshot -r /opt/sub_folder/ /opt/sub_snap/
btrfs subvolume show /opt/sub_snap/sub_folder/
cat fstab



sdb                          8:16   0   10G  0 disk 
├─vg_opt-lv_opt            253:8    0  952M  0 lvm  /opt


