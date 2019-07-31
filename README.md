# Arch Linuxインストール手順

* 参考：[Arch Linux](https://www.archlinux.org/)
* Myマシン構成：
    * M/B: MSi Z370I GAMING PRO CARBON AC
    * CPU: i5-8400
    * MEM: 16GB
    * SSD: 360GB
    * HDD: 1TB
    * DGP: GTX1070

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
options root=/dev/vg_ssd/lv_root rw
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
# systemctl enable gdm
# vi /etc/gdm/custom.conf
アンコメント
WaylandEnable=false
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
参考
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

* WiFi無効化
```
$ sudo vim /etc/modprobe.d/iwlwifi.conf
blacklist iwlwifi
```

* Bluetooth無効化
```
$ sudo vim /etc/modprobe.d/bluetooth.conf
blacklist btusb
blacklist bluetooth
```

* モニタ設定変更時はmonitors.xmlを毎回コピー
```
$ sudo cp ~/.config/monitors.xml /var/lib/gdm/.config/monitors.xml
$ sudo chown gdm.dgm /var/lib/gdm/.config/monitors.xml
```

## NVIDIA
---

### Xorg
* NVIDIAグラフィックカードを表示に使わない場合（CUDA等）、Intelのみを使うように明示
```
$ lspci | grep VGA | grep -i intel
$ sudo vim /etc/X11/xorg.conf.d/20-intel.conf
Section "Device"
   Identifier  "Intel Graphics"
   Driver      "modesetting"
   Option      "AccelMethod" "glamor"
   BusID       lspciして確認する(例えば"PCI:0:2:0")
EndSection
```

### ドライバインストール
* インストール後再起動
```
$ yay -S nvidia
```

## QEMU/KVM
---

### ゴール
* VM上でWindowsを実行
* DGP、USBをPCIパススルー(vfio)
    * BluetoothはPCIではなくUSBホストデバイスをパススルーする
* DGPを普段はホスト側で利用して、VM起動時にはVM側に動的に割り当て(unbind/bind)
    * 手順確立するまではVM専用設定とする
* ホストとVMでディレクトリ共有(samba/iscsi)
    * PHOTOfunSTUDIOがネットワークドライブへの保存に対応していないため、iSCSIで対応(ホスト側からはReadOnlyマウントして同時書込のないようにする)
* マウス/キーボードを共有(evdev)
* 1つのモニタでマウス/キーボードに連動して入力ソース切り替え(ddcutil/xrandr)
    * MyモニタがDDC/CIに対応していないので、xrandrで対応

### LTS版Kernelに変更
* LTS版Kernelインストール
* LTS版でないとVMが正常動作しなかった
```
$ yay -S kernel-lts
```

* 設定
```
# vi /boot/loader/loader.conf
変更
default arch-lts

$ sudo vim /boot/loader/entries/arch-lts.conf 
title Arch Linux - LTS Kernel
linux /vmlinuz-linux-lts
initrd /intel-ucode.img
initrd /initramfs-linux-lts.img
options root=/dev/vg_ssd/lv_root rw i915.enable_fbc=1 i915.fastboot=1
```

### LTS版NVIDIAドライバインストール
```
$ yay -S nvidia-lts
```

### IOMMU有効化
* BIOSでVT-dを有効にして、カーネルパラメータを設定後、再起動
```
$ sudo vim /boot/loader/entries/arch-lts.conf
オプション追加
options ... intel_iommu=on iommu=pt
```

### IOMMUグループ確認
* IOMMUグループ確認
```
$ for d in /sys/kernel/iommu_groups/*/devices/*; do
    n=${d#*/iommu_groups/*}; n=${n%%/*}
    printf 'IOMMU Group %s ' "$n"
    lspci -nns "${d##*/}"
done
```

* My出力
* GDPはIOMMUグループ1、USBコントローラはグループ4と13と、グループ内はパススルー対象デバイスだけにまとまっている
* 同一グループ内にパススルーしたくないデバイスが混じっている場合、ACS上書きパッチで対応できる可能性がある
* グループ1のPCIブリッジは、vfioにバインドしたりVMに追加しないこと
```
IOMMU Group 0 00:00.0 Host bridge [0600]: Intel Corporation 8th Gen Core Processor Host Bridge/DRAM Registers [8086:3ec2] (rev 07)
IOMMU Group 10 00:1d.3 PCI bridge [0604]: Intel Corporation 200 Series PCH PCI Express Root Port #12 [8086:a29b] (rev f0)
IOMMU Group 11 00:1f.0 ISA bridge [0601]: Intel Corporation Z370 Chipset LPC/eSPI Controller [8086:a2c9]
IOMMU Group 11 00:1f.2 Memory controller [0580]: Intel Corporation 200 Series/Z370 Chipset Family Power Management Controller [8086:a2a1]
IOMMU Group 11 00:1f.3 Audio device [0403]: Intel Corporation 200 Series PCH HD Audio [8086:a2f0]
IOMMU Group 11 00:1f.4 SMBus [0c05]: Intel Corporation 200 Series/Z370 Chipset Family SMBus Controller [8086:a2a3]
IOMMU Group 12 00:1f.6 Ethernet controller [0200]: Intel Corporation Ethernet Connection (2) I219-V [8086:15b8]
IOMMU Group 13 03:00.0 USB controller [0c03]: ASMedia Technology Inc. ASM2142 USB 3.1 Host Controller [1b21:2142]
IOMMU Group 14 05:00.0 Network controller [0280]: Intel Corporation Wireless 8265 / 8275 [8086:24fd] (rev 78)
IOMMU Group 1 00:01.0 PCI bridge [0604]: Intel Corporation Xeon E3-1200 v5/E3-1500 v5/6th Gen Core Processor PCIe Controller (x16) [8086:1901] (rev 07)
IOMMU Group 1 01:00.0 VGA compatible controller [0300]: NVIDIA Corporation GP104 [GeForce GTX 1070] [10de:1b81] (rev a1)
IOMMU Group 1 01:00.1 Audio device [0403]: NVIDIA Corporation GP104 High Definition Audio Controller [10de:10f0] (rev a1)
IOMMU Group 2 00:02.0 VGA compatible controller [0300]: Intel Corporation UHD Graphics 630 (Desktop) [8086:3e92]
IOMMU Group 3 00:08.0 System peripheral [0880]: Intel Corporation Xeon E3-1200 v5/v6 / E3-1500 v5 / 6th/7th Gen Core Processor Gaussian Mixture Model [8086:1911]
IOMMU Group 4 00:14.0 USB controller [0c03]: Intel Corporation 200 Series/Z370 Chipset Family USB 3.0 xHCI Controller [8086:a2af]
IOMMU Group 4 00:14.2 Signal processing controller [1180]: Intel Corporation 200 Series PCH Thermal Subsystem [8086:a2b1]
IOMMU Group 5 00:16.0 Communication controller [0780]: Intel Corporation 200 Series PCH CSME HECI #1 [8086:a2ba]
IOMMU Group 6 00:17.0 SATA controller [0106]: Intel Corporation 200 Series PCH SATA controller [AHCI mode] [8086:a282]
IOMMU Group 7 00:1c.0 PCI bridge [0604]: Intel Corporation 200 Series PCH PCI Express Root Port #1 [8086:a290] (rev f0)
IOMMU Group 8 00:1c.6 PCI bridge [0604]: Intel Corporation 200 Series PCH PCI Express Root Port #7 [8086:a296] (rev f0)
IOMMU Group 9 00:1d.0 PCI bridge [0604]: Intel Corporation 200 Series PCH PCI Express Root Port #9 [8086:a298] (rev f0)
```

* リセット対応確認
```
$ for iommu_group in $(find /sys/kernel/iommu_groups/ -maxdepth 1 -mindepth 1 -type d); do
    echo "IOMMU group $(basename "$iommu_group")"
    for device in $(\ls -1 "$iommu_group"/devices/); do
        if [[ -e "$iommu_group"/devices/"$device"/reset ]]; then
            echo -n "[RESET]"
        fi
        echo -n $'\t';lspci -nns "$device"
    done
done
```

* My出力
* 03:00.0のUSBコントローラはリセット対応しているが、00:14.0は対応しておらず、したがって問題なくパススルーできるIOMMUグループ13のUSBコントローラを使う
```
IOMMU group 7
[RESET] 00:1c.0 PCI bridge [0604]: Intel Corporation 200 Series PCH PCI Express Root Port #1 [8086:a290] (rev f0)
IOMMU group 5
        00:16.0 Communication controller [0780]: Intel Corporation 200 Series PCH CSME HECI #1 [8086:a2ba]
IOMMU group 13
[RESET] 03:00.0 USB controller [0c03]: ASMedia Technology Inc. ASM2142 USB 3.1 Host Controller [1b21:2142]
IOMMU group 3
[RESET] 00:08.0 System peripheral [0880]: Intel Corporation Xeon E3-1200 v5/v6 / E3-1500 v5 / 6th/7th Gen Core Processor Gaussian Mixture Model [8086:1911]
IOMMU group 11
        00:1f.0 ISA bridge [0601]: Intel Corporation Z370 Chipset LPC/eSPI Controller [8086:a2c9]
        00:1f.2 Memory controller [0580]: Intel Corporation 200 Series/Z370 Chipset Family Power Management Controller [8086:a2a1]
        00:1f.3 Audio device [0403]: Intel Corporation 200 Series PCH HD Audio [8086:a2f0]
        00:1f.4 SMBus [0c05]: Intel Corporation 200 Series/Z370 Chipset Family SMBus Controller [8086:a2a3]
IOMMU group 1
        00:01.0 PCI bridge [0604]: Intel Corporation Xeon E3-1200 v5/E3-1500 v5/6th Gen Core Processor PCIe Controller (x16) [8086:1901] (rev 07)
[RESET] 01:00.0 VGA compatible controller [0300]: NVIDIA Corporation GP104 [GeForce GTX 1070] [10de:1b81] (rev a1)
        01:00.1 Audio device [0403]: NVIDIA Corporation GP104 High Definition Audio Controller [10de:10f0] (rev a1)
IOMMU group 8
[RESET] 00:1c.6 PCI bridge [0604]: Intel Corporation 200 Series PCH PCI Express Root Port #7 [8086:a296] (rev f0)
IOMMU group 6
        00:17.0 SATA controller [0106]: Intel Corporation 200 Series PCH SATA controller [AHCI mode] [8086:a282]
IOMMU group 14
[RESET] 05:00.0 Network controller [0280]: Intel Corporation Wireless 8265 / 8275 [8086:24fd] (rev 78)
IOMMU group 4
        00:14.0 USB controller [0c03]: Intel Corporation 200 Series/Z370 Chipset Family USB 3.0 xHCI Controller [8086:a2af]
        00:14.2 Signal processing controller [1180]: Intel Corporation 200 Series PCH Thermal Subsystem [8086:a2b1]
IOMMU group 12
[RESET] 00:1f.6 Ethernet controller [0200]: Intel Corporation Ethernet Connection (2) I219-V [8086:15b8]
IOMMU group 2
[RESET] 00:02.0 VGA compatible controller [0300]: Intel Corporation UHD Graphics 630 (Desktop) [8086:3e92]
IOMMU group 10
[RESET] 00:1d.3 PCI bridge [0604]: Intel Corporation 200 Series PCH PCI Express Root Port #12 [8086:a29b] (rev f0)
IOMMU group 0
        00:00.0 Host bridge [0600]: Intel Corporation 8th Gen Core Processor Host Bridge/DRAM Registers [8086:3ec2] (rev 07)
IOMMU group 9
[RESET] 00:1d.0 PCI bridge [0604]: Intel Corporation 200 Series PCH PCI Express Root Port #9 [8086:a298] (rev f0)
```

### デバイス接続計画
* USBコントローラが2つあることがポイントで、リセット対応コントローラをVMに割り当てる
* 何故かUSB-HUBを使うとPCがダウンしやすいが、パススルーするコントローラにはポートが2つしか無く仕方ない

|コントローラ|IOMMU|IF|VM|接続デバイス|
|-|-|-|-|-|
|Z370 USB 3.0 xHCI|4|2.0TypeA|-|キーボード/マウス(Unifying)|
|Z370 USB 3.0 xHCI|4|3.1Gen1TypeA|-|外付けHDD|
|ASM2142 USB 3.1|13|3.1Gen2TypeC|Win|HUB経由でゲームパッド等|
|ASM2142 USB 3.1|13|3.1Gen2TypeA|Win|HMD|
|GeForce GTX 1070|1|HDMI|Win|HMD|
|GeForce GTX 1070|1|DP|Win|モニタ|
|UHD Graphics 630|2|DP|-|モニタ|
|UHD Graphics 630|2|HDMI|-|AVアンプ/TV|

### Bluescreen at boot since Windows 10 1803
```
$ sudo vim /etc/modprobe.d/kvm.conf
options kvm ignore_msrs=1
```

### DGP分離
* DGP分離は動的に行いたいが、手順が確立するまでは取り敢えず静的に行う
    * DGPをunbindすると固まってしまうので動的にできない(kernelのバグか?)
* IOMMUグループ確認時に表示されたDGPとそれに付随するオーディオのPCIデバイスIDを設定する
```
$ sudo vim /etc/modprobe.d/vfio.conf
options vfio-pci ids=DGPのID,オーディオのID
```

* 起動時にロードされるように、mkinitcpio.confに設定
```
$ sudo vim /etc/mkinitcpio.conf
他のグラフィックドライバーより前に記述すること
MODULES=(vfio_pci vfio vfio_iommu_type1 vfio_virqfd ...)
```

* initramfs再生成(要再起動)
```
$ sudo mkinitcpio -p linux-lts
```

### 関連パッケージインストール
* qemu、libvirt、bridge-utilsはインストール済みになっているかも
```
$ yay -S qemu ovmf libvirt virt-manager dnsmasq ebtables dmidecode bridge-utils gnome-xrandr
```

### VM用ディスクイメージ
* ディスクイメージはqcow2ではなくLVMパーティションを使い、OSはSSD、データはHDDに割り当てる

|LV|VG|サイズ|
|-|-|-|
|lv_win10os|vg_ssd|100G|
|lv_win10data|vg_hdd|100G|

```
$ sudo lvcreate -L サイズ VG名 -n LV名
$ sudo lvdisplay
```

### グループ追加
```
$ sudo usermod -aG input ユーザ名
$ sudo usermod -aG kvm ユーザ名
$ sudo usermod -aG libvirt ユーザ名
```

### libvirt設定
* libvirt設定
```
$ sudo vim /etc/libvirt/qemu.conf
コメントアウトされている箇所の下に追加
nvram = [
    "/usr/share/ovmf/x64/OVMF_CODE.fd:/usr/share/ovmf/x64/OVMF_VARS.fd"
]
```

* libvirtサービス起動
```
$ sudo systemctl enable libvirtd
$ sudo systemctl start libvirtd
$ sudo systemctl enable virtlogd.socket
```

### VM新規追加

* インストール・ドライバメディア(ISOファイル)を`/var/lib/libvirt/images/`に事前保存
    * Windows10のディスクイメージ(ISOファイル)はMicrosoft公式ページからダウンロード(Win10_1809Oct_v2_Japanese_x64.iso)
        * バージョンが新しい(1903)と問題が起こりやすいかも
        * 自動的にWindows Updateされないよう、ネットワーク未接続で作業する
    * VirtIOドライバは下記のfedora公式ページからStable版をダウンロード(virtio-win-0.1.141.iso): https://docs.fedoraproject.org/en-US/quick-docs/creating-windows-virtual-machines-using-virtio-drivers/index.html
        * Latest版を使う必要はない
    * NVIDIAドライバは公式ページからダウンロードしたものをISO化(nvidia.iso)する
        * 自動的にドライバインストールされないようネットワーク未接続で作業するので、ドライバは事前ダウンロードしておく
```
$ mkisofs -o nvidia.iso NVIDIAドライバファイル
```

* virt-managerを起動し、下記のとおり設定して、Begin Installationからインスール実施("Manually set CPU topology"は「VM only uses one core」対策)
* 設定値は環境による
    * Local install media(ISO)
    * Windows10のISOを選択
    * Memory: 8192MiB、CPUs: 4
    * Storage: /dev/vg_ssd/lv_win10os
    * Customize configuration before installにチェック
    * Add Hardware
        * Storage > /var/vg_hdd/lv_win10data, Bus Type: VirtIO
        * Storage > Device type: CD-ROM
        * Controller > Type: SCSI, Model: VirtIO SCSI
    * Overview:
        * Chipset: Q35
        * Firmware: UEFI
    * CPUs:
        * Copy host CPU configurationのチェックを外す
        * Model: host-passthrough
        * Manually set CPU topologyをチェック
            * Sokets: 1
            * Cores: 4
            * Threads: 1
        * Current allocation: 4
    * Boot Options:
        * SATA CDROM1にチェック
    * SATA Disk1:
        * Disk bus: SCSI
    * SATA CDROM2:
        * Source path: ダウンロードしたVirtIOメディア(virtio-win-0.1.141.iso)
    * NIC:
        * Device model: virtio

### Windows10インストール
* Windows Updateを無効にするために、Pro版をインストールする
* インストール時に、OS用ストレージのSCSIドライバが無くてドライブが見つからないので、ドライバの読み込みからインストールするドライバを選ぶ(ドライバメディアの/vioscsi/w10/amd64フォルダにある)
    * データ用ストレージはWindowsのセットアップが全て完了して、必要になってからドライバ追加(ドライバメディアの/viostor/w10/amd64フォルダにある)して利用すればよい
    * Bus TypeはSCSIよりVirtIOのほうがIO性能が1割ほど高かった(CrystalDiskMark)ので、OS用ストレージもVirtIOにすれば良かったかもしれない(SSDなのでそこまで気にしない)
* インストール中はNIC用VirtIOドライバがないためネットワーク接続できない
* インストール後は、ネットワーク接続する前に、Windows Updateを無効にする
    * gpedit.msc実行 > コンピューターの構成 > 管理用テンプレート > Windowsコンポーネント > Windows Update > 自動更新を構成する > 有効 > 2-ダウンロードと自動インストールを通知 > OK

### VM追加設定
* Bluetoothデバイス確認
```
$ lsusb
```

* 一旦VMをオフして、VMの追加設定をする
    * Add Hardware
        * PCI Host Device(IOMMUグループ確認したパススルーするデバイスを追加)
            * DGP
            * DGPに付随するオーディオ
            * リセット対応USBコントローラ
        * USB Host Device
            * Bluetoothデバイス
    * SATA CDROM1:
        * Source path: ISO化したNVIDIAドライバ(nvidia.iso)

### "Error 43: Driver failed to load" on Nvidia GPUs passed to Windows VMs
```
$ sudo virsh edit VM名
該当箇所に追加
<features>
  <hyperv>
    <vendor_id state='on' value='whatever'/>
    ...
  </hyperv>
  ...
  <kvm>
    <hidden state='on'/>
  </kvm>
</features>
```

### QEMU 4.0: Unable to load graphics drivers/BSOD after driver install using Q35
```
$ sudo virsh edit VM名
該当箇所に追加
<features>
  ...
  <ioapic driver='kvm'/>
</features>
```

### Slowed down audio pumped through HDMI on the video card
* DGPとオーディオデバイスのMSI(Message Signaled-Based Interrupts)を有効にする
    1. Win10側で、デバイスマネージャー > 表示 > リソース(種類別) > 割り込み要求(IRQ) > (PCI)0x...(xx)の対象デバイス(xxがマイナス値なら既にMSI) > 右クリック > プロパティ > 詳細 > デバイスインスタンスパス確認
    2. regedit > HKEY_LOCAL_MACHINE > SYSTEM > CurrentControlSet > Enum > デバイスインスタンスパス > Device Parameters > Interrupt Management > MessageSignaledInterruptPropertiesキー追加(既存の場合もある) > MSISupported値(DWORD)に1をセット
    3. Win10を再起動し、デバイスマネージャーから対象デバイスがマイナス値になっていることを確認する

### CPU pinning
* 設定値は環境による
```
<vcpu placement='static'>4</vcpu>
該当箇所に追加
<iothreads>1</iothreads>
<cputune>
    <vcpupin vcpu='0' cpuset='2'/>
    <vcpupin vcpu='1' cpuset='3'/>
    <vcpupin vcpu='2' cpuset='4'/>
    <vcpupin vcpu='3' cpuset='5'/>
    <emulatorpin cpuset='0,1'/>
    <iothreadpin iothread='1' cpuset='0,1'/>
</cputune>
```

### NVIDIAドライバ追加
* 再びVMを起動して、CD-ROMのNVIDIAドライバをインストールする
* NVIDIAコントロールパネルから、PhysX設定をDGPに明示的に設定(nvlddmkmエラー対策)

### 仮想デバイス削除
* ここまでVMが正常に機能していることが確認できたら、一旦VMをオフして、VMの不要デバイスを削除する
    * Tablet: Remove
    * Display Spice: Remove
    * Sound: Remove
    * Serial: Remove
    * Channel spice: Remove
    * Video QXL: Remove
    * Controller VirtIO Serial: Remove
    * USB Redirector: Remove

### マウス/キーボード共有
* `/dev/input/by-id/`以下から、キーボードとマウスのIDを検索して、`cat /dev/input/by-id/デバイス名`してみて何か出力されたら当たり
```
$ ls -l /dev/input/by-id/
$ cat /dev/input/by-id/キーボード名
$ cat /dev/input/by-id/マウス名
```

* VM設定
```
$ sudo virsh edit VM名
先頭行書き換え
<domain type='kvm' xmlns:qemu='http://libvirt.org/schemas/domain/qemu/1.0'>

末尾に追加
  <qemu:commandline>
    <qemu:arg value='-object'/>
    <qemu:arg value='input-linux,id=mouse1,evdev=/dev/input/by-id/マウス名'/>
    <qemu:arg value='-object'/>
    <qemu:arg value='input-linux,id=kbd1,evdev=/dev/input/by-id/キーボード名,grab_all=on,repeat=on'/>
  </qemu:commandline>
 </domain>
```

* QEMU設定
```
$ sudo vim /etc/libvirt/qemu.conf
コメントアウトされている箇所の下に追加
user = "ユーザ名"

コメントアウトされている箇所の下に追加
group = "kvm"

コメントアウトされている箇所の下に追加
cgroup_device_acl = [
    "/dev/input/by-id/キーボード名",
    "/dev/input/by-id/マウス名",
    "/dev/null", "/dev/full", "/dev/zero",
    "/dev/random", "/dev/urandom",
    "/dev/ptmx", "/dev/kvm",
    "/dev/rtc","/dev/hpet"
]
```

### モニタ切替
* 切替はアクティブなディスプレイを一時的にOFFすることで、他方に切替える仕組み
    * `ddcutil`が利用できれば、もっとスマートな切替が可能
* 既存の環境変数DISPLAYとXAUTHORITYを確認
```
$ env
```

* VM起動時のフック作成
```
$ sudo mkdir /etc/libvirt/hooks
$ sudo vim /etc/libvirt/hooks/qemu
#!/bin/bash

display_switch () {
        export DISPLAY=上記環境変数に合わせる
        export XAUTHORITY=上記環境変数に合わせる
        display=$(xrandr | grep -E " connected (primary )?[1-9]+" | sed -e "s/\([A-Z0-9]\+\) connected.*/\1/")
        xrandr --output $display --off && \
        sleep 5 && \
        xrandr --output $display --auto
}

if [[ $1 == "win10" ]]; then
    if [[ $2 == "started" ]]; then
        display_switch
        nohup /etc/libvirt/hooks/display_switch.py 0<&- &>/dev/null &
    elif [[ $2 == "stopped" ]]; then
        pkill -f display_switch.py
    fi
elif [[ $1 == "display_switch" ]]; then
    display_switch
fi
```

* ホスト側の切替機構
```
$ yay -S python-evdev
$ sudo vim /etc/libvirt/hooks/display_switch.py 
#!/usr/bin/env python
import asyncio
from evdev import InputDevice

dev = InputDevice('/dev/input/by-id/usb-Logitech_USB_Receiver-if02-event-kbd')

async def helper(dev):
    both = 0
    async for ev in dev.async_read_loop():
        if ev.type == 1 and (ev.code == 29 or ev.code == 97):
            if ev.value == 1:
                both += 1
                if both == 2:
                    process = await asyncio.create_subprocess_exec('/etc/libvirt/hooks/qemu', *('display_switch',))
            elif ev.value == 0:
                both -= 1

loop = asyncio.get_event_loop()
loop.run_until_complete(helper(dev))
```

* 実行権限付与
```
$ sudo chmod 755 /etc/libvirt/hooks/qemu
$ sudo chmod 755 /etc/libvirt/hooks/display_switch.py
```

* スケール設定
    * スケールが100%のままなら設定不要
    * xrandrで復帰したときに、Settings > Devices > Displays > Scaleで設定した倍率が戻ってしまうので、gsettingsで設定しておく
```
$ gsettings set org.gnome.settings-daemon.plugins.xsettings overrides "[{'Gdk/WindowScalingFactor', <2>}]"
$ gsettings set org.gnome.desktop.interface scaling-factor 2
```

* VM側の切替機構
    * AutoHotkeyアプリでLCtrl&RCtrlのホットキーでBAT起動できるようにする
    * `display_switch.ahk`をスタートアップ/タスクスケジューラで自動起動させる
```
> Notepad.exe %USERPROFILE%\bin\display_switch.bat
powershell (Add-Type '[DllImport(\"user32.dll\")]^public static extern int SendMessage(int hWnd, int hMsg, int wParam, int lParam);' -Name a -Pas)::SendMessage(-1,0x0112,0xF170,2)
powershell Start-Sleep -s 5
powershell (Add-Type '[DllImport(\"user32.dll\")]^public static extern void mouse_event(Int32 dwFlags, Int32 dx, Int32 dy, Int32 dwData, UIntPtr dwExtraInfo);' -Name a -Pas)::mouse_event(1,0,1,0,[UIntPtr]::Zero)

> Notepad.exe %USERPROFILE%\bin\display_switch.ahk 
#SingleInstance Ignore
LCtrl & RCtrl::
RCtrl & LCtrl::
Run %USERPROFILE%\bin\display_switch.bat
return
```

### Windows10カスタマイズ
* 一旦ここでバックアップを取っておいても良いかも
* libvirtd再起動後、VM起動
* Ctrl左右両押しでマウス/キーボードが切り替わる
* Windows軽量化
* データ用ストレージ(VirtIO)のドライバ更新して(ドライバメディアのRed Hat VirtIO SCSI controller)、NTFSフォーマットして利用可能にする
    * 主にSteamゲームインストール先だが、ディスクIO性能によりロードが時間がかかりすぎるのであれば、OS用ストレージ(SSD)にフォルダ移動する
* ネットワークアダプタ(virtio)のドライバ更新する(ドライバメディアを参照すればVirtIO Ethernet Adapterが出てくる)

### Sambaフォルダ共有
* パッケージインストール
```
$ yay -S samba
```

* 設定ファイル作成
    * `read only = no`のコメントアウトは必要に応じて切り替える
```
$ sudo vim /etc/samba/smb.con
[global]
   dos charset = CP932 
   unix charset = UTF-8 
   server multi channel support = yes
   socket options = IPTOS_THROUGHPUT SO_KEEPALIVE
   deadtime = 30
   use sendfile = Yes
   write cache size = 262144
   min receivefile size = 16384
   aio read size = 16384
   aio write size = 16384
   load printers = no
   printcap name = /dev/null
   disable spoolss = yes
   workgroup = WORKGROUP
   server string = Samba Server
   server role = standalone server
   log file = /var/log/samba/%m.log
   max log size = 50
   dns proxy = no 

[homes]
   comment = Home Directories
   browseable = no
   valid users = %S
   #read only = no
```

* サービス起動
```
$ sudo systemctl start smb
$ sudo systemctl enable smb
```

* sambaユーザ作成
```
$ sudo smbpasswd -a ユーザ名
```

* VM側でネットワークドライブ作成
```
\\ホスト側のIPアドレス\ユーザ名
```

### iSCSI
* ターゲット/イニシエータインストール
```
$ yay -S targetcli-fb python-rtslib-fb python-configshell-fb open-iscsi
```

* ターゲット用LV作成

|LV|VG|サイズ|
|-|-|-|
|lv_iscsi|vg_hdd|50G|

```
$ sudo lvcreate -L サイズ VG名 -n LV名
```

* ターゲット設定
```
$ sudo systemctl start target
$ sudo systemctl enable target
$ sudo targetcli
> cd backstores/block
> create md_block0 /dev/vg_hdd/lv_iscsi
> cd /iscsi
> create
> cd
<iqn>/tpg1を選ぶ
> cd luns
> create /backstores/block/md_block0
> cd ../acls
> create <ホスト側イニシエータiqn(/etc/iscsi/initiatorname.iscsi)>
> create <VM側イニシエータiqn(iSCSIイニシエータプロパティの構成タブ)>
> cd ..
> set attribute authentication=0
> cd /
> saveconfig
```

* VM側イニシエータ設定
    * iSCSIイニシエータサービスを自動開始に設定
    * iSCSIイニシエータ > プロパティ > 探索タブ > ポータルの探索 > IPアドレス：ホスト側IP > ターゲットタブ > 接続
    * ディスクの管理より、NTFSフォーマットする

* ホスト側イニシエータ設定
```
$ sudo vim /etc/iscsi/iscsid.conf
該当箇所を変更
node.startup = automatic
$ sudo systemctl enable iscsid
$ sudo systemctl start iscsid
$ sudo iscsiadm -m discovery -p 127.0.0.1 -t st
$ sudo iscsiadm -m node -L all
$ sudo iscsiadm -m session -P 3
「Attached scsi disk sdx State: running」からどのデバイス(/dev/sdx)にアタッチされたか確認
$ sudo blkid
対象のUUIDを確認(TYPE="ntfs" PARTLABEL="Basic data partition")
$ sudo vim /etc/fstab
他にならって設定
UUID=上記UUID   マウントポイント    ntfs            ro,noatime,dmask=022,fmask=133,uid=ユーザID,gid=グループID,windows_names,x-systemd.device-timeout=5,_netdev,noauto,x-systemd.automount   0 2

$ sudo iscsiadm -m node -U all
```

* 現時点でiSCSIの自動ログインができていないため、/etc/fstabによる自動マウントもできない。そのため/etc/fstabの該当箇所はコメントアウトしておく