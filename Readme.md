## ДЗ.3 Файловые системы и LVM

### Создаём временный том для раздела / :
>pvcreate /dev/sdb
>vgcreate vg_root /dev/sdb
>lvcreate -n lv_root -l +100%FREE /dev/vg_root

### Создаём на подготовленном временном разделе файловую систему xfs и монтируем этот раздел в /mnt :
>mkfs.xfs /dev/vg_root/lv_root
>mount /dev/vg_root/lv_root /mnt

### Копируем все данные с / раздела в /mnt:
>xfsdump -J - /dev/VolGroup00/LogVol00 | xfsrestore -J - /mnt

### Перенастраиваем grub для загрузки с нового раздела /:

>for i in /proc/ /sys/ /dev/ /run/ /boot/; do mount --bind $i /mnt/$i; done
>chroot /mnt/
>grub2-mkconfig -o /boot/grub2/grub.cfg

### Обновляем образ initrd
>cd /boot ; for i in `ls initramfs-*img`; do dracut -v $i `echo $i|sed "s/initramfs-//g; s/.img//g"` --force; done

### Вносим правки /boot/grub2/grub.cfg заменив rd.lvm.lv=VolGroup00/LogVol00 на rd.lvm.lv=vg_root/lv_root

### Перезагружаемся, запись вышеописанных действий утилитой script  в файле typescript_stage01

### Для изменения размера старой VG и возврата на неё раздела / с временного раздела удаляем старый LV размерм 40G и создаем новый на 8G:
>lvremove /dev/VolGroup00/LogVol00
>lvcreate -n VolGroup00/LogVol00 -L 8G /dev/VolGroup00

### Создаём на вновь созданном LV файловую систему xfs и монтируем этот раздел в /mnt :
>mkfs.xfs /dev/VolGroup00/LogVol00
>mount /dev/VolGroup00/LogVol00 /mnt

### Копируем все данные с временного раздела / в /mnt:
>xfsdump -J - /dev/vg_root/lv_root | xfsrestore -J - /mnt

### Перенастраиваем grub для загрузки с уменьшенного раздела / исключая правку /etc/grub2/grub.cfg
>for i in /proc/ /sys/ /dev/ /run/ /boot/; do mount --bind $i /mnt/$i; done
>chroot /mnt/
>grub2-mkconfig -o /boot/grub2/grub.cfg
>cd /boot ; for i in `ls initramfs-*img`; do dracut -v $i `echo $i|sed "s/initramfs-//g;s/.img//g"` --force; done

### Не выходя из chroot создаём раздел под /var:

>pvcreate /dev/sdc /dev/sdd
>vgcreate vg_var /dev/sdc /dev/sdd
>vcreate -L 950M -m1 -n lv_var vg_var

### Создаём этом разделе файловую систему ext4 и перемещаем туда данные /var:
>mkfs.ext4 /dev/vg_var/lv_var
>mount /dev/vg_var/lv_var /mnt
>cp -aR /var/* /mnt/
### Сохраняем содержимое старого /var:
>mkdir /tmp/oldvar && mv /var/* /tmp/oldvar
### Монтируем var на созданом раделе в каталог /var:
>umount /mnt
>mount /dev/vg_var/lv_var /var

### Вносим правки в fstsb для автомонтирования var на новом разделе при загрузке:
>echo "`blkid | grep var: | awk '{print $2}'` /var ext4 defaults 0 0" >> /etc/fstab

### Перезагружаемся, запись вышеописанных действий утилитой script  в файле typescript_stage02

### После презагрузки удаляем временные LV,VG и PV:
>lvremove /dev/vg_root/lv_root
>vgremove /dev/vg_root
>pvremove /dev/sdb


### Создаём отдельный том под /home с файловой системой ext4, перемещаем содержимое действующей /home в новую, монтируем новый /home и удаляем старый:
>lvcreate -n LogVol_Home -L 2G /dev/VolGroup00
>mkfs.ext4 /dev/VolGroup00/LogVol_Home
>mount /dev/VolGroup00/LogVol_Home /mnt/
>cp -aR /home/* /mnt/
>rm -rf /home/*
>umount /mnt
>mount /dev/VolGroup00/LogVol_Home /home/

### Вносим правки в fstsb для автомонтирования home на новом разделе при загрузке:
>echo "`blkid | grep Home | awk '{print $2}'` /home xfs defaults 0 0" >> /etc/fstab/

### Создаём снапшот,удаляем часть файлов и восстановливаем удалённое из снепшота:

#### Сгенерируем 20 файлов /home/:
>touch /home/file{1..20}
#### Делаем снапшот:
>lvcreate -L 100MB -s -n home_snap /dev/VolGroup00/LogVol_Home

#### Удаляем файлы:
>rm -f /home/file{11..20}

#### Восстанавливаем удалённое из снапшота:
>umount /home
>lvconvert --merge /dev/VolGroup00/home_snap
>mount /home
### Запись вышеописанных действий утилитой script  в файле typescript_stage03
