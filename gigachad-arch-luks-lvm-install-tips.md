## Что это?
Закладка мне будущему, если забуду как ставить ArchLinux с LVM и шифрованием LUKS вручную. Используется 2 шт NVMe, один под систему и т.д., другой полностью под home.
Была статья из заблокированого немногим ранее ресурса, скопирован и адаптирован под себя.

## Начнём-с

Запускаемся с образа. Проверяем наличие Интернета (оно нам понадобиться в процессе установки) и доступность диска, над которым планируем поработать.

### Разметка и шифрование

Для начала размечаем диск с помощью fdisk:

```
fdisk /dev/nvmeXXX
```
Первый раздел — EFI sytem Partition, я выделяю под него 256Mb

Второй раздел — boot, 512 Mb
Третий раздел будет зашифрован (нет, boot мы шифровать не будем, ибо запускаться и получать возможность ввода пароля на расшифровку нам как-то надо).

Для первого раздела выбираем тип EFI system, для остальных оставляем по умолчанию Linux filesystem.

Весь оставшийся размер диска размечаем под третью часть (на разделы мы его разобьём чуть позже).

Переходим к шифрованию. Для начала подгрузим необходимые модули ядра (не пугайтесь, это не больно). В этом нам поможет утилита modprobe.
```
modprobe dm-crypt
```
```
modprobe dm-mod
```
Устанавливаем шифрование на наши разделы. 
Нас попросят подтвердить наше намерение заглавными буквами (то есть YES, а не yes или y), а далее ввести и подтвердить пароль. Во избежание проблем крайне не рекомендую этот пароль забывать.
Для root:
```
cryptsetup luksFormat -v -s 512 -h sha512 /dev/nvme0n1p3
```
И для home:
```
cryptsetup luksFormat -v -s 512 -h sha512 /dev/nvme1n1p1
``` 

Открываем его (запросит установленный пароль):
```
cryptsetup open /dev/nvme0n1p3 luks_lvm
```
```
cryptsetup open /dev/nvme1n1p1 luks_lvm2
```

Если нужно поменять пароль:
1. Добавить новый ключ:
```
cryptsetup luksAddKey /dev/nvme1n1p3
```
2. Проверить
```
cryptsetup luksOpen /dev/nvme1n1p3 test
```
3. Если всё ок, удалить старый ключ
```
cryptsetup luksRemoveKey /dev/nvme1n1p3
```

И теперь разбиваем оба диска.

root и SWAP:
Создаём раздел:
```
pvcreate /dev/mapper/luks_lvm
```

Создаём группу:
```
vgcreate arch /dev/mapper/luks_lvm
```
Создаём логические тома в меру своей испорченности:
```
lvcreate -n swap -L 16G -C y arch
```
```
lvcreate -n root -L 100G arch
```
Если у вас один диск, а не два, то можно всё оставшееся место отдать под home:
```
lvcreate -n home -l +100%FREE arch
```

home:
Тут я заполнил полностью:
```
pvcreate /dev/mapper/luks_lvm2
```
```
vgcreate arch /dev/mapper/luks_lvm2
```
```
lvcreate -n home -l +100%FREE arch2
```

Разумеется, количество и размеры разделов зависят от ваших пожеланий и возможностей используемого “железа”.

Если ошиблись и создали неверный раздел или объём, то его можно удалить, как пример "home"
```
lvremove /dev/mapper/arch-home
```

Форматируем разделы.
Я использую файловую систему ext4, но ничего не мешает вам использовать то, что больше по вкусу.
Обратите внимание, что раздел EFI должен быть отформатирован в fat32.
```
mkfs.fat -F32 /dev/nvmeon1p1
```
```
mkfs.ext4 /dev/nvme0n1p2
```
```
mkfs.ext4 root /dev/mapper/arch-root
```
```
mkfs.ext4 home /dev/mapper/arch2-home
```
```
mkswap /dev/mapper/arch-swap
```

Монтируем то, что получилось:
```
mount /dev/mapper/arch-root /mnt
```
```
mkdir -p /mnt/{boot,home}
```
```
mount /dev/nvme0n1p2 /mnt/boot
```
```
mkdir /mnt/boot/efi
```
```
mount /dev/nvme0n1p1 /mnt/boot/efi
```
```
mount /dev/mapper/arch2-home /mnt/home
```
```
swapon /dev/mapper/arch-swap
```
Проверяем что всё смонтировано правильно:
```
swapon -a; swapon -s; lsblk
```

У меня выглядит всё так:
```
Filename				 Type		Size		Used		Priority
/dev/dm-1                               partition	16777212	0		-2
NAME             MAJ:MIN RM   SIZE RO TYPE  MOUNTPOINTS
nvme0n1          259:0    0   125G  0 disk  
├─nvme0n1p1      259:1    0   256M  0 part  /boot/efi
├─nvme0n1p2      259:2    0   512M  0 part  /boot
└─nvme0n1p3      259:3    0   124G  0 part  
  └─luks_lvm     254:0    0   124G  0 crypt 
    ├─arch-swap  254:1    0    16G  0 lvm   [SWAP]
    └─arch-root  254:2    0   108G  0 lvm   /
nvme0n1          269:0    0 953.9G  0 disk  
└─nvme1n1p1      269:1    0 953.9G  0 part  
  └─luks2_lvm    264:0    0 953.9G  0 crypt 
    └─arch2-home 264:1    0 953.9G  0 lvm   /home
```

### Установка и первичная настройка OC

Приступаем к установке Arch на наш свежеразделанный диск*:
```
pacstrap /mnt base base-devel efibootmgr nano grub mkinitcpio linux linux-firmware lvm2 vi nano
```
* - я использую ядро linux (есть ешё такие как linux-zen и т.д.).
 
Если всё прошло хорошо с установкой, генерируем fstab
```
genfstab -U -p /mnt > /mnt/etc/fstab
```
И заходим внутрь нашей новенькой системы
```
arch-chroot /mnt /bin/bash
```
Для удобства ставим сразу тип раскладки клавиатуры (у меня jp106)
```
localectl set-keymap --no-convert jp106
```
Или же тоже самое можно сделать в файле
```
nano /etc/vconsole.conf
```
Конфигурируем mkinitcpio (что это такое и с чем это едят можно подглядеть на ArchWiki https://wiki.archlinux.org/index.php/Mkinitcpio_(Русский) )
```
vim /etc/mkinitcpio.conf
```
Находим строчку HOOKS и дописываем encrypt, lvm2 и другие необходимые параметры. Мой HOOKS выглядит, обычно, так:
```
HOOKS=(base udev autodetect keyboard keymap modconf block encrypt lvm2 filesystems fsck)
```
Обратите внимание — порядок имеет значение!


В приведённой мной конфигурации в MODULES я добавляю ext4 и модули для видеокарт nvidia для дальнейшей установки wayland.
Получается такое для nvidia:
```
MODULES=(ext4 usbhid xhci_hcd  nvidia  nvidia_modeset nvidia_uvm nvidia_drm)
```
Для видеокарт AMD:
```
MODULES=(ext4 usbhid xhci_hcd amdgpu)
```
Для видеокарт Intel (в данном случае uhd620):
```
MODULES=(ext4 usbhid xhci_hcd i915)
```
Устанавливаем grub
```
grub-install — efi-directory=/boot/efi
```
Конфигурируем
```
vim /etc/default/grub
```
Перед следующим этапом нужно получить UUID диска (не раздела, а именно диска) с root:
```
blkid |grep nvme0n1p3
```
Обычно, он указывается в начале, по типу: UUID= ...

```
GRUB_CMDLINE_LINUX_DEFAULT=”loglevel=3 quiet nvidia_drm.modeset=1”
GRUB_CMDLINE_LINUX=”resume=/dev/mapper/arch-swap cryptdevice=UUID=<UUID диска с root>:luks_lvm root=/dev/mapper/arch-root”
```
Если диск всего один и не планируете апгрейд, то можно вместо ```cryptdevice=UUID=<UUID диска с root>:luks_lvm``` написать ```cryptdevice=/dev/nvme0n1p3:luks_lvm```, всё будет работать.
В противном случае, нужно указывать UUID. Дело в том, что при запуске ПК, диски могут определяться в разном порядке, в зависимости от фазы луны и прочей магии. По этому в initramfs, бывает, что обрабатывается только один cryptdevice, а второй не расшифровывается и, следовательно, root не маппится и система не может смонтировать / или же проблемы с /home.
Как вывод, проще и надёжней, а так же безопасней расшифровывать последующие диски уже после загрузки root.
Это делается через crypttab.

Последующие диски указываются в ```/etc/crypttab```

Пример:
```
# <name>       <device>                                     <password>              <options>
luks_lvm2      UUID=<UUID диска с home>                     none                    luks
```
password - none означает, что не указан файл или пароль, по этому он будет запрашиваться при загрузке.
UUID можно узнать так же, как было выше.

Не забываем добавить пользователя и добавить его группы и домашнюю папку. Дополнительно поставить пароли для пользователя и root.
```
useradd -m -G adm,ftp,games,http,log,rfkill,,sys,systemd-journal,uucp,wheel,audio,disk,floppy,input,kvm,optical,scanner,storage,video,i2c <username>
```
Устанавливаем пароль для root'a
```
passwd
```
И созданному пользователю
```
passwd <username>
```
Чтобы пользоваться командой "sudo" нужно раскомментировать строку в sudoers (/etc/sudoers):
```
%wheel ALL=(ALL:ALL) ALL
```
Гененрируем образ
```
mkinitcpio -v -p linux
```
Если установлено несколько ОС параллельно и чтобы можно было выбирать из grub куда вам грузиться, то необходимо установить os-prober 
```
sudo pacman -Suy
```
```
sudo pacman -S os-prober
```
и необходимо включить его, раскомментировав строку (/etc/default/grub)
```
GRUB_DISABLE_OS_PROBER=false
```
Генерируем конфигурацию загрузчика
```
grub-mkconfig -o /boot/grub/grub.cfg
```
```
grub-mkconfig -o /boot/efi/EFI/arch/grub.cfg
```
Меняем хостнейм с дефолтного на свой в файле:
```
/etc/hostname
```

Устанавливаем необходимые драйверы и, при необходимости, нужные оконные менеджеры/композиторы.
У меня было следующее:
```
pacman -S networkmanager nvidia-dkms
```
В случае, если у вас установочный образ пакет старый и проблемы вида "pacman-key", "keyring" и т.д.
Тогда следует их обновить:
```
pacman-key --init
```
```
pacman-key --populate 
```
```
pacman-key --refresh-keys
```
Включаем службы NetworkManager'a и иных менеджеров:
```
systemctl enable NetworkManager
```
Выходим обратно в загрузочную флешку
```
exit
```
Размонтируемся
```
umount -R /mnt
```
Перезагружаемся
```
reboot
```
Если у вас возникли вопросы по установке которые вы не можете решить с помощью вики арча.


Полезности:
1. Установка темы GTK (должно быть расположено в ~/.themes)
```
gsettings set org.gnome.desktop.interface gtk-theme <название темы>
```
Тёмная тема для GTK3:
```
gtk-application-prefer-dark-theme = true
```
Тёмная тема для GTK4:
```
gsettings set org.gnome.desktop.interface color-scheme prefer-dark
```
mpd не запускается с медиакнопок - проверь:
```
systemctl --user --now enable mpd-mpris
```

Включение NTP
```
timedatectl status
```
```
timedatectl set-ntp true
```
```
timedatectl set-timezone "Asia/Tokyo"
```
2. MTP для подключения телефона и т.д.
   
Нужный пакет - ```jmtpfs```

2.1. Создать папку для MTP;

2.2. сделать ссылку (но зачем?):
```
sudo ln -s /<путь дo>/MTP /<путь дo>/MTP.jmtpfs
```
2.3. добавить в ```/etc/fstab```:
```
#jmtpfs <mount path>        fuse nodev,allow_other,<other options>                             0    0
 jmtpfs                 /<путь дo>/MTP           fuse nodev,allow_other,rw,user,noauto,noatime,uid=1000,gid=1000    0    0
```
2.4. раскомментировать "user_allow_other" в ```/etc/fuse.conf```
