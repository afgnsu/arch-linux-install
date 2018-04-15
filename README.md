# Arch Linux 安裝教學

## 說明

[參考自 Arch Wiki](https://wiki.archlinux.org/index.php/installation_guide)
文章貢獻感謝原作者 Cheng-Yi Hong 以及 Arch Linux Taiwan 社群人員對本文之貢獻以及轉發

## 預安裝

一但你準備好了開機隨身碟，也在 bios 中使用 UEFI 模式，並且選擇以 usb 開機後，一般來說你可以毫無意外得進入 ArchISO 的 shell 畫面中，那麼如同大部分發行板的 live 安裝模式，我們擁有一個完整的 bash shell ，以及 kernel 環境。那麼它看起來就像各位熟悉的 Linux 純文字界面，那麼我們可以直接在裡頭進行安裝工作。

### 驗證起動模式

如果你已經啟用 UEFI 模式，那麼 Arch ISO 就會透過 UEFI 啟動你的系統，當然我們如果要驗證這一點，便可以列出 efivars 目錄

```shell
ls /sys/firmware/efi/efivars
```

### 設定網路連線

```shell
ping www.google.com
```

有線連線方法: ifconfig + dhclient

```shell
ifconfig <interface> up;
dhclient <interface>;
```

無線連線方法: ifconfig + wpa_supplicant

```shell
ifconfig <interface> up
wpa_passphrase <ESSID> <password> >> /etc/wpa_supplicant/wpa_supplicant.conf
wpa_supplicant -B -i <interface> -c /etc/wpa_supplicant/wpa_supplicant.conf
dhclient <interface>
```

### 分割磁區

在我們開始分割你的除存區以前我們要先卻認他的分區代號以及它是否被正確讀到，那麼我們可以運行行以下幾個指令

```shell
lsblk -a
```

上面這個指令可以幫你列出硬碟名，大小以及型態，那麼如果你需要稱加詳細的資料你，可以運行

```shell
fdisk -l
```

那麼值得一提的是所有在以上兩個指令下顯示掛載的除存裝置都會被系統認定為/dev 底下的其他設備，因此所有除存裝置的位置開頭都為/dev，舉例來說透過運行 lsblk 後，我得知我得固態硬碟名稱為 nvme0n1 那麼他在系統中掛載的位置便是/dev/nvme0n1
那麼在了解以上規則後我們就可以來分割磁區，這裡以最常見的機械硬碟磁區名稱/dev/sdax 來做講解

```shell
cfdisk /dev/sda
```

* /dev/sda1: /boot
  **空間至少 512MB，類型為 EFI System**
  <br />

* /dev/sda2: Swap
  **自訂，作者使用 8G，類型為 Linux Swap**
  <br />

* /dev/sda3: /
  **自訂，作者使用全部剩餘空間，類型為 Linux filesystem**
  <br />

基本上來說，我們都會在系統上加上 Swap（至換）分區。當然這個不是必須的，如果你覺得你的 RAM 大小足夠。那麼你可能覺得不需要這個分區也是可以的。順帶依提，當系統建立完成後想要新增 Swap 分區也是可行的。

### 格式化磁區

```shell
mkfs -t vfat /dev/sda1;
mkswap /dev/sda2;
mkfs -t ext4 /dev/sda3;
```

### 掛載磁區

```shell
mount /dev/sda3 /mnt;
mkdir /mnt/boot;
mount /dev/sda1 /mnt/boot;
```

## 安裝

一般來說我們都是使用 mirrorlist 來取得我們的 kernel 包，那麼你也可以選擇使用 Install Scripts 來安裝若是要使用 scripts 來安裝的話請參考此網址：https://github.com/danny8376/arch_install_script
若是想要使用 mirrirlist 的話便可以繼續閱讀本文

### 設定 pacman 的 mirrorlist

重新排序 pacman 的鏡像站順序，可以提高下載安裝的速度。

```shell
pacman -Sy reflector
reflector --verbose --latest 100 --sort rate --country 'Taiwan' --save /etc/pacman.d/mirrorlist
```

### 安裝 base 和 base-devel packages

```shell
pacstrap /mnt base base-devel
```

### 建立 fstab

接下來我們要生成一個 fstab 文件，其中-U 代表透過 UUID 來分類定義那麼這個檔案提供了檔案系統的資訊，他定義了儲存設備和磁區如何初始化和如何聯接至整個系統

```shell
genfstab -U /mnt >> /mnt/etc/fstab
```

### chroot 至新系統

chroot 是更改系統根目錄的位置

```shell
arch-chroot /mnt
```

### 設定時區

```shell
ln -sf /usr/share/zoneinfo/Asia/Taipei /etc/localtime
```

### Step 10 設定語言環境

```shell
echo "en_US.UTF-8 UTF-8" > /etc/locale.gen
echo "zh_TW.UTF-8 UTF-8" >> /etc/locale.gen
locale-gen
echo "LANG=en_US.UTF-8" > /etc/locale.conf
```

### 設定電腦名稱

```shell
echo "<your-pc-name>" > /etc/hostname
```

```shell
pacman -Sy vim
vim /etc/hosts
```

在 /etc/hosts 中加入最後一行

```shell
127.0.0.1  localhost.localdomain       localhost
::1        localhost.localdomain       localhost
127.0.0.1  <your-pc-name>.localdomain  <your-pc-name>
```

### 建立開機映像檔

[mkinitcpio 介紹](<https://wiki.archlinux.org/index.php/Mkinitcpio_(%E7%AE%80%E4%BD%93%E4%B8%AD%E6%96%87)>)

```shell
mkinitcpio -p linux
```

### 設定 root 密碼

```shell
passwd
```

### 安裝 grub 啟動載入程式

```shell
pacman -Sy grub os-prober efibootmgr
```

os-prober 可以用以偵測其他系統的存在，並在之後加入 grub 選單中。

```shell
os-prober
```

```shell
grub-install --target=x86_64-efi --efi-directory=/boot --bootloader-id=grub
grub-mkconfig -o /boot/grub/grub.cfg
```

### 安裝必要網路工具

```shell
pacman -S net-tools;
pacman -S wireless_tools;
pacman -S dhclient;
pacman -S wpa_supplicant;
```

### 建立新使用者

安裝 sudo

```shell
pacman -S sudo
```

設定 sudo 群組

```shell
vim /etc/sudoers
```

找到該行(大約在第 82 行)，並刪除前方的 # 號

```shell
# %wheel ALL=(ALL) ALL
```

建立新使用者，並加入 sudo 群組

```shell
useradd -m -u   <your-user-name>
passwd <your-user-name>
usermod <your-user-name> -G wheel
```

### 啟動必要開機模塊

```shell
systemctl enable dhcpd.service
```

### 重新啟動進入新系統

```shell
exit
umount -R /mnt
reboot
```

進入新系統後的網路設定請參考 Step 1

**(建議) 手動設定 DNS**

因筆者曾被預設的 DNS 雷過，建議手動設定

```shell
vim /etc/resolv.conf
```

將所有設定前方加上 # 作註解添加以下 DNS (最少 1 種，看個人選擇)

* nameserver 168.95.192.1 #中華電信
* nameserver 168.95.1.1 #中華電信
* nameserver 8.8.8.8 #Google
* nameserver 8.8.4.4 #Google

## 初次進入系統

### 安裝 CPU 微代碼(Microcode)

詳細請參閱：[Microcode](<https://wiki.archlinux.org/index.php/Microcode_(%E7%AE%80%E4%BD%93%E4%B8%AD%E6%96%87)>)

#### AMD

對於 AMD 處理器，其 Microcode 更新以包在 linux-firmware 中合併進系統中，因此不需要額外動作

#### Intel

對於 Intel 處理器我們需要另外安裝套件，並且在 bootloader 啟用 Microcode 更新

```shell
sudo pacman -S intel-ucode
```

/usr/bin/grub-mkconfig 可以自动处理微指令更新.用戶在安装完 intel-ucode 后，运行下面命令就可以重新生成配置文件启用微指令更新。

```shell
sudo grub-mkconfig -o /boot/grub/grub.cfg
```

### 安裝顯示晶片驅動

如果你有顯示卡的話，那我們就要需要安裝顯示晶片驅動，當然這並非必要步驟。

##### 注意: 通常顯示晶片驅動會在安裝或是啟動 xwindows 之前先行安裝完畢以免發生錯誤，若是在已經加載 xorg 的狀況下想要安裝驅動，建議先關閉 xwindows system。

#### Nvidia

目前較新型的晶片都可以被 arch 官方提供的開源驅動兼容因此我們只需要安裝

```shell
sudo pacman -S nvidia-lts
```

然後我們可以透過其提供的 nvidia-settings 圖形界面程式來調整設定。

#### AMD

因為筆者目前沒有 AMD 的顯示卡因此請直接參參閱 https://wiki.archlinux.org/index.php/AMD_Catalyst_(%E7%AE%80%E4%BD%93%E4%B8%AD%E6%96%87)

### 安裝桌面環境

#### Gnome

安裝 gnome 和 gnome-extra packages

```shell
sudo pacman -Sy gnome gnome-extra
```

使用 systemd 開機啟動 gnome 及 networkmanager (gnome 使用的網路管理工具)模塊

```shell
sudo systemctl enable NetworkManager
sudo systemctl enable gdm
```

### 重新開機

```shell
reboot
```

### 安裝 aur helper

Arch 使用者軟體倉庫 (AUR) 是由社群推動的使用者軟體庫。它包含了軟體包描述單 (PKGBUILD)，可以用 makepkg 從原始碼編譯軟體包，並透過 Pacman 安裝。 透過 AUR 可以在社群間分享、組織新進軟體包，熱門的軟體包有機會被收錄進 community 軟體庫。這份文件將解釋如何存取、使用 AUR。(本段來自 Arch Wiki)
那麼，如果我們想要使用 aur 上的資源，我們需要確認我們已經被妥一個擁有 [makepkg](https://wiki.archlinux.org/index.php/Makepkg)指令的環境。然後我們還需要使用 aur helper 來幫我們編譯 aur 上的內容。以下推薦幾個 aur helper
其他請參閱[arch aur](<https://wiki.archlinux.org/index.php/Arch_User_Repository_(%E6%AD%A3%E9%AB%94%E4%B8%AD%E6%96%87)>) 以及[aur helper](https://wiki.archlinux.org/index.php/AUR_helpers)頁面。

#### aurman

基於 pyrhon 的 aur helper

#### yaourt

```shell
sudo vim /etc/pacman.conf
```

找到以下兩行(約在第 93 行)，將前方的 # 刪除

```shell
#[multilib]
#Include = /etc/pacman.d/mirrorlist
```

下載 yaourt 及所需的依賴套件

```shell
pacman -Sy yajl git
git clone https://aur.archlinux.org/package-query.git
git clone https://aur.archlinux.org/yaourt.git
```

安裝 yaourt

```shell
cd package-query
makepkg -si
cd ../yaourt
makepkg -si
```

### 安裝中文輸入法 (fcitx)

安裝 fcitx

```shell
sudo pacman -S  fcitx-im;
sudo pacman -S  fcitx-chewing;
sudo pacman -S  fcitx-configtool;
```

```shell
sudo vim /etc/environment
```

在最後方添加以下三行

```shell
export GTK_IM_MODULE=fcitx
export QT_IM_MODULE=fcitx
export XMODIFIERS="@im=fcitx"
```

開啟 Fcitx Configuration 圖形界面新增 input method
找到 Chewing 並新增

## 安裝字型

```shell
sudo pacman -S noto-fonts;
sudo pacman -S noto-fonts-cjk;
sudo pacman -S ttf-roboto;
sudo pacman -S ttf-roboto-mono;
```

ttf-noto 支援所有 Unicode 的語言與字元
noto-fonts-cjk 為 Google 提供的免費字型(Chinese Japanese Korean)
ttf-robot 也是 Google 提供的很潮的字型，適合用來設計 UI

安裝過程電腦可能會好像當機、沒有反應，純屬正常現象，字型安裝完成就會恢復。

### NTFS 檔案系統讀寫支援

Linux kernel 不支援對 NTFS 檔案系統的讀取，如果額外的資料硬碟、其他硬碟是 NTFS 檔案系統的話，想要寫入就必須安裝額外的 [NTFS-3G](https://wiki.archlinux.org/index.php/NTFS-3G) Package

```shell
yaourt -S ntfs-3g
```

### 桌面美化工程

如果你已經可以完整的使用你的系統後，你可能覺的自己的桌面不太好看，那麼我們來將我們的桌面美化一下吧！！

首先：

Arch 自己的字體渲染實在不能看，在這方面 Ubuntu 做的比較好，那我們直接拿來用

```shell
yaourt -S freetype2-ubuntu;
yaourt -S fontconfig-ubuntu;
yaourt -S cairo-ubuntu;
```

接下來我們可以安裝 theme 系統，主流的有：

##### Arc

```shell
sudo pacman -S arc-gtk-theme;
```

##### Numix

```shell
yaourt -S numix-theme;
```

#### 接下來是 icon 系統

##### Arc-icon

```shell
sudo pacman -S arc-icon-theme ;
```

##### Numix-icon

```shell
yaourt -S numix-circle-icon-theme-git;
```

##### Vivacious Colors icon

```shell
yaourt -S vivacious-colors-icon-theme;
```

其他的可以去 gtk 的網站找來玩玩看：
https://www.gnome-look.org/browse/cat/135/

## Enjoy your new system

享受你的新系統，盡情客製化它吧！
