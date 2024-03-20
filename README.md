# Arch Linux文档

我的ArchLinux使用文档

[TOC]


## 安装系统

连接网络

```bash
iwctl # 进入交互式命令行
device list # 列出无线网卡设备名，比如无线网卡看到叫 wlan0
station wlan0 scan # 扫描网络
station wlan0 get-networks # 列出所有 wifi 网络
station wlan0 connect wifi-name 
# 进行连接，注意这里无法输入中文。回车后输入密码即可

exit # 连接成功后退出

ping www.bilibili.com # 测试
```

设置时间

```bash
timedatectl set-ntp true # 将系统时间与网络时间进行同步
timedatectl status # 检查服务状态
```

分区 1

```bash
lsblk # 显示当前分区情况
cfdisk /dev/nvme0n1 # 自己分区

# 格式化
mkfs.fat -F32 /dev/nvme0n1p1
mkfs.btrfs -L Rikka /dev/nvme1n1p2

# 挂载分区
mount -t btrfs -o compress=zstd /dev/nvme0n1p2 /mnt

# 创建根目录家目录子卷
btrfs subvolume create /mnt/@
btrfs subvolume create /mnt/@home

# 检查子卷情况
btrfs subvolume list -p /mnt 

# 解除挂载
umount /mnt

# 挂载
mount -t btrfs -o subvol=@,noatime,discard=async,compress=zstd /dev/nvmexn1p2 /mnt
mkdir /mnt/home
mount -t btrfs -o subvol=@home,noatime,discard=async,compress=zstd /dev/nvmexn1p2 /mnt/home
mkdir -p /mnt/boot/efi
mount /dev/nvme0n1p1 /mnt/boot/efi

pacman -Sy archlinux-ketring
# pacman-key --init
# pacman-key --populate archlinux
# pacman-key --refresh-keys

# 安装一些基本的东东
pacstrap -K /mnt base base-devel linux linux-firmware btrfs-progs dhcpcd iwd vim sudo bash-completion networkmanager

# 做grub
genfstab -U /mnt > /mnt/etc/fstab

# 看看发育正不正常
cat /mnt/etc/fstab

# 切到chroot开始访问
arch-chroot /mnt
```

设置主机名与时区

```bash
vim /etc/hostname 
# 里面写入主机名

vim /etc/hosts
# 里面写入：
127.0.0.1   localhost
::1         localhost
127.0.1.1   主机名.localdomain	主机名

# 时区
ln -sf /usr/share/zoneinfo/Asia/Shanghai /etc/localtime
# 同步到硬件
hwclock --systohc
```

设置 Locale

```bash
# 通过以下命令安装对应芯片制造商的微码：
# 编辑下面文件
vim /etc/locale.gen

# 去掉 
# en_US.UTF-8 UTF-8
# zh_CN.UTF-8 UTF-8 
# 行前的注释符号[ # ]

# 然后使用如下命令生成 locale
locale-gen

# 向 /etc/locale.conf 输入内容：
echo 'LANG=en_US.UTF-8'  > /etc/locale.conf
# 这里先不要设置任何中文 locale，会导致 tty 乱码。
# 等安装好了回来再设置也不迟
```

为 root 用户设置密码

```bash
passwd root
```

安装对应的微码

```bash
# 通过以下命令安装对应芯片制造商的微码：
pacman -S amd-ucode # AMD
```

安装引导驱动程序

```bash
# 1. 安装相应的包：
pacman -S grub efibootmgr
# 命令参数说明：
# 	-S 选项后指定要通过 pacman 包管理器安装的包：
# 	grub —— 启动引导器
# 	efibootmgr —— efibootmgr 被 grub 脚本用来将启动项写入 NVRAM

# 2. 安装 GRUB 到 EFI 分区：
grub-install --target=x86_64-efi --efi-directory=/boot --bootloader-id=Arch

# 3. 接下来使用 vim 编辑 /etc/default/grub 文件：
vim /etc/default/grub
# 修改操作：
# 1. 去掉 GRUB_CMDLINE_LINUX_DEFAULT 一行中最后的 quiet 参数
# 2. 把 loglevel 的数值从 3 改成 5。这样是为了后续出现系统错误，方便排错
# 3. 加入 nowatchdog 参数，这可以显著提高开关机速度

# 最后生成 GRUB 所需的配置文件：
grub-mkconfig -o /boot/grub/grub.cfg
```

完成安装

```bash
exit # 退回安装环境
umount -R /mnt # 卸载新分区
reboot # 重启
```

重启电脑 拔出u盘

重启后使用 root 账户登录系统

设置开机自启并立即启动 dhcp 服务

```bash
systemctl enable --now dhcpcd # 设置开机自启并立即启动 dhcp 服务
ping www.bilibili.com # 测试网络连接
```

若为无线连接，则还需要启动 `iwd` 才可以使用 `iwctl` 连接网络：

```bash
systemctl start iwd # 立即启动 iwd
iwctl # 和之前的方式一样，连接无线网络
device list # 列出无线网卡设备名，比如无线网卡看到叫 wlan0
station wlan0 scan # 扫描网络
station wlan0 get-networks # 列出所有 wifi 网络
station wlan0 connect wifi-name 
exit # 连接成功后退出
ping www.bilibili.com # 测试
```

`neofetch` 可以将系统信息和发行版 logo 一并打印出来。

```bash
# 通过 pacman 安装 neofetch：
pacman -S neofetch

# 使用 neofetch 打印系统信息：
neofetch
```

关机 / 重启

```bash
reboot # 重启
shutdown -h now # 关机
```



## 用户配置

确保系统为最新

```bash
pacman -Syyu # 升级系统中全部包
```

配置 root 账户的默认编辑器

```bash
# 1. 使用 vim 编辑 ~/.bash_profile 文件：
# 如果不做额外配置且不显式的指定编辑器，
# 在一些终端场景下（如下面的 visudo、git 等）调用编辑器时会出错。
vim ~/.bash_profile
# 写入 export EDITOR='vim'


# 也可以添加到 ~/.bashrc 中，但是（如果不做其它配置或显式的执行）在登录命令行 tty 后不会被执行，也就失去了意义。
# 一般来说我们登录 root 账户时很可能是在命令行 tty 登录的（有时也会 su）。
```

准备非 root 用户

```bash
# 创建用户
useradd -m -G wheel -s /bin/bash myusername

# 为该用户设置密码
passwd myusername

# 使用 vim 编辑器通过 visudo 命令编辑 sudoers 文件：
EDITOR=vim visudo # 这里需要显式的指定编辑器，因为上面的环境变量还未生效
# 找到如下这样的一行，把前面的注释符号 # 去掉：
# %wheel ALL=(ALL) ALL
```

>📑 命令参数说明：**`useradd`**
>
>- `-m` 创建用户的同时创建用户家目录
>
>- -G 选项后指定附加组
> - `wheel` —— `wheel` 附加组可 `sudo` 进行提权
    >
    >  - `-s` 选项后指定 shell 程序
>
>  - `myusername` —— 用户名（**请自定义**，但不要包含空>格和特殊字符）
>
>
>📑 这里稍微解释一下：**`/etc/sudoers.tmp文件内容`**
>
> - `%wheel` —— 用户名或用户组，此处则代表是 `wheel` 组，`%` 是用户组的前缀
>- `ALL=` —— 主机名，此处则代表在所有主机上都生效（如果把同样的 `sudoers` 文件下发到了多个主机上）
> - `(ALL)` —— 用户名，此处则代表可以成为任意目标用户
> - 最后的 `ALL` —— 代表可以执行任意命令
>
> 几个更详细的例子:
>
> 1. 在 `mailadmin` 组里的用户可以作为 `root` 用户，在 `snow` 和 `rain` 这两台主机执行一些邮件服务器控制命令（命令之间用 `,` 分隔）：
>
> ```bash
>%mailadmin  snow,rain=(root)  /usr/sbin/postfix, /usr/sbin/postsuper, /usr/bin/doveadm
> ```
>
> 1. 用户 `whoami` 可以在所有主机上以 `root` 用户不输入密码执行 `rndc reload` 这条命令（正常来说 `sudo` 都是要求输入调用方的密码的）：
>
> ```bash
>whoami  ALL=(root)  NOPASSWD: /usr/sbin/rndc reload
> ```
>
> 1. 当在 `users` 组里的用户以 `sudo passwd` 或者 `sudo passwd root` 方式运行命令的时候，可以直接把 `root` 用户的密> 码 改掉，这真是太危险了！必须要把这两条命令禁止掉，但我们又希望用户可以通过 `sudo passwd` 修改其它用户的密码。那么我们可以在命令前面加上 `!` 来表示不可执行的命令：
>
> ```bash
>%users  ALL=(root)  !/usr/bin/passwd, /usr/bin/passwd [A-Za-z]*, !/usr/bin/passwd root
> ```
>
> 总结一下，语法如下：
>
> ```bash
>用户名/%用户组名 主机名=(目标用户名) 命令1, 命令2, !命令3
> ```

---

开启 32 位支持库与 Arch Linux 中文社区仓库（archlinuxcn）

```bash
# 编辑 /etc/pacman.conf 文件：
vim /etc/pacman.conf

# 1. 去掉 [multilib] 一节中两行的注释，来开启 32 位库支持
# [multilib]
# Include = /etc/pacman.d/mirrorlist

# 2. 在文档结尾处加入下面的文字，来添加 archlinuxcn 源

# 中国科学技术大学开源镜像站
[archlinuxcn]
Server = https://mirrors.ustc.edu.cn/archlinuxcn/$arch 

# 清华大学开源软件镜像站
[archlinuxcn]
Server = https://mirrors.tuna.tsinghua.edu.cn/archlinuxcn/$arch 

# 哈尔滨工业大学开源镜像站
[archlinuxcn]
Server = https://mirrors.hit.edu.cn/archlinuxcn/$arch 

# 华为开源镜像站
[archlinuxcn]
Server = https://repo.huaweicloud.com/archlinuxcn/$arch 

# 或其他的源
# 保存并退出 
# 更新
pacman -Syyu
```

重启

```bash
reboot
```

使用非root 刚刚创建的用户登陆

```bash
用户名
密码
```

配置非 root 账户的默认编辑器

```bash
# 1. 使用vim编辑 ~/.bashrc 文件
vim ~/.bashrc
# 在适当位置加入以下内容：
# export EDITOR='vim'
# 也可以添加到 ~/.bash_profile 文件中。
```









## 安装桌面

Hyprland桌面

安装Hyprland

```bash
yay -S hyprland-git  # compiles from latest source
yay -S hyprland		# compiles from latest release source
yay -S hyprland-bin	 # compiled latest release, prone to breaking on ARM devices as Hyprland binary is compiled for x86
```

包装启动器

```bash
vim /usr/share/wayland-sessions/hyprland.desktop

[Desktop Entry]
Name=Hyprland
Comment=An intelligent dynamic tiling Wayland compositor
Exec=hyprland-session
Type=Application
```

vim /usr/bin/hyprland-session

```bash
#!/bin/bash

cd ~

export LIBVA_DRIVER_NAME=nvidia
export XDG_SESSION_TYPE=wayland
export GBM_BACKEND=nvidia-drm
export __GLX_VENDOR_LIBRARY_NAME=nvidia
export WLR_NO_HARDWARE_CURSORS=1
export _JAVA_AWT_WM_NONREPARENTING=1
export XCURSOR_SIZE=24

exec Hyprland
```

[开始配置](https://wiki.hyprland.org/Configuring/Configuring-Hyprland/)

```
https://wiki.hyprland.org/Configuring/Configuring-Hyprland/
```

[我的配置](https://github.com/521Ayaka/mycfg)

```
https://github.com/521Ayaka/mycfg
```

关于INVIDIA

安装驱动程序并将其添加到您的初始化参数和内核参数

```bash
MODULES=(nvidia nvidia_modeset nvidia_uvm nvidia_drm)
https://wynio.space/posts/37c1/
```

[更多](https://wiki.hyprland.org/Nvidia/)



我的配置依赖包

```bash
yay -S kitty   # 终端
yay -S rofi    # 程序启动器
yay -S nemo    # 资源管理器
yay -S ranger  # 资源查看
yay -S waybar  # 任务栏
yay -S swww	   # 壁纸
yay -S htop btop
yay -S zsh     # zsh
yay -S clash   # 科学上网
yay -S nitch   # 查看配置
yay -S cmatrix # 数字雨     
yay -S wlogout # 关机 注销 等

```







## 安装基础功能包



```bash
# 确保 iwd 开机处于关闭状态，因为其无线连接会与 NetworkManager 冲突
sudo systemctl disable iwd 
# 立即关闭 iwd
sudo systemctl stop iwd 
# 确保先启动 NetworkManager，并进行网络连接。若 iwd 已经与 NetworkManager 冲突，则执行完上一步重启一下电脑即可
sudo systemctl enable --now NetworkManager 
# 测试网络连通性
ping www.bilibili.com 
```

安装一些基础功能包：

```bash
sudo pacman -S ntfs-3g # 使系统可以识别 NTFS 格式的硬盘
sudo pacman -S adobe-source-han-serif-cn-fonts wqy-zenhei # 安装几个开源中文字体。一般装上文泉驿就能解决大多 wine 应用中文方块的问题
sudo pacman -S noto-fonts noto-fonts-cjk noto-fonts-emoji noto-fonts-extra # 安装谷歌开源字体及表情
sudo pacman -S firefox chromium # 安装常用的火狐、chromium 浏览器
sudo pacman -S ark # 压缩软件。在 dolphin 中可用右键解压压缩包
sudo pacman -S gwenview # 图片查看器
sudo pacman -S steam # 游戏商店。稍后看完显卡驱动章节再使用
```

安装输入法

```bash
sudo pacman -S fcitx5-im # 输入法基础包组
sudo pacman -S fcitx5-chinese-addons # 官方中文输入引擎
sudo pacman -S fcitx5-anthy # 日文输入引擎
sudo pacman -S fcitx5-pinyin-moegirl # 萌娘百科词库。二刺猿必备（archlinuxcn）
sudo pacman -S fcitx5-material-color # 输入法主题
```

显卡安装

```bash
# Intel 核显
sudo pacman -S mesa lib32-mesa vulkan-intel lib32-vulkan-intel

# AMD 核芯显卡 开源驱动 AMDGPU
sudo pacman -S mesa lib32-mesa xf86-video-amdgpu vulkan-radeon lib32-vulkan-radeon
# AMD 核芯显卡 开源 ATI 驱动
sudo pacman -S mesa lib32-mesa xf86-video-ati

# NVIDIA独显 LINUX
sudo pacman -S nvidia nvidia-settings lib32-nvidia-utils # 必须安装
# NVIDIA独显 others
sudo pacman -S nvidia-dkms nvidia-settings lib32-nvidia-utils # 必须安装


# 工具
yay -S optimus-manager optimus-manager-qt
sudo systemctl enable optimus-manager.service
```

安装 `archlinuxcn` 源所需的相关步骤

```bash
# cn 源中的签名（archlinuxcn-keyring 在 archlinuxcn）
sudo pacman -S archlinuxcn-keyring 
# yay 命令可以让用户安装 AUR 中的软件（yay 在 archlinuxcn）
sudo pacman -S yay 
```

> ℹ️ 提示
>
> 若安装 `archlinuxcn-keyring` 时报错，是由于密钥环的问题。可先按照 [archlinuxcn 官方说明open in new window](https://www.archlinuxcn.org/gnupg-2-1-and-the-pacman-keyring/) 执行其中的命令，再安装 `archlinuxcn-keyring`。






































使用文本编辑器打开/etc/pacman.conf，找到

```bash
#[multilib]
#Include = /etc/pacman.d/mirrorlist
12
```

将之修改为

```bash
[multilib]
Include = /etc/pacman.d/mirrorlist
```

编辑下列文件：/etc/systemd/logind.conf

\#HandlePowerKey按下电源键后的行为，默认power off

\#HandleSleepKey 按下挂起键后的行为，默认suspend

\#HandleHibernateKey按下休眠键后的行为，默认hibernate

\#HandleLidSwitch合上笔记本盖后的行为，默认suspend（改为lock；即合盖不休眠）在原文件中，还要去掉前面的#

运行：systemctl restart systemd-logind 就会生效。



![image-20221023231624248](MD图片/Arch  dwm文档.assets/image-20221023231624248.png)





## 常用包

主题 lxappearance  kvantummanager







## ---------



## 常见问题解决

手动安装字体

```bash
cp . ~/.local/share/font/
fc-cache -f -v
```







## TODO:
