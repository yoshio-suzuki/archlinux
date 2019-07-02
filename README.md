# Arch Linuxインストール手順

参考：[Arch Linux](https://www.archlinux.org/)

## Pre-installation
---

### Verify signature
* isoダウンロード(未検証)

### Boot the live environment
* isoファイルをddでUSBメディアに書き込み
* BIOS画面でUSBメディアが先に読み込まれるように設定してPC起動

### Set the keyboard layout
* 日本語キーボードの場合
```
# loadkeys jp106
```
* CtrlとCapsの入れ替えはライブ環境では行わない(面倒なので)

### Verify the boot mode
* UEFI環境確認
```
# ls /sys/firmware/efi/efivars
```

### Connect to the internet
* 有線ネットワークを使う(WiFi使うなら設定はGUIでする)
* ネットワーク接続確認(dhcpcdにより自動設定されているはず)
```
# ip a
# ping www.yahoo.co.jp
```
### Update the system clock
* ライブ環境で有効になっているだけなので、インストール環境には再度必要
```
# timedatectl set-ntp true
# timedatectl status
```

### Partition the disks
* SSD(/dev/sda)とHDD(/dev/sdb)の2台がありLVMを使う例
    * HDDにデータ(/home、/var、/tmp、/opt等)を割り当て、起動に必要な領域はSSDに割り当てる
    * スワップパーティションは作らないが、必要ならスワップファイルを使えば良い
    * ハイバーネート用のスワップも作らない(サスペンドを使えばよいし、SSDなら早いのでハイバーネート不要)
* パーティション設定

|デバイス|パーティション|サイズ|タイプ|
|-|-|-|-|
|/dev/sda|/dev/sda1|256M|EFI|
|〃|/dev/sda2|残り全て|LVM|
|/dev/sdb|/dev/sdb1|全て|LVM|

```
# lsblk
# fdisk デバイス
```

* LVM設定

|PV|デバイス|
|-|-|
|/dev/sda2|/dev/sda2|
|/dev/sdb1|/dev/sdb1|

```
# pvcreate デバイス
# pvdisplay
```

|VG|PV|
|-|-|
|vg_ssd|/dev/sda2|
|vg_hdd|/dev/sdb1|

```
# vgcreate VG名 PV名
# vgdisplay
```

|LV|VG|サイズ|
|-|-|-|
|lv_root|vg_ssd|50G|
|lv_home|vg_hdd|500G|
|lv_var|vg_hdd|20G|
|lv_tmp|vg_hdd|10G|
|lv_opt|vg_hdd|1G|

```
# lvcreate -L サイズ VG名 -n LV名
# lvdisplay
```

### Format the partitions
* EFIはFAT32で、それ以外のLVはEXT4でフォーマットする
```
# mkfs.fat -F32 /dev/sda1
# mkfs.ext4 /dev/VG名/LV名
```

### Mount the file systems
```
# mount /dev/vg_ssd/lv_root /mnt
# mkdir /mnt/{boot,home,var,tmp,opt}
# mount /dev/sda1 /mnt/boot
# mount /dev/vg_hdd/lv_home /mnt/home
# mount /dev/vg_hdd/lv_var /mnt/var
# mount /dev/vg_hdd/lv_tmp /mnt/tmp
# mount /dev/vg_hdd/lv_opt /mnt/opt
```

## Installation
---

### Select the mirrors
* 日本のミラーサイト(3つある)を先頭にする

```
# vi /etc/pacman.d/mirrorlist
日本ミラーサイト

```

### Install the base packages
* base-devも追加
```
# pacstrap /mnt base base-dev
```

## Configure the system
---

### Fstab
```
# genfstab -U /mnt >> /mnt/etc/fstab
```

### Chroot
```
# arch-chroot /mnt
```

### Time zone
* `ln`ではなく`timedatectl set-timezone Asia/Tokyo`ではダメか？
```
# ln -sf /usr/share/zoneinfo/Asia/Tokyo /etc/localtime
# hwclock --systohc
```

### Localization
* en_US.UTF-8とja_JP.UTF-8のコメント(#)を取り除き、それを生成する
```
# vi /etc/locale.gen
アンコメント

# locale-gen
```

* LANGはen_US.UTF-8にすること(さもないとフォルダ名が日本語になってしまう？)
```
echo "LANG=en_US.UTF-8" > /etc/locale.conf
```

* キーマップ設定
```
# mkdir -p /usr/local/share/kbd/keymaps
# vi /usr/local/share/kbd/keymaps/personal.map
include "/usr/share/kbd/keymaps/i386/qwerty/jp106.map.gz"
keycode 58 = Control

# echo "KEYMAP=/usr/local/share/kbd/keymaps/personal.map" > /etc/vconsole.conf
# loadkeys /usr/local/share/kbd/keymaps/personal.map
```

### Network configuration
* /etc/hostnameを直接編集しなくてもよいはず
```
# hostnamectl set-hostname ホスト名
# vi /etc/hosts
127.0.0.1  localhost
::1        localhost
127.0.1.1  ホスト名.localdomain  ホスト名
```

### Initramfs
* HOOKSにkeymap(/etc/vconsole.conf用)とlvm2(LVMなので)を追加する
* systemdではなくudevを使うこと(さもないと起動時に時々LVMが失敗する)
```
# vi /etc/mkinitcpio.conf
フック修正
HOOKS=(base udev autodetect modconf block lvm2 filesystems keyboard keymap fsck)

# mkinitcpio -p linux
```

### Root password
```
# passwd
```

### Boot loader
* systemd-bootを使う
* Intelプロセッサなので、Intel用マイクロコードを追加
* systemd-boot更新フックは後で追加
```
# bootctl --path=/boot install
# pacman -S intel-ucode
```

* 設定
* カーネルパラメータは要調整
```
# vi /boot/loader/loader.conf
default arch
timeout 2
console-mode max
editor no

# vi /boot/loader/entries/arch.conf
title Arch Linux
linux /vmlinuz-linux
initrd /intel-ucode.img
initrd /initramfs-linux.img
options root=/dev/vg_ssd/lv_root rw nowatchdog
```

## Reboot
---

* chrootから`Ctrl-D`で抜け、`umount -R /mnt`でアンマウントして、`reboot`で再起動(USBメディアは取り除くこと)

## Post-installation
---

### ネットワーク一時設定
* gnomeインストールまではNetworkManagerが入っていないのでdhcpcdを使う
```
# systemctl start dhcpcd
# ip a
# ping www.yahoo.co.jp
```

### 各種設定
* パッケージビルド時のプロセス数
```
# vi /etc/makepkg.conf
プロセス数
MAKEFLAGS="-j$(nproc)"
```

* pacman色付け
```
# vi /etc/pacman.conf
アンコメント
Color
```

* 時刻同期(ステータスがactive、タイムゾーンがJSTになること)
```
# timedatectl set-ntp true
# timedatectl status
```

* SSD TRIM
```
# systemctl enable fstrim.timer
# systemctl start fstrim.timer
```

* journaldログサイズ
```
vi /etc/systemd/journald.conf
アンコメントしてサイズ指定
SystemMaxUse=50M
```

### gnomeインストール
* ログイン時にWaylandを選ばないようにする
```
# pacman -S gnome
# vi /etc/gdm/custom.conf
アンコメント
WaylandEnable=false

# systemctl enable gdm
```

### ネットワーク設定変更
* dhcpcdからNetworkManagerに変更
```
# systemctl stop dhcpcd
# systemctl enable NetworkManager
# systemctl start NetworkManager
# ip a
# ping www.yahoo.co.jp
```

### AURヘルパー
* yay追加
```
# pacman -S git
# cd /tmp
# git clone https://aur.archlinux.org/yay.git
# cd yay
# makepkg -si
```

### ユーザ設定
* グループをusersにしたいのでlogin.defsを修正する
```
# vi /etc/login.defs
アンコメント
USERGROUPS_ENAB no
```

* wheelグループはパスなしでsudoできるようにする
```
# visudo
アンコメント
%wheel ALL=(ALL) NOPASSWD: ALL
```

* ユーザ追加
```
# useradd -m ユーザ名
# usermod -aG wheel ユーザ名
# passwd ユーザ名
```

### ユーザ切り替え
* 以降の作業は一般ユーザで行う
```
# su - ユーザ名
```

### systemd-boot更新フック追加
```
$ yay -S systemd-boot-pacman-hook
```

### GNOME起動
* 以降の作業はGUIで行う
```
$ sudo systemctl start gdm
```

### 各種パッケージ追加
* Caps-Ctrl入れ替え
```
$ yay -S gnome-tweaks
Tweeksを起動 > Keyboard&Mouse > Additional Layout Options > Caps Lock behavior > Caps Lock is also a Ctrl
```

* ターミナル透明化
```
$ yay -S gnome-terminal-transparency
既存のgnome-terminalを削除・置き換えする
設定はお好みで(テーマDark、透明化、パレットSolarized)
```

* コンソール
```
$ yay -S vim screen
$ vim ~/.vimrc
syntax on
set t_ti= t_te=

$ vim ~/.screenrc
https://qiita.com/kamykn/items/9939b67e923dbb87f39c

$ vim ~/.bashrc
エイリアス追加
alias ll='ls -la'
alias less='less -XR'
alias vi='vim'
```

* 状態取得
```
$ yay -S hdparm smartmontools lm_sensors htop iotop ddcutil i2c-tools
$ echo "i2c-dev" > /etc/modules-load.d/i2c-dev.conf
$ sudo modprobe i2c-dev
$ sudo sensors-detect
全てEnter
```

* Hardware video acceleration
```
$ yay -S intel-media-driver libva-utils
$ LIBVA_DRIVER_NAME=iHD vainfo
$ sudo vim /etc/environment
追加
LIBVA_DRIVER_NAME=iHD
```

* Intel graphics
```
$ sudo vim /boot/loader/entries/arch.conf
オプション追加
options ... i915.enable_fbc=1 i915.fastboot=1
```

* OpenCL
```
$ yay -S intel-compute-runtime clinfo
$ clinfo
```

* ファイル
```
$ yay -S ntfs-3g exfat-utils rsync
```

* 日本語
```
$ yay -S otf-ipafont noto-fonts-cjk noto-fonts-emoji
$ yay -S fcitx-mozc fcitx-gtk3 fcitx-configtool
Fcitx Config > Input Method > Add input method > Only Show Current Language:アンチェック > Keybod - JapaneseとMozcを追加(Keyboard - English(US)は削除)
$ sudo vim /etc/environment
追加
GTK_IM_MODULE=fcitx
XMODIFIERS=@im=fcitx
```

* Google Chrome
```
$ yay -S google-chrome
``` 

* マルチメディア
```
$ yay -S youtube-dl mpv feh inkscape
$ sudo vim /etc/mpv/mpv.conf
vo=vaapi
hwdec=vaapi
```

* 開発
```
$ yay -S visual-studio-code-bin slack-desktop
$ yay -S docker-compose
$ sudo systemctl enable docker
$ sudo systemctl start docker
$ sudo usermod -aG docker ユーザ名
```

* 省電力
```
$ sudo vim /etc/modprobe.d/iwlwifi.conf
options iwlwifi power_save=1 d0i3_disable=0 uapsd_disable=0
options iwldvm force_cam=0
```

* モニタ設定変更時はmonitors.xmlを毎回コピー
```
$ sudo cp ~/.config/monitors.xml /var/lib/gdm/.config/monitors.xml
$ sudo chown gdm.dgm /var/lib/gdm/.config/monitors.xml
```

## NVIDIA
---

### ドライバインストール
* インストール後再起動
```
$ yay -S nvidia
```

* モニタ設定変更時はmonitors.xmlを毎回コピー
```
$ sudo cp ~/.config/monitors.xml /var/lib/gdm/.config/monitors.xml
$ sudo chown gdm.dgm /var/lib/gdm/.config/monitors.xml
```

## QEMU/KBM
---
```
$ yay -S gnome-xrandr
```
