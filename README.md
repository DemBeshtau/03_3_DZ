# Работа с mdadm # 

- Перенести работающую систему с одним диском на RAID 1. Даунтайм на загрузку с нового диска предполагается.
### Исходные данные ###
   На диске /dev/sda в разделе /dev/sda1 размером 40 Гб установлена система; <br/>
   Диск /dev/sdb чистый
### Ход решения ###
1. Копирование разметки дискового пространства c sda на sdb:
```shell
sfdisk -d /dev/sda | sfdisk /dev/sdb:
```	
2. Смена типа раздела на sdb1 c id: 83 (linux) на fd (raid)
```shell
fdisk /dev/sdb: 
```
3. Подготовка RAID 1 в статусе degraded
```shell
mdadm --create /dev/md0 -l 1 -n 2 missing /dev/sdb1 
```
4. Форматирование получившегося /dev/md0 (исходная файловая система xfs):
```shell
mkfs.xfs /dev/md0
```
5. Монтрование /dev/md0 для последующих манипуляций:
```shell
mount /dev/md0 /mnt
```
6. Копирование исходной системы на /dev/md0:
```shell
rsync -axu / /mnt
```
7. Монтирование данных исходной системы в новый корень с последующим в него переходом:
```shell
mount --bind /proc /mnt/proc 
mount --bind /dev /mnt/dev 
mount --bind /sys /mnt/sys 
mount --bind /run /mnt/run 
chroot /mnt
```
8. Получение UUID устройства /dev/md0 с перенаправлением вывода в /etc/fstab:
```shell
ls -l /dev/disk/by-uuid | grep md >> /etc/fstab
```
&ensp;После выполнения данной команды требуется правка файла fstab, в котором необходимо заменить UUID sda1 на UUID md0.
9. Создание конфигурационного файла для mdadm. Данное действие осуществляется с целью предупреждения смены имени устройства /dev/md0 при перезагрузке:
```shell
mdadm --detail --scan > /etc/mdadm/mdadm.conf
```
10. Подготовка нового образа initramfs с необходимыми модулями:
