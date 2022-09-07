<!-- 在VirtualBox虚拟机下ArchLinux安装 -->
# ArchLinux 安装  

## 准备

1. 下载[ArchLinux镜像](https://archlinux.org/download/)
    > 实体机这里将镜像放到ventoy格式化好的u盘下
2. VirtualBox --> 新建虚拟电脑
    1. 名称 --> Arch01
    2. 内存 --> 8192MB
    3. 虚拟硬盘 --> 类型VDI(只适用于VitrualBox,类型VMDK可以在其他虚拟机(例如VM Ware)使用) --> 固定大小100.00GB

    4. 选择创建好的虚拟机项目 --> 设置 --> 常规 --> 高级 --> 开启共享粘贴板
    5. 系统 --> 启动顺序 光驱>硬盘 --> 启动EFI --> 处理器 --> 8
    6. 显存大小 --> 最大
    7. 存储 --> 添加虚拟光驱 --> 注册 --> 选中镜像archlinux-x86_64.ios文件
    8. 网络 --> 网卡1 NAT(默认) --> 网卡2 --> 启动网络连接 --> 连接方式 --> 仅主机(Host-only)网络

3. 启动(若过程中有弹出窗口 --> 取消) --> 进入到引导界面 --> Arch Linux install medium --> 进入命令行(root@archiso ` #)(此时无法ssh 需要在命令行操作 这时候命令没有提示无法复制粘贴)
4. ip a(确定两个网卡 网卡1:enp0s3和网卡2:enp0s8)(如果只有一个 --> 宿主机windows11下 --> 网络和lnternet -->高级网络设置 -->查看是否有VirtualBox Host-Only Network这个网络适配器)

## 配置软件源

1. `setfont ter-132n`(调整下命令行字体大小 若显示太多内容 --> Ctrl+L清屏)
2. `ping -c 10 baidu.com`(查看下是否有网络)(实体机使用iwctl连接wifi 无线网卡服务需要打开rfkill unblock wifi)
    > iwctl --> device list (查看无线网卡列表)--> station + 无线网卡name + scan(搜索wifi) --> station + 无线网卡name + get-networks --> station +无线网卡name + connect +wifi名 -->输入wifi密码

    > dhcpcd 分配下ip地址
3. 修改镜像源(通过修改/etc/pacman.d/mirrorlist或者自动搜索并添加)  
    这里自动搜索

    ```shell
    reflector -c China -a 5 --sort rate --save /etc/pacman.d/mirrorlist
    ```

    > -c 国家 -a 需要源的数量 --sort rate 速度排序 --save 写入覆盖 执行后确定下mirrorlist是否被修改了 这里可能会超时

4. 同步下源  

    ```shell
    pacman -Syy
    ```

## 磁盘分区(使用gdisk、fdisk、cfdisk(这个有图型界面))

1. 查看磁盘分区 `lsblk`
2. 使用gidsk分区 -->设置efi引导分区+512M ext4格式文件分区+98G swap交换分区

    ```shell
    gdisk /dev/sda
    ```

    > 设置分区操作大同小异输入n 开辟空间 基本流程是分配大小 起始->最后 这里只设置最后+512M +98G 剩余空间分配给交换分区 -> 设置格式gdisk中 efi格式:ef00 ext4:8300 swap:8200 ->输入p查看分区情况 -> w保存 sda是磁盘空间的名字 可以lsblk查看硬盘名

3. `lsblk -f` 确定下分区情况
4. 格式化分区
    > 例如lsblk下三个分区为sda1引导分区 sda2主分区 sda3交换分区

    ```shell
    mkfs.vfat /dev/sda1
    mkfs.xfs /dev/sda2
    mkswap /dev/sda3
    ```

    > 将引导分区格式化为fat32 主分区格式化为ext4或者xfs 将交换分区格式化为swap
5. 挂载分区(可以理解为标签)

    ```shell
    mount /dev/sda2 /mnt
    mkdir -p /mnt/boot/efi
    mount /dev/sda1 /mnt/boot/efi
    swapon /dev/sda3
    lsblk
    ```

    > 这里实体机安装时 /mnt/boot/efi 挂载到 windows 的efi同一磁盘下

    > swapon /dev/sda3这里为打开交换分区

## 安装

1. 安装linux系统和基础包

    ```shell

   pacstrap /mnt linux linux-firmware linux-headers base base-devel vim bash-completion
    ```

    > pacstrap是一个安装脚本 /mnt表示安装在sda2中 linux为最新版或安装linux-lts为稳定版 linux-firware和linux-headers为固件和头文件 base和base-devel为基础包和开发基础包 vim是一个文本编辑器 bash-completion为命令自动补全
  
    > pacman -S archlinux-keyring 如果安装出现错误可以检查下是否安装PGP密钥

2. 生成fstab文件

    ```shell

   genfstab -U /mnt >> /mnt/etc/fstab
   cat /mnt/etc/fstab
    ```

    > 生成文件系统的表文件 cat查看下是否修改

## 配置grup

1. 切换系统

    ```linux
   arch-chroot /mnt
   pacman -Syy
   pacman -S grub efibootmgr efivar networkmanager amd-ucode
   grub-install /dev/sda
   vim /etc/default/grub
   grub-mkconfig -o /boot/grub/grub.cfg
   systemctl enable NetworkManager
   passwd
   exit
   umount /mnt/boot/efi
   umount /mnt
   reboot
    ```

    >实体机安装时这里指定下架构 例如 grub-install --target=x86_64-efi --efi-directory=/efi --bootloader-id=arch 将启动项取名为arch 启动类型为efi的64位系统 系统启动位置在 /efi

    > 进入 /mnt 即进入主分区已安装好的系统中 此时root@... `变为root@.../即成功 grub efibootmgr efivar为grub启动引导所需要的包networkmanager为网络管理 amd-ucode这里是cpu微代码 根据cpu为inter或amd安装不同的包 虚拟机使用的cpu和宿主机一样 grub-install /dev/sda 这里是将archlinux设为grep启动项(例如实体机为双系统 这里可以选择和切换) 如果这里如果出现报错大概率是在设置虚拟机时没有开启EFI grub-mkconfig -o /boot/grub/grub.cfg这里是生成grub的配置文件写入到指定目录 systemctl enable NetworkManager这里为开机启动网络服务 passwd这里为设置root用户密码 需要输入两次 exit退出系统 umount /mnt/boot/efih和 umount /mnt这里为卸载分区 reboot重启 pacman -S 这里的参数-S 为安装
    这里vim修改下grup启动引导的时间 默认5秒 GRUB_TIMEOUT=5 将loglevel=3 quiet 去掉quiet 可以查看grup的报错日志 i进入编辑模式 wq保存退出

## 配置系统

1. 确定网络

    ```shell
   ping -c 10 baidu.com
   setfont /usr/share/kbd/consolefonts/sum12x22.psfu.gz
    ```

    > setfont /usr/share/kbd/consolefonts/sum12x22.psfu.gz修改下字号 如果是实体机此时没有网络可以输入nmtui使用图形界面连接wifi这里可以使用nmtui是因为安装了networkmanager 如果这时候没有网可能是没有开启networkmanager服务
2. 修改主机名并配置host

   ```shell
   vim /etc/hostname
   vim /etc/hosts
    ```

    hosts修改
    > 例如hostname设置为Arch hosts添加以下3行

     ```vim
   127.0.0.1    localhost
   ::1          localhost
   127.0.1.1    Arch.localdomain Arch
    ```

3. 设置时间

    ```shell
   timedatectl set-timezone Asia/Shanghai
   timedatectl set-ntp true
   timdatectl status
   reboot
    ```

   > timedatectl set-timezone Asia/Shanghai这里是设置时区亚洲->上海 timedatectl set-ntp true这里是同步网络时间  timdatectl status这里是查看此时时间

4. 添加标准用户

    ```shell
   useradd --create-home leegl
   id leegl
   passwd leegl
   usermod -aG wheel,users,storage,power,lp,adm,optical leegl
   id leegl
   visudo
   reboot
    ```

    > 例如添加一个名为leegl的用户 1.0 useradd --create-home leegl这里 --create-home 会生成一个同名的目录存放用户配置 root用户和标准用户类似vscode全局配置和工作区配置的区别 2.0 usermod -aG wheel,users,storage,power,lp,adm,optical leegl 这里的参数-aG a代表添加append G代表组 wheel,users,storage,power,lp,adm,optical为组名称 最后 + 用户名 然后id leegl查看是否加入组中 visudo 编辑sudo的配置文件 找到这里# %wheel ALL=(ALL) ALL去掉注释 这里的作用是使标准用户正常使用sudo 在此之前如果没有将vim设为默认编辑器 可以export EDITOR=vim

5. 配置ssh(实体机可以忽略)
    > 重启后会提示登陆用户 这里使用标注用户例如leegl登录

    ```shell
    sudo pacman -S openssh
    systemctl status sshd.service
    systemctl start sshd
    systemctl enable sshd.service
    systemctl status sshd.service
    cat /etc/services | grep 22
    ip a
    ```

    > systemctl status sshd.service 这句为查看ssh状态如果未开启 systemctl start sshd 这里开启服务 systemctl enable sshd.service 这里为开机启动ssh服务然后再次查看状态  cat /etc/services | grep 22 这里查看linux 22端口是否开启 ssh默认端口为22 ip a 记录下enp0s8(设置的第二块网卡名称不固定)的ip地址
    > 如果这里ip a地址为类似乱码 --> VirtualBox管理器 -->管理 --> 主机网络管理器 -->启用DHCP服务器 --> 重启linux 再次ip a  

    在宿主机上开启终端(或者任意ssh工具)
    > 这里使用的是VS Code中默认的powershell 这里举例enp0s8获取的ip地址为192.168.7.777 用户名为leegl ///这里是宿主机

    ```shell
    ping 192.168.7.777
    ssh leegl@192.168.7.777 -p 22
    ```
    > ssh leegl@192.168.7.777 -p 22这里会生成一个凭证 输入yes 后会提示输入密码 这里的leegl为之前的标准用户-->

6. 在宿主机后台运行(这里可以省略)
    > VirtualBox 可以把虚拟机设置为后台启动 在管理器这里 启动->无界面启动
    虚拟机下

    ```shell
    sudo poweroff
    ```

    > 这里接上面步骤  sudo poweroff为关机 也可以使用VirtualBox的强制关闭 然后无界面启动

    启动后在宿主机使用ssh连接

    ```shell
    ssh leegl@192.168.7.777 -p 22
    ```

7. 设置bash配置文件(这里可以忽略)

    ```shell
    cd /etc/skel
    ls -al
    vim .bashrc
    cp -a . ~
    ```

    在.bashrc配置文件之添加5行

     ```vim
    export EDITOR=vim
    alias grep='grep --color=auto'
    alias egrep='egrep --color=auto'
    alias fgrep='fgrep --color=auto'
    [ ! -e ~/.dircolors ] && eval $(dircolors -p > ~/.dircolors)
    [ -e /bin/dircolors ] && eval $(dircolors -b ~/.dircolors)
    ```

    > .bashrc这个是命令行的配置文件可以设置全局变量 export EDITOR=vim这里设置的是将编辑器默认为vim 剩下的4行设置的是grep的关键字和根据文件类型设置的高亮显示 这里不建议修改 cp -a . ~ 这里是 将这个文件的修改覆盖到~目录下 -a参数保留原文件属性的前提下复制

8. 安装xorg

    ```shell
    sudo pacman -Syy
    sudo pacman -S xorg
    ```

    > xorg是一个组合包 包含字体驱动之类 (为了简约(可以安装xorg-server)后续我们需要安装dwm这里安装xorg) 它的作用是处理硬件和桌面环境交互 无论安装任何桌面环境还是窗口管理器都需要

9. 安装字体

    ```shell
    sudo vim /etc/locale.gen
    ```

    > locale.gen包含所有语言
    去注释2行

    ```vim
    en.US.UTF-8
    zh.CN.UTF-8
    ```

    > 去注释为中文(简体)和英文(美国)
    编辑下locale.conf

     ```shell
    sudo vim /etc/locale.conf
    ```

    修改或添加

    ```vim
    LANG=en_US.UTF-8
    ```

    > 如果存在则修改 设置系统语言为英文
    生成字体

    ```shell
    sudo locale-gen
    ```

    其他字体
    > 上英文相关 下中文相关

    ```shell
    sudo pacman -S ttf-dejavu ttf-droid ttf-hack ttf-font-awesome otf-font-awesome ttf-lato ttf-liberation ttf-linux-libertine ttf-opensans ttf-ubuntu-font-family
        
    sudo pacman -S ttf-hannom  noto-fonts noto-fonts-extra noto-fonts-emoji  noto-fonts-cjk  adobe-source-code-pro-fonts adobe-source-sans-fonts adobe-source-serif-fonts adobe-source-han-sans-cn-fonts adobe-source-han-sans-hk-fonts adobe-source-han-sans-tw-fonts adobe-source-han-serif-cn-fonts adobe-source-han-serif-hk-fonts adobe-source-han-serif-tw-fonts wqy-zenhei wqy-microhei        
    ```

    开启freetype2
    > freetype2是一个字体渲染工具

    ```shell
    sudo vim /etc/profile.d/freetype2.sh
    ```

    > 去注释export开头这行

10. 显卡驱动(虚拟机下可以忽略)  

    inter

    ```shell
    sudo Pacman -S xf86-video-inter vulkan-inter mesa
    ```

    nvidia

    ```shell
    sudo Pacman -S nvidia nvidia-settings nvidia-utils
    ```

11. 安装声音相关工具

    ```shell
    sudo pacman -S alsa-utils pulseaudio pulseaudio-bluetooth 
    ```

12. 安装git

    ```shell
    sudo pacman -S git
    ```

13. 安装AUR helper(paru或yay)
    >  AUR helper 是用户软件仓库的封包工具 [AUR](https://aur.archlinux.org/) 这里的包我们都可以手动或者通过AUR helper安装到我们的arch中

    > paru 基本用法 输入paru效果等同pacman -Syy 输入paru + 包名称 这里会搜索所有包含此包名称的包以列表的形式 输入列表前的数字安装对应包

    以下通过两种方式安装  
    1. 通过下载包
        > 所有AUR上的包都可以通过这种方式安装 AUR-helper只是通过命令简化了这些步骤
        1. 在[AUR](https://aur.archlinux.org/)中搜索paru -->复制paru的git clone URL
        2. `git clone https://aur.archlinux.org/paru.git`(克隆到本地后 在`目录下会生成安装paru的包)
        3. `cd paru/`
        4. `sudo vim /etc/makepkg.conf`
            > 修改makepkg的配置文件 这里修改的是编译的并发数量 这里可以忽略
            > 找到MAKEFLAGS="-j2"这行 --> 去注释 --> 修改为

            ~~~shell
            MAKEFLAGS="-j$(nproc)"
            ~~~

        5. 编译安装

           ~~~shell
           makepkg -si
           ~~~

    2. 添加archlinuxcn源 通过pacman安装

        ~~~shell
        sudo vim /etc/pacman.conf
        ~~~

        添加3行

        ~~~vim
        [archlinuxcn]
        SigLevel = Optional TrustAll
        Include = /etc/pacman.d/mirrirlistcn
        ~~~

        > [archlinuxcn]这里是添加archlinuxcn中文仓库 SigLevel = Optional TrustAll这里是包的签名密钥 可以选择修改为Never跳过包签名 Include = /etc/pacman.d/mirrirlistcn这里是源指向的地址(指向一个文件 和mirrirlist同理 ) [archlinuxcn]这里只是一个变量对应的名字 通过修改软件源的变量找到仓库的服务器 接下来创建这个指向的文件

        ~~~shell
        [archlinuxcn]
        cp /etc/pacman.d/mirrorlist /etc/pacman.d/mirrorlistcn
        ~~~

        > 这里通过 -a 复制下mirrorlist的文件属性并清除文件内容并添加源 两行

        ~~~vim
        Server = https://mirrors.ustc.edu.cn/$repo/$arch 
        Server = https://mirrors.tuna.tsinghua.edu.cn/$repo/$arch
        ~~~

        > 这里的$repo 就是archlinuxcn

        更新下源 清楚缓存

        ~~~shell
        sudo pacman -Syy
        sudo pacman -Scc
        ~~~

        >如果这里设置正确 应该会有archlinuxcn这个仓库被同步

        防止PGP签名出错

         ~~~shell
        sudo pacman -S archlinuxcn-keyring
         ~~~

        安装 paru

        ~~~shell
        sudo pacman -S paru
         ~~~

         修改下配置文件(可以忽略)

         ~~~shell
        sudo vim /etc/paru.conf
         ~~~

         去注释这一行

         ~~~vim
        #BottomUp
         ~~~

         > 这里的修改翻转paru检索包列表的顺序 相关度较大的包放在了下方

    3. paru安装chrome谷歌浏览器
        >paru 或者 yay 等命令不允许使用root权限操作 这里不需要sudo

        ~~~shell
        paru google-chrome
        ~~~

        这里会搜索各个仓库包含google-chrome的包选择一个安装
        > 这里中途可能会弹出一些说明 q 之后继续安装
        > 这里没有安装桌面环境 先确定下是否安装成功whereis google-chrome
    4. paru安装firefox火狐浏览器

        ~~~shell
        paru firefox
        ~~~

        > 如果提示签名错误 可以尝试

        ~~~shell
        sudo pacman-key --init
        sudo pacman-key --populate
        ~~~

## 安装窗口管理器dwm(基础)

1. 在用户目录下(~) 新建一个文件例如ProgramFiles 将源码克隆到这里

    ~~~shell
    cd ~
    mkdir ProgramFiles
    cd ProgfamFiles
    git clone https://git.suckless.org/dwm
    git clone https://git.suckless.org/st
    git clone https://git.suckless.org/dmenu
    ~~~

    > 这里如果git clone的很慢 可以修改下hosts 并重启

    ~~~vim
    199.232.69.194  github.global-ssl.fastly.net
    140.82.112.4    github.com
    ~~~

    >这里确定下安装xorg

    ~~~shell
    whereis xorg
    ~~~

2. 安装dwm(窗口管理器)、st(终端)、dmenu(动态菜单)

    ~~~shell
    cd dwm
    sudo make clean install
    cd ../st
    sudo make clean install
    cd ../dmenu
    sudo make clean install

    sudo pacman -S xorg-xinit feh udisks2 udiskie pcmanfm
    ~~~

    >dwm是用c写的

    >  xorg-xinit开启xorg协议 feh设置壁纸 udisks2 udiskie usb相关 pcmanfm文件管理器

    ~~~shell
    cp /etc/X11/xinit/xinitrc ~/.xinitrc
    cd ~
    sudo vim .xinitrc
    ~~~

    >注释掉最后5行 添加1行

    ~~~shell
    exec dwm
    ~~~

    > 开启udisks2服务

    ~~~shell
    sudo systemctl enable udisks2
    ~~~

    > 显示硬件信息

    ~~~shell
    sudo pacman -S neofetch
    ~~~

    在~目录下编辑.bashrc配置文件 末尾添加一行

    ~~~vim
    neofetch
    ~~~

    > 这样每次打开新终端都会显示终端信息

    查看硬件信息

    ~~~shell
    source ~/.bashrc
    ~~~

3. 进入dwm(至此以下需要在虚拟机内操作 ssh工具无法进入桌面环境)

    ~~~shell
    startx
    ~~~

    > dwm大部分操作都可以用键盘操作 这里有个基础按键 默认为alt  

    基本操作:(这里的基础键和指令键都是可以修改的 修改配置文件在/dwm/config.h)

    > config.h 大概分三个部分 appearance 这里规定了行列间距大小状态下颜色之类
        tagging 这里是dwm左上角各个标签的修改  
        Layout  这里可以修改默认快捷键

    alt + p 类似windows下的快速启动 例如 alt + p 后输入 chrome 就会打开 chrome了浏览器 输入 code 就会打开 VS Code 会有提示  

    alt + shift + enter 打开st dwm下的默认终端模拟器  

    alt + shift + q 退出dwm  

    alt + shift + c 关闭最近打开的一个窗口 默认 后打开 先关闭  

    alt + h 右窗口向左扩大  

    alt + l 左窗口向左扩大  

    alt + j 窗口前进 最近打开窗口 --> 首先打开的窗口  

    alt + k 窗口后退 首先打开的窗口 --> 最近打开窗口  

    ctrl + shift +paUp 增大st终端字号

    alt + f 窗口之间改为浮动

    alt + t 窗口之间改为平铺(默认)

    alt + m 窗口之间改为叠加(全屏下叠加)

    ~~~shell
    xrandr --output Virtual -1 --mode 1920x1080 --rate 60.00
    ~~~

4. 配置dwm
    1. 安装输入法
        > fcitx5的配置修改用命令行复杂 所以选择安装好桌面环境后再安装输入法(在图形界面下设置)

        ~~~shell
        sudo pacman -S fcitx5
        sudo pacman -S fcitx5-im fcitx5-chinese-addons fcitx5-material-color
        ~~~

        > fcitx5-chinese-addons中文支持 fcitx5-material-color皮肤管理器
        设置fcitx5 --> super(这里为基础键) + p -->fcitx5-config-qt
        - [x] Only Show Current Language 这里去掉选中 显示更多语言
        - [ ] Pinyin -- > 双击 --> Apply

        > Global Options 这里可以设置关于fcitx5的默认按键  
        > Addons 这里可以设置输入法大小和主题(主题可以使用自带字体或者去AUR上面下载主题)
        重启使输入法生效

        ~~~shell
        sudo reboot
        ~~~

        配置环境变量 这里参考[Arch.Wiki](https://wiki.archlinux.org/title/Fcitx5_(%E7%AE%80%E4%BD%93%E4%B8%AD%E6%96%87)#%E8%BE%93%E5%85%A5%E6%B3%95%E6%A8%A1%E5%9D%97)
        编辑配置文件

        ~~~shell
        sudo vim /etc/environment
        ~~~

        添加以下几行

        ~~~vim
        GTK_IM_MODULE=fcitx
        QT_IM_MODULE=fcitx
        XMODIFIERS=@im=fcitx
        SDL_IM_MODULE=fcitx
        GLFW_IM_MODULE=ibus
        ~~~

        开启云拼音(输入法联想)
        1. 打开fcitx5设置 --> Addons

        2. Input Method --> pinyin -->选中 Enable Clound Pinyin  
           cloud pinyin -> configure --> 引擎设置为由googlecn -> baidu
        下载离线字库

        ~~~shell
        sudo pacman -S fcitx5-pinyin-moegirl fcitx5-pinyin-zhwiki
        ~~~

        设置开启dwm之前启动输入法

        ~~~shell
        cd ~
        sudo vim .xinitrc
        ~~~

        在exec dwm之前添加

        ~~~vim
        # fcitx5
        fcitx5 &
        ~~~

    2. 修改基础键改为win(这里可以忽略)

        ~~~shell
        #define MODKEY Mod1Mask
        ~~~

        --> 这样就把基础键由alt改为了win键(super键)

        ~~~shell
        #define MODKEY Mod4Mask
        ~~~

        退出编辑 重新安装并重启

        ~~~shell
        sudo make clean install
        ~~~

        > shift + alt + q

        ~~~shell
        startx
        ~~~

    3. 添加壁纸
        > super + p 输入chrome(会有提示 提示为包含chrome的软件包名) 测试下

        ~~~shell
        sudo pacman -S archlinux-wallpaper
        ~~~

        > 这里包中有许多arch相关的壁纸 路径为(/usr/backgrounds/archlinux/)或者 自己自定义壁纸就不需要下载这个包(需要手动下载到本地 想要设置随机壁纸需要把所有壁纸放在同一个文件目录下)

        在文件目录下 feh --bg 查看图片

        添加到xorg-init的配置文件xinitrc中
        > 这里启动dwm桌面之前就会设置 分辨率 壁纸

        ~~~shell
        cd ~
        sudo vim .xinitrc
        ~~~

        在exec dwm之前添加

        ~~~vim
        #xrandr
        xrandr --output Virtual-1 --mode 1920x1080 --rate 60.00
        # feh
        feh --bg-fill --randomize /usr/share/backgrounds/archlinux/*  
        ~~~

        > --randomize代表随机 如果选定某一张壁纸可以不添加这个参数 路径为壁纸绝对路径 --output 参数后跟显示器名称 --mode 分辨率 --rate 刷新率
    4. 修改status bar(状态栏)中文显示字号
        > 例如打开chrome status bar(状态栏)会显示chrome正在访问的网站名 但是中文和英文字号不一致

        ~~~shell
        sudo pacman -S wqy-microhei ttf-nerd-fonts-symbols
        ~~~

        > 安装文泉驿雅黑和矢量图标相关字体
        编辑dwm配置文件 -->sudo vim /home/leegl/ProgramFiles/dwm/config.h
        static const char *fonts[]= {};在这个数组中追加

        ~~~vim
        "WenQuanYi Micro Hei:size=10:type=Regular:antialias=true:antohint=true",
        "Symbols Nerd Font:poxelsize=14:type=2048-em:antialias=true:antohint=true"
        ~~~

        > 这里规定了文泉驿雅黑和图标在status bar(状态栏)中的字体显示
        > 完成操作后重新安装make clean install

    5. 工作空间图标配置(可以忽略)

        > dwm默认9个工作区 以数字为工作区名 这里可以通过修改配置文件 修改数量 或通过矢量图标修改工作区默认名
        这里通过[nerdfonts --> 它包含大部分常用的图标](https://www.nerdfonts.com/cheat-sheet) 搜索相关矢量图标 -->选择 点击icon 就会复制到粘贴板

        修改sudo vim /home/leegl/ProgramFiles/dwm/config.h配置文件中(修改完需要重现安装)
        static const char *tags={}:这个数组 替换掉默认的"1" "2" "3"修改图标或者数量

    6. 使用slstatus 配置status bar(状态栏)

        > slstatus是dwm官方提供的一个可以修改状态栏的工具

        > 下载slstatus --> 在和dwm st同级目录下 这里是ProgramFiles

        下载安装

        ~~~shell
        cd /home/leegl/ProgramFiles
        git clone https://git.suckless.org/slstatus
        cd slstatus
        sudo make clean instal
        ~~~

        修改为启动dwm之前启动
        在home目录下.xinintc
        > 在exec dwm之前添加

        ~~~vim
        # status bar
        exec slstatus &
        ~~~

        退出dwm并重新启动
        > 此时状态栏由显示dwm版本 改为 显示当前 时间 这里表示slstatus已经生效 显示时间是slstatus的默认设置 通过修改在目录 vim /slstatus/config.h下可以添加或修改设置

    7. 为状态栏添加彩色表情emoji(可以忽略)

        > 安装JoyPixels

       ~~~shell
       paru libxft-bgra
       ~~~

       规定JoyPixels在状态栏的显示  

       编辑dwm配置文件 -->sudo vim /home/leegl/ProgramFiles/dwm/config.h
        static const char *fonts[]= {};在这个数组中追加

        ~~~vim
        "JoyPixels:pixelsize=12:type=Regular:antialias=true:antohint=true"
        ~~~

        >dwm 默认不允许使用emoji字体 这里需要取消这个设置 在dwm安装目录下

        ~~~shell
        sudo vim drw.c
        ~~~

        > 找到FcBool isacol;这一行 注释到含有空格行之前 之后dwm需要重新安装

        ~~~vim
        # return NULL;
        ~~~

        > [emoji网站](https://www.unicode.org/emoji/charts/)相关emoji可以在这里搜索到

        > 此时状态栏所有图标相关都可以修改为emoji表情字体(修改方式和矢量图标类似)

    8. 通过补丁(修改原文件)为dwm修改设置
        > dwm的目录下存在.git文件 通过git合并分支的方式将补丁安装 在安装目录下/dwm/

        ~~~shell
        make clean
        git config user.name name
        git config user.email "emiall"
        ~~~

        > 这里登陆下git 后续才可以对本地仓库操作 操作逻辑就是我们建立分支 --> 通过下载补丁对分支修改 --> 合并到主分支 --> make install

        将之前的修改合并进master分支

        ~~~shell
        git branch config
        git checkout config
        sudo cp config.h config.def.h
        git add config.def.h
        git commit -m config
        git checkout master
        git merge config -m config
        sudo make clean && rm -f config.h
        ~~~

        安装 fullgaps 补丁(所有的补丁安装方式都可以参考这个)
        > 这个补丁可以令不同窗口之间存在间距 并通过命令调节间距大小

        ~~~shell
        git branch fullgaps
        git checkout fullgaps
        ~~~

        [dwm官网 --> patches --> fullgaps](http://dwm.suckless.org/patches/fullgaps/) 选择版本后下载到dwm目录下 例如dwm-fullgaps-20200508-7b77734.diff

        打补丁过程

        ~~~shell
        patch -p1 -F  0 < dwm-fullgaps-20200508-7b77734.diff
        ~~~

        > 过程中会提示那些文件修改 我们需要将被修改的文件存放到暂存区并提交 之后合并到主分支(可能会存在冲突)

        ~~~shell
        git add config.def.h dwm.1 dwm.c
        git commit -m fullgaps
        git checkout master
        git merge fullgaps -m fullgaps
        sudo make clean install
        ~~~

        解决代码冲突 设置终端透明效果

        ~~~shell
        sudo pacman -S picom kdiff3
        ~~~

        > picom可以令终端实现半透明效果 kdiff3是一个代码比对工具用于合并冲突

        设置pocpm在dwm之前运行 -b后台运行
        在~目录下 "sudo vim .xinitrc" 添加

        ~~~vim
        # picom
        picom -b
        ~~~

        > 虚拟机下需要修改下picom的配置文件 sudo vim /etc/xdg/picom.conf 注释掉一行

        ~~~vim
        vsync = true
        ~~~

        > git存在默认的解决冲突的软件 这里将它修改为kdiff3

        ~~~shell
        git config --global merge.tool kdiff3
        ~~~

        为st下载补丁
        > 选择下版本  

        [dwm官网 --> st -->patches --> alpha](http://st.suckless.org/patches/alpha/) 这个补丁可以更改背景的不透明度

        [st -->patches --> anysize](http://st.suckless.org/patches/anysize/)
         这个补丁可以结局st终端和其他程序之间的间隙

        [st -->patches --> blinking_cursor](http://st.suckless.org/patches/blinking_cursor/) 这个补丁可以修改st终端的闪烁光标样式  

        [st -->patches --> xresources](http://st.suckless.org/patches/xresources/)
        这个补丁可以修改st终端的配色

        > 登录git 清理下不需要的文件 开始操作

        ~~~shell
        make clean && rm -f config.h && git reset --hard origin/master
        git branch config
        git checkout config
        sudo vim config.def.h
        ~~~

        修改两处(可以忽略)
        > pixelsize=22 这里修改了st下默认字号 int cursorshape = 4;这里修改了光标的样式

        ~~~vim
        pixelsize=22
        int cursorshape = 4;
        ~~~

        继续

        ~~~shell
        git add config.def.h
        git commit -m config
        git checkout master
        git branch alpha
        git checkout alpha
        patch  < st-alpha-20220206-0.8.5.diff
        git add config.def.h config.mk st.h x.c
        git commit -m alpha
        git checkout master
        git branch anysize
        git checkout anysize
        patch < st-anysize-20220718-baa9357.diff 
        git add x.c
        git commit -m anysize
        git checkout master
        git branch blinking_cursor
        git checkout blinking_cursor
        patch  < st-blinking_cursor-20211116-2f6e597.diff
        ~~~

        这里提示出现一个错误(不一定)
        > 这里会生成一个错误文件.re 或者 错误文件.rej 打开后显示了的是补丁和本地文件的冲突 例如 变量名不同 缺失变量 需要手动修改的地方会有+号 错误的行号在文件头@@后有标注 这里通过vim tabnew 打开新的文件 然后手动修改错误

        ~~~shell
        git add config.def.h x.c
        git commit -m blinking_cursor
        git checkout master
        git branch xresources
        git checkout xresources
        patch  < st-xresources-20200604-9ba7ecf.diff
        git add config.def.h x.c
        git commit -m xresources
        git checkout master
        git merge config -m config
        git merge alpha -m alpha
        git merge anysize -m anysize
        git merge blinking_cursor -m blinking_cursor
        ~~~

        > 这里合并分支提示出现错误 错误来自于config.def.h
        利用kdiff3解决冲突

        ~~~shell
        git mergetool
        git merge xresources -m xresources
        sudo make clean install
        ~~~

        > 重新安装后重启

        修改st配色

        >在~目录下新建

        ~~~shell
        sudo vim ~/.Xresources
        ~~~

        在st目录下

        ~~~shell
        sudo vim config.h
        ~~~

        > 找到resources 其中包括color 0-15 background foreground color256 color257 共20项 寻找到合适的主题并修改.Xresources中的颜色设置
        > 例如sudo vim ~/.Xresources中编辑颜色例如 *.color0: #000000
        设置完成后重新加载文件重启dwm后生效

        ~~~shell
        xrdb ~/.Xresources
        ~~~

        例如

        ~~~shell
        # theme
        *.color0: "#3b4252"
        *.color1: "#bf616a"
        *.color2: "#a3be8c"
        *.color3: "#ebcb8b"
        *.color4: "#81a1c1"
        *.color5: "#b48ead"
        *.color6: "#88c0d0"
        *.color7: "#e5e9f0"
        *.color8: "#4c566a"
        *.color9: "#bf616a"
        *.color10: "#a3be8c"
        *.color11: "#ebcb8b"
        *.color12: "#81a1c1"
        *.color13: "#b48ead"
        *.color14: "#8fbcbb"
        *.color15: "#eceff4"
        *.background: "#2e3440"
        *.foreground: "#d8dee9"
        *.color256: "#cccccc"
        *.color257: "#555555"

        ~~~
