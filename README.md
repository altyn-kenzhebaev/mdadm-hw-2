## Задание со *, собрать Vagrantfile, который сразу собирает систему с подключенным рейдом и смонтированными разделами.
### Vagrant
Для выполнения этого действия требуется установить приложением git:
`git clone https://github.com/altyn-kenzhebaev/mdadm-hw-2.git`
В текущей директории появится папка с именем репозитория. В данном случае hw-1. Ознакомимся с содержимым:
```
cd mdadm-hw-2
ls -l
README.md
mv_OS_raid1
Vagrantfile
mdadm.sh
```
Здесь:
- README.md - файл с данным руководством
- mv_OS_raid1 -  директория со Vagrantfile для выполнения задания **
- Vagrantfile - файл описывающий виртуальную инфраструктуру для `Vagrant`
- mdadm.sh - скрипт выполняющийся при создании виртуальной машины

### Разбор Vagrantfile
Для добавления дисков и дальнейшего монтирования после перезагрузки (в т.ч. чтобы данные файлы не кофликтовали с другими ВМ):
```
		      :sata1 => {
            :dfile => home + '/VirtualBox VMs/mdadmhw2/sata1.vdi', #переносим в папку mdadmhw2, чтобы не кофликтовала с др. ВМ
			      :size => 1024,
		      	:port => 1
		      },
```
В провижн добавляем запуск скрипта:
```
          box.vm.provision "shell", inline: <<-SHELL
            yum install -y mdadm smartmontools hdparm gdisk lvm2
            /vagrant/mdadm.sh
  	      SHELL
```
### Разбор mdadm.sh
Создаем RAID и делаем чтобы он автоматически собирался во время загрузки ОС:
```
mdadm --zero-superblock --force /dev/sd{b,c,d,e,f}
mdadm --create --verbose /dev/md0 -l 6 -n 5 /dev/sd{b,c,d,e,f}

mkdir /etc/mdadm
echo "DEVICE partitions" > /etc/mdadm/mdadm.conf
mdadm --detail --scan --verbose | awk '/ARRAY/{print}' >> /etc/mdadm/mdadm.conf
```
Создаем таблицу разделов и выполняем разметку диска:
```
parted -s /dev/md0 mklabel gpt
parted /dev/md0 mkpart primary ext4 0% 20%
parted /dev/md0 mkpart primary ext4 20% 40%
parted /dev/md0 mkpart primary ext4 40% 60%
parted /dev/md0 mkpart primary ext4 60% 80%
parted /dev/md0 mkpart primary ext4 80% 100%
```
Форматируем в нужную файловую систему, монтируем и добавляем fstab, для автоматического монтирования после перезагрузки:
```
for i in $(seq 1 5); do 
    mkfs.ext4 /dev/md0p$i; 
    mkdir -p /raid/part$i; 
    mount /dev/md0p$i /raid/part$i; 
    echo "UUID=$(blkid -o value -s UUID /dev/md0p$i)  /raid/part$i   ext4  defaults 0 0" >> /etc/fstab;
```
------------

# Задание со **, перенести работающую систему с одним диском на RAID 1.
Теперь  создаем виртуальную машину, на которо мы должны пперенести работающую систему с одним диском на RAID 1.
### Разбор Vagrantfile
Для добавления дисков и дальнейшего монтирования после перезагрузки (в т.ч. чтобы данные файлы не кофликтовали с другими ВМ):
```
:sata1 => {
			      :dfile => home + '/VirtualBox VMs/mv_OS_raid1/sata1.vdi', # Диск для загрузки, раздел boot
			      :size => 1024,
		      	:port => 1
		      },
		      :sata2 => {
            :dfile => home + '/VirtualBox VMs/mv_OS_raid1/sata2.vdi', # Для создания диска RAID 1, с тем же размером раздела, на котором ОС
            :size => 40960,
			      :port => 2
		      },
```
### Запуск ВМ и lsblk
Переходим в папку mv_OS_raid1 и поднимаем виртуальную машину:
```
cd mv_OS_raid1
vagrant up
```
Далее заходим в ВМ, и выводим lsblk:
```
vagrant ssh
[vagrant@mdadm ~]$ sudo -i
[root@mdadm ~]# lsblk
NAME   MAJ:MIN RM SIZE RO TYPE MOUNTPOINT
sda      8:0    0  40G  0 disk 
└─sda1   8:1    0  40G  0 part /
sdb      8:16   0   1G  0 disk 
sdc      8:32   0  40G  0 disk 
```
### Переносим раздел boot на отдельный диск
Для переноса раздела boot мы возьмем диск sdb, сначала создаем таблицу разделов, выполняем разметку диска, форматируем в файловую систему XFS:
```
parted /dev/sdb mklabel msdos
parted /dev/sdb mkpart p xfs 1MiB 100%
mkfs.xfs /dev/sdb1
```
Монтируем и переносим данные:
```
mkdir /mnt/sdb1
mount /dev/sdb1 /mnt/sdb1
cp -a /boot/* /mnt/sdb1
```
Настраиваем grub на этот диск:
```
grub2-install --boot-directory=/mnt/sdb1 /dev/sdb
```
Добавляем запись в fstab:
```
echo "UUID=$(blkid -o value -s UUID /dev/sdb1)  /boot   xfs  defaults 0 0" >> /etc/fstab
```
раздел boot готов, не нужно забывать что загрузку ОС нужно выбрать именно с этого диска, для этого в время перезагрузки ВМ требуется нажать кнопку F12 и выбрать диск с которого требуется загрузиться

### Создаем RAID1 без одного диска
Для создания RAID1 без одного диска потребуется выполнить следующее:
```
mdadm --zero-superblock --force /dev/sdc
mdadm --create --verbose /dev/md0 -l 1 -n 2 /dev/sdc missing
mkdir /etc/mdadm
echo "DEVICE partitions" > /etc/mdadm/mdadm.conf
mdadm --detail --scan --verbose | awk '/ARRAY/{print}' >> /etc/mdadm/mdadm.conf 
```
Далее создаем LVM, форматируем ФС:
```
pvcreate /dev/md0
vgcreate vg01 /dev/md0
llvcreate -L 10Gb -n root vg01
mkfs.xfs /dev/vg01/root
```
Монтируем и переносим данные:
```
mkdir /mnt/rootfs
mount /dev/vg01/root /mnt/rootfs/
rsync -aHAXx --delete --exclude={/dev/*,/proc/*,/sys/*,/tmp/*,/run/*,/mnt/*,/media/*,/boot/*,/lost+found} / /mnt/rootfs/
```
Для успешной загрузки RAID1, требуется выполнить следующие команды:
```
mount --bind /dev /mnt/rootfs/dev/
mount --bind /sys /mnt/rootfs/sys/
mount -t proc /proc /mnt/rootfs/proc/
chroot /mnt/rootfs/
```
Редактируем /etc/fstab, подставив значения `blkid -o value -s UUID /dev/vg01/root`
Требуется изменение настройках файла GRUB /etc/default/grub, должен иметь следующий вид:
```
GRUB_CMDLINE_LINUX="no_timer_check console=tty0 console=tty net.ifnames=0 biosdevname=0 elevator=noop crashkernel=auto rd.auto rd.auto=1"
GRUB_PRELOAD_MODULES="mdraid1x"
```
Монтируем /boot, генерируем ядро с опциями предзагрузки mdraid, генерируем меню GRUB:
```
mount /boot
dracut -v --mdadmconf --fstab --add="mdraid" --filesystems "xfs ext4 ext3" --add-drivers="raid1" --force
grub2-mkconfig -o /boot/grub2/grub.cfg
touch /.autorelabel
```
Далее выходим из chroot и перезагружаем ОС:
```
exit
reboot
```
Система перезагрузится 2 раза, не забываем жать F12 и выбирать нужный диск. Далее заходим в ВМ, проверяем загрузку ОС с нужного нам раздела:
```
vagrant ssh
Last login: Tue Mar 14 07:09:28 2023 from 10.0.2.2
[vagrant@mdadm ~]$ sudo -i
[root@mdadm ~]# lsblk
NAME          MAJ:MIN RM  SIZE RO TYPE  MOUNTPOINT
sda             8:0    0    1G  0 disk  
└─sda1          8:1    0 1023M  0 part  /boot
sdb             8:16   0   40G  0 disk  
└─md0           9:0    0   40G  0 raid1 
  └─vg01-root 253:0    0   10G  0 lvm   /
sdc             8:32   0   40G  0 disk  
└─sdc1          8:33   0   40G  0 part  
[root@mdadm ~]#
```
ОС успешно загрузилась с RAID1, далее добавляем диск /dev/sdc в массив, предворительно вычистив superblock:
```
[root@mdadm ~]# mdadm --zero-superblock --force /dev/sdc
mdadm: Unrecognised md component device - /dev/sdc
[root@mdadm ~]# mdadm --manage /dev/md0 --add /dev/sdc
mdadm: added /dev/sdc
[root@mdadm ~]# cat /proc/mdstat 
Personalities : [raid1] 
md0 : active raid1 sdc[2] sdb[0]
      41909248 blocks super 1.2 [2/1] [U_]
      [>....................]  recovery =  1.9% (815872/41909248) finish=11.7min speed=58276K/sec
      
unused devices: <none>
```
Конечный результат lsblk:
```
[root@mdadm ~]# lsblk 
NAME          MAJ:MIN RM  SIZE RO TYPE  MOUNTPOINT
sda             8:0    0    1G  0 disk  
└─sda1          8:1    0 1023M  0 part  /boot
sdb             8:16   0   40G  0 disk  
└─md0           9:0    0   40G  0 raid1 
  └─vg01-root 253:0    0   10G  0 lvm   /
sdc             8:32   0   40G  0 disk  
├─sdc1          8:33   0   40G  0 part  
└─md0           9:0    0   40G  0 raid1 
  └─vg01-root 253:0    0   10G  0 lvm   /
```
