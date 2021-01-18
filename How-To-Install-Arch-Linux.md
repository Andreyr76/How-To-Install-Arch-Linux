# Установка

## Процесс загрузки Linux

Прочитайте [эту страницу](https://wiki.gentoo.org/wiki/Initramfs/Guide/ru). Она поможет понять действия, производимые при установке.

## Записываем образ

```bash
$ sudo dd if= of=/dev/sdX bs=8M status=progress; sync
```

Далее нужно загрузиться с флешки. Для этого во время загрузки системы нажимаем F2 или Del. Далее заходим в раздел Boot биоса и выбираем загрузку с флешки.

## Размечаем диск

```bash
$ cfdisk /dev/sda
```

Создаем таблицу разделов GPT и разбиваем диск на три раздела в следующем порядке:

1. Загрузочный с файловой системой FAT-32 ~ 150-250 MB. Выбираем Type `EFI System`;
1. Раздел с btrfs ‒ все оставшееся место за вычетом места зарезервированного под swap;
1. Swap ‒ от 2 гигабайт и более. Для использования гибернации нужно как минимум 2/3 от размера оперативной памяти. Обязательно должен быть последним чтобы мы могли менять его размер динамически, откусывая место от предыдущего раздела.

В конце в меню cfdisk не забываем записать изменения на диск `Write`, а потом выходим  ‒`Quit`.

```bash
$ mkfs.fat -n ESP -F32 /dev/sda1
$ mkfs.btrfs -L Linux /dev/sda2
$ mkswap -L Swap /dev/sda3
```

* Swap-файл будет занимать лишнее место в снапшотах поэтому отказываемся от него в пользу отдельного раздела.
* Если на диске уже установлена Windows и планируется использовать Dual Boot, то пропускаем создание EPS раздела.
* Почему btrfs? ‒ Это файловая система с поддержкой сжатия, дефрагментации и создания снапшотов на лету, динамическим размером разделов, которая себя зарекомендовала как весьма стабильная.

## Создаем сабвольюмы

```bash
$ mount /dev/sda2 /mnt
$ btrfs sub create /mnt/@
$ btrfs sub create /mnt/@home
# Создаем каталог для снапшотов (опционально). Снапшоты будут храниться вне корня
$ mkdir -p /mnt/snapshots
$ umount /mnt
```

## Монтируем систему

```bash
$ mount -o noatime,compress=lzo,space_cache,subvol=@ /dev/sda2 /mnt
$ mkdir /mnt/{boot,home}
$ mount /dev/sda1 /mnt/boot
$ mount -o noatime,compress=lzo,space_cache,subvol=@home /dev/sda2 /mnt/home
$ swapon /dev/sda3
```

## Устанавливаем ядро и базовые пакеты

```bash
$ pacstrap /mnt base \
    base-devel \
    linux \
    linux-firmware \
    linux-headers \
    alsa-utils \
    btrfs-progs \
    dialog \
    efibootmgr \
    exfat-utils \
    git \
    grub \
    man-db \
    man-pages \
    nano \
    netctl \
    net-tools \
    ntfs-3g \
    os-prober \
    vim \
    wget \
    zsh \
    xorg-server \
    xorg-apps \
    # дописываем нужные пакеты
```

В этот список можно добавить сл. пакеты:

| Название | Описание |
| -- | -- |
| xf86-video-ati |драйвер для видеокарт AMD |
| xf86-video-nouveau | драйвер для видеокарт Nvidia |
| intel-ucode | патч для процессоров Intel | 
| amd-ucode | патч для процессоров AMD |

## Генерируем fstab

```bash
$ genfstab -U /mnt >> /mnt/etc/fstab
```

## Chroot

```bash
$ arch-chroot /mnt
```

## Настраиваем время

```bash
$ ln -sf /usr/share/zoneinfo/Europe/Moscow /etc/localtime

# Устанавливаем аппартатные часы (материнка) по системным
$ hwclock --systohc
```

## Редактирование файлов

В дальнейших примерах используется vim. Чтобы выйти из Vim нужно нажать `<ESC>`, ввести `:q` и нажать `<Enter>`. Более простым редактором является nano:

```bash
$ nano <filename>
```

`^X` означает сочетание клавиш `Ctrl + X`.

## Локаль

```bash
$ vim /etc/locale.gen
```

Нужно найти и раскомментировать строки:

```
en_US.UTF-8 UTF-8
...
ru_RU.UTF-8 UTF-8
```

Генерируем локали:

```bash
$ locale-gen
```

Устанавливаем локаль по-умолчанию:

```bash
$ echo "LANG=en_US.UTF-8" > /etc/locale.conf
```

Исползуем ru_RU.UTF-8 для интерфейса на русском.

> Настройку локали терминала и ее переключения можно произвести после завершения установки.

Поддержка русских символов в системном терминале:

```bash
$ vim /etc/vconsole.conf
```

Пример:

```
KEYMAP=ru
FONT=cyr-sun16
```

Для перключения раскладки в терминале

```bash
$ vim /etc/X11/xorg.conf.d/00-keyboard.conf
```

Пример:

```
Section "InputClass"
        Identifier "system-keyboard"
        MatchIsKeyboard "on"
        Option "XkbLayout" "us,ru"
        Option "XkbModel" "pc104"
        Option "XkbOptions" "grp:alt_shift_toggle"
EndSection
```

Узнать геометрию клавитуры (pc104/pc105) можно так:

```bash
$ setxkbmap -print                                     
xkb_keymap {
	xkb_keycodes  { include "evdev+aliases(qwerty)"	};
	xkb_types     { include "complete"	};
	xkb_compat    { include "complete"	};
	xkb_symbols   { include "pc+us+ru:2+inet(evdev)+group(win_space_toggle)+compose(ralt)"	};
	xkb_geometry  { include "pc(pc104)"	};
};
```

## Хосты

```bash
$ echo имя_хоста > /etc/hostname
```

Прописываем все хосты:

```bash
$ vim /etc/hosts
```

Пример:

```
127.0.0.1 имя_хоста
::1 имя_хоста
127.0.1.1 имя_хоста.localdomain имя_хоста
```

## Создаем пользователя

```bash
# Устанавливаем пароль для пользователя root
$ passwd

# Создаем пользователя и устанавливаем ему шелл по-умолчанию (ничего лучше zsh нет)
$ useradd -m -g users -G wheel -s /bin/zsh имя_пользователя

# Устанавливаем пароль для нового пользователя
$ passwd имя_пользователя
```

Настройки sudo:

```bash
$ vim /etc/sudoers
```

Нужно раскомментировать:

```
%wheel ALL=(ALL) ALL
```

Для изменения времени через которое нужно вводить пароль для sudo меняем:

```
Defaults timestamp_timeout=<число_минут>
```

Если установить значение равное `-1`, то пароль нужно будет вводить единожды.

## Установка Grub

```bash
# Устанавливаем grub и прописывает себя в nvram материнки
$ grub-install --target=x86_64-efi --efi-directory=/boot/efi --bootloader-id=Arch
```

Отключаем автоматическую загрузку:

```bash
$ vim /etc/default/grub
```

Находим и изменяем:

```bash
GRUB_TIMEOUT=-1
```

Настраиваем mkinitcpio (можно добавить хуки): 

```bash
$ vim /etc/mkinitcpio.conf
```

Например для использования btrfs на нескольких устройствах в HOOKS нужно добавить btrfs до filesystems.

Если файл конфигурации был измнен, то генерируем новый (он генерируеться при установке ядра) архив initramfs (initcpio):

```bash
$ mkinitcpio -p linux
```

Генерируем конфигурацию для grub:

```bash
$ grub-mkconfig -o /boot/grub/grub.cfg
```

Аналогом использованию Grub является использование [systemd-boot](https://wiki.archlinux.org/index.php/Systemd-boot_(%D0%A0%D1%83%D1%81%D1%81%D0%BA%D0%B8%D0%B9)).

## Графическое окружение

Тут я не буду оригинальным и предложу установить Gnome:

```bash
$ pacman -S gnome
$ systemctl enbale gdm
$ systemctl enbale Networkmanager
```

## Перегружаемся

```bash
$ exit
$ reboot
```

# Настройка рабочей среды

## Работа с пакетами

Pacman не умеет работать с [AUR](https://wiki.archlinux.org/index.php/Arch_User_Repository_(%D0%A0%D1%83%D1%81%D1%81%D0%BA%D0%B8%D0%B9)). Для этого потребуется более продвинутый пакетный менеджер:

```zsh
cd /tmp
git clone https://aur.archlinux.org/yay-bin.git
cd yay-bin
makepkg -si
```

Пакеты стоит ставить только из репозиторяи, так можно будет их безболезненно удалить со всеми зависимости. Собирать из исходников программы то же не имеет смысла так как в репозиторяих Арча и так все самое последнее. Некоторые пакеты по-умолчанеиб собираются из исходников в таком случае следует отдавать предпочтение пакетам с постфиксом `-bin`.

У пакетов есть опциональныйе зависимости, которые можно посмотреть с помощью `yay -Qi <package>`. Если пакет не работает после установки, то ищем описание его настройки на  [ArchWiki](https://wiki.archlinux.org/index.php/Main_page).

## Настраиваем ZSH

```zsh
# Ставим менеджер плагинов zgen
$ yay -S zgen-git

$ vim ~/.zshrc
# Файл в итоге должен выглядеть так
# load zgen
source /usr/share/zsh/share/zgen.zsh

# if the init script doesn't exist
if ! zgen saved; then

  # specify plugins here
  zgen oh-my-zsh
  zgen oh-my-zsh plugins/archlinux
  zgen oh-my-zsh plugins/command-not-found
  zgen oh-my-zsh plugins/docker
  zgen oh-my-zsh plugins/extract
  zgen oh-my-zsh plugins/git
  # zgen oh-my-zsh plugins/git-flow-avh
  zgen oh-my-zsh plugins/history
  # zgen oh-my-zsh plugins/httpie
  # zgen oh-my-zsh plugins/node
  # zgen oh-my-zsh plugins/npm
  # zgen oh-my-zsh plugins/pass
  zgen oh-my-zsh plugins/pip
  zgen oh-my-zsh plugins/python
  zgen oh-my-zsh plugins/sudo
  zgen oh-my-zsh plugins/systemd
  zgen oh-my-zsh plugins/vscode
  zgen load zsh-users/zsh-autosuggestions
  zgen load zsh-users/zsh-completions
  zgen load zsh-users/zsh-syntax-highlighting
  zgen load caiogondim/bullet-train-oh-my-zsh-theme bullet-train

  # generate the init script from plugins above
  zgen save
fi

# После сохранения, перегружаем Shell и начнется установка плагинов
$ exec "$SHELL"

# Теперь нужно настроить плагин command-not-found
$ yay -S pkgfile
$ sudo pkgfile --update
# Включаем сервис для автообновления базы
$ sudo systemctl start pkgfile-update.timer
$ sudo systemctl enable pkgfile-update.timer

# Опять перегружаемся
$ exec "$SHELL"

# Результат
$ ruby
ruby may be found in the following packages:
  extra/ruby 2.6.5-1            /usr/bin/ruby
  community/gitlab 12.3.5-1     /usr/share/webapps/gitlab/vendor/bundle/ruby/2.5.0/gems/hamlit-2.8.8/bin/ruby
  community/logstash 7.4.0-1    /usr/share/logstash/bin/ruby
  community/ruby2.5 2.5.7-1     /opt/ruby2.5/bin/ruby
```

Для терминала стоит использовать шрифт типа `Source Code Pro`.

## [Цветовые темы для терминала](https://github.com/Mayccoll/Gogh)

```zsh
# Падает с ошибкой, если нет профиля по-умолчаниюю для терминала
# Для всех созданных профилей будет использовавться шрифт профиля по-умолчанию
$ bash -c  "$(wget -qO- https://git.io/vQgMr)"
# Удаление всех профилей
$ dconf reset -f /org/gnome/terminal/legacy/profiles:/
```

## Ставим нормальный браузер

```zsh
# На выбор
$ yay -S google-chrome chromium firefox

# И для управления расширениями Gnome из браузера
$ yay -S chrome-gnome-shell
```

## Расширения для Gnome

Список must have расширений:

| Название | Описание|
| -- | -- |
| [Desktop Icons](https://extensions.gnome.org/extension/1465/desktop-icons/) | Иконки на рабочем столе | 
| [KStatusNotifierItem/AppIndicator Support](https://extensions.gnome.org/extension/615/appindicator-support/) | Иконки в трее |
| [Dash to Dock](https://extensions.gnome.org/extension/307/dash-to-dock/) | Выдвигающееся меню с иконками приложений |
| [Clipboard Indicator](https://extensions.gnome.org/extension/779/clipboard-indicator/) | История буфера обмена |
| [Arch Linux Updates Indicator](https://extensions.gnome.org/extension/1010/archlinux-updates-indicator/)| Показывает количество и список пакетов требующих обновления |

## [Gnome Look](https://www.gnome-look.org/)

Это сайт с обоями, курсорами, иконками и темами для Gnome.

```zsh
$ yay -S ocs-url
```

Темы распаковываются в `~/.themes`, курсоры, внезапно, в `~/.icons`, а сами иконки в `~/.local/share/icons`.

* [Yaru Colors](https://www.gnome-look.org/p/1299514/) ‒ одна из дефолтных тем в Ubuntu. Установщик по-умолчанию предложит установить темы и иконки в `/usr/share`. В последствии все придется удалять ручками.

## Telegram

```zsh
# Это версия от разработчиков. Не вижу смысла в левых «опенсорсных» сборках от Васяна
$ yay -S telegram-desktop-bin ttf-opensans
```

## VS Code

```zsh
$ yay -S visual-studio-code-bin ttf-droid ttf-roboto
```

## locate

Позволяет искать файлы по шаблону.

```zsh
$ yay -S mlocate
$ sudo systemctl start updatedb.timer && sudo systemctl enable updatedb.timer
```

## Xclip

Работа с буфером обмена из терминала:

```zsh
$ yay -S xclip

# Копировать содержимое файла
$ xclip -sel c <filename>

# Вывод предыдущей команды
$ <command> | xclip -sel c

# Вывести содержимое
$ xclip -sel c -o
```

## fzf

Универсальный помощник для поиска по командам (`Ctrl + R`) и поиска файлов (`Ctrl + T`) с использованием нечеткого поиска.

```zsh
$ yay -S fzf

# Поиск в текущем каталоге
$ fzf

# Тоже самое, но искать так же скрытые
$ find . | fzf

# И процессов
$ ps aux | fzf

# А это, если хотим копировать в буфер обмена выбранный варинт
$ fzf | xargs echo -n | xclip -sel c
```

## Cht.sh

Справка по языкам программирования и командам:

```zsh
$ yay -S cht.sh
```

## asdf-vm

В процессе разработки может возникнуть необходимость иметь несколько версий того же Python. Обычно для этого используют pyenv, но есть вещь более универсальная, которая служит для управления версиями десятков пакетов.

```ash
$ yay -S asdf-vm
```

В `~/.zshrc` (после compinit, т.е. в конец) добавляем строки:

```zsh
. /opt/asdf-vm/asdf.sh
. /opt/asdf-vm/completions/asdf.bash
```

В `~/.zprofile`:

```zsh
export PATH=/opt/asdf-vm/bin:$PATH
```

Примеры:

```zsh
$ asdf plugin-add python
$ asdf install python 3.8.0
$ asdf global python 3.8.0
# После установки python пакетов, который содержат исполняемые файлы, делает доступными команды
# $ pip install glances
# $ glances
# error
$ asdf reshim python
# $glances
# ok
```

## Настраиваем Vim

Делаем `vi` алиасом `vim` (внезапно, это разные приложения):

```zsh
$ yay -S vi-vim-symlink
```

[SpaceVim](https://spacevim.org/) делает из Vim настоящую IDE и сильно упрощает настройку редактора!!! Если что-то отвалилось (не работает автоподстановка), то смотрим документацию на сайте. Автодополнение по `Ctrl+N`.

Включаем копирование из Vim в системный буфер обмена. Vim должен быть собран с поддержкой `+clipboard`. Проверить это можно, выполнив в vim:

```vim
:echo has('clipboard')
```

Пакет `vim` собран без поддержки `+clipboard`. Установка `gvim` с разрешением замены vim решает эту проблему.

Возвращаемое значение должно быть равно 1.

```zsh
# Создадим скрипт с функцией
$ vim ~/.SpaceVim.d/autoload/myspacevim.vim
func! myspacevim#before() abort
  set clipboard=unnamedplus
endf

# Добавим функцию в автозагрузку
$ vim ~/.SpaceVim.d/init.toml
...
[options]
    ...
    bootstrap_before = "myspacevim#before"
```

# Разное

## Man and Help

```zsh
$ man [ <section> ] <page>

# Section	Description
# 1	General Commands
# 2	System Calls
# 3	Library functions, covering in particular the C standard library
# 4	Special files (usually devices, those found in /dev) and drivers
# 5	File formats and conventions
# 6	Games and screensavers
# 7	Miscellanea
# 8	System administration commands and daemons

# Узнать где хранятся страницы манулов можно так
$ manpath
/usr/local/man:/usr/local/share/man:/usr/share/man

$ man -w printf
/usr/share/man/man1/printf.1.gz

# Поиск страниц по ключевому слову
$ man -k printf

# Смотрим все страницы
$ man -f printf
printf (1)           - format and print data
printf (1p)          - write formatted output
printf (3)           - formatted output conversion
printf (3p)          - print formatted output

# Выбираем конкретную
$ man 1p printf

# Краткая справка по функции
$ command -h
$ command --help
```

## Типы файлов в выывде ls и других стандартных команд

Вам нужно знать только одну команду, которая поможет вам идентифицировать и классифицировать все семь различных типов файлов, имеющихся в Linux.

```zsh
$ ls -ld <file name>
```

Пример вывода:

```zsh
 $ ls -ld /etc/services
-rw-r--r-- 1 root root 19281 Feb 14  2012 /etc/services
```

Команда ls показывает тип файла с помощью одного символа. В данном случае это «-», что означает «обычный файл». Важно отметить, что типы файлов Linux не следует путать с расширениями файлов. Давайте взглянем на краткий обзор всех семи различных типов типов файлов Linux и идентификаторов команды ls:

```
- : regular file
d : directory
c : character device file
b : block device file
s : local socket file
p : named pipe
l : symbolic link
```

## Немного про права

```
4 - Чтение (r) 
2 - Запись (w) 
1 - Выполнение (x)
```

Сумма этих чисел дает разные сочетания типа: 1 + 2 + 4 = 7 или 1 + 4 = 5

Права задаются тремя числами, например, 755, где первое число – права владельца, далее: группа и остальные пользователи. Владелец может делать все (1 + 2 + 4 = 7), другие пользователи – только читать и исполнять файлы (1 + 4 = 5).

Для работы с правами на файлы используется команда chroot:

```zsh
$ chroot --help
```

В Python права можно записывать так:

```python
0o755
```

```zsh
$ ll
total 20K
drwxr-xr-x 3 sergey sergey 4,0K июн 20 17:22 backend
...

d         | rwx      | r-x    | r-x
тип файла | владелец | группа | остальные
```

## Где хранятся исполняемые файлы

| Путь | Описание |
| -- | -- |
| `/bin` | Системные исполняемые файлы (например, ls и cat) |
| `/usr/bin` | Исполняемые файлы, управляемые пакетными менеджерами |
| `/usr/local/bin` | Исполняемые файлы, НЕуправляемые пакетными менеджерами, т.е. тут можно хранить свои скрипты |
| `/opt` | Установленные программы |

## Горячие клавиши терминала

```zsh
$ bindkey -l     
.safe
command
emacs
isearch
listscroll
main
menuselect
vicmd
viins
viopp
visual

# Список клавиш
$ bindkey -M main
"^@" _marker_get
"^A" beginning-of-line
"^B" backward-char
...
```

# Ссылки

* [Btrfs](https://wiki.archlinux.org/index.php/Btrfs)
* [Man page](https://wiki.archlinux.org/index.php/Man_page)
* [Zsh](https://wiki.archlinux.org/index.php/Zsh)
* [MLocate](https://wiki.archlinux.org/index.php/Mlocate)
* [FZF](https://habr.com/ru/company/wrike/blog/344770/)
* [Букварь по Vim](https://ru.wikibooks.org/wiki/Vim)