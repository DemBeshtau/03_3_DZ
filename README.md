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
2. Смена типа раздела на sdb c id 83 (linux) на fd (raid)
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
&ensp;После выполнения данной команды требуется правка файла fstab, в котором необходимо заменить UUID sda1 на UUID md0.<br/>
9. Создание конфигурационного файла для mdadm. Данное действие осуществляется с целью предупреждения смены имени устройства /dev/md0 при перезагрузке:
```shell
mdadm --detail --scan > /etc/mdadm/mdadm.conf
```
10. Подготовка нового образа initramfs с необходимыми модулями:
```shell
mv initramfs-3.10.0-1127.el7.x86_64.img initramfs-3.10.0-1127.el7.x86_64.img.bak
dracut /boot/initramfs-$(uname -r).img
```
11. Правка файла /etc/default/grub. Необходимо передать ядру опцию rd.auto=1, путём добавления её в параметр GRUB_CMDLINE_LINUX. Данное действие осуществляется для того, чтобы имеющийся RAID собирался автоматически.
&ensp;Кроме того, в параметр GRUB_CMDLINE_LINUX добавляются опции rd.break (переход в командную строку в конце обработки initramfs) и enforcing=0 (загрузка SELinux в permissive режиме). Данные действия позволяют залогиниться в системе при загрузке с раздела sdb1.
12. Реконфигурирование GRUB:
```shell
grub2-mkconfig -o /boot/grub2/grub.cfg
```
13. Установка GRUB на sdb:
```shell
grub2-install /dev/sdb
```
14. Перезагрузка системы и её запуск с RAID;
15. Смена типа раздела на sda c id 83 (linux) на fd (raid):
```shell
fdisk /dev/sda: 
```
16. Добавление /dev/sda1 в RAID 1:
```shell
mdadm --manage /dev/md0 --add /dev/sda1
```
17. Установка GRUB на sda:
```shell
grub2-install /dev/sda
```
18. Перезагрузка и старт системы с md0;
19. Просмотр устройства, с которого стартовала система:
```shell
df -h | grep -Po "/dev/..."
/dev/shm
/dev/md0
```
20. Вывод списка блочных устройств:
```shell
lsblk
NAME    MAJ:MIN RM SIZE RO TYPE  MOUNTPOINT
sda       8:0    0  40G  0 disk  
`-sda1    8:1    0  40G  0 part  
  `-md0   9:0    0  40G  0 raid1 /
sdb       8:16   0  40G  0 disk  
`-sdb1    8:17   0  40G  0 part  
  `-md0   9:0    0  40G  0 raid1 / 
```
#### Более подробный ход решения задачи и вывод команд представлен в прилагающемся логе утилиты Script - relocation.txt. ####
