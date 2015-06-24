#wndr4300刷openwrt
## 新机刷openwrt的方法
 1. 下载openwrt升级包。
 2. 进入原版系统，在固件升级的配置项，选择刚刚下载的文件进行升级。
 3. 重启就是openwrt系统。
## 救砖方法
 1. 下载openwrt刷机包。
 2. 下载tftp软件[!http://www.wayos.cn/down/other/tftp.rar](下载链接)。
 3. 把电脑用网线连接路由器，电脑改为固定ip:192.168.1.2，子网掩码:255.255.255.0，ip网关:192.168.1.1。
 4. wndr4300用电源键断电。
 5. 用细针按住reset，恢复出厂设置。
 6. 按下电源键，通电。
 7. 直到电源灯由黄色闪烁到绿色闪烁，松开reset。
 8. 使用tftp上传固件：路由器ip填192.168.1.1密码为空，固件就是刚下载的。后刷新固件。
 9. 等待tftp显示绿色圆点，表示刷机成功，再等几分钟，就可以看到电源键变绿色，4300又活了。
 10. 现在可以进入luci管理平台设置上网了。
## 新刷openwrt的初始化设置
 1. 进入luci管理平台，进行一系列的设置尤其是以下几点：
 	* Network --> wireless 设置中选择enbale wireless，并具体配置，从而可以用无线连接。
	* System --> Admin 设置中为root用户设置密码，设置密码后就可以使用SSH方式来访问。
	* 设置时区为上海，用品会变为正确的时间
 2. 使用SSH方式访问4300，例如使用secureCRT客户端，配置好192.168.1.1，使用22端口，使用root用户和上面设置的密码登陆：


 ```
 alias ll = 'ls -l'
 alias diff = cmp
# alias reboot='curl "http://124.205.127.158/F";sleep 1; reboot"

 ```

 执行以下命令，安装luci中文界面：

 ```
 opkg update
 opkg install luci-i18n-chinese

 ```

 安装以下常用命令工具：

 ```
#支持mkfs、ext2、mkfs等的命令
 opkg install e2fsprogs
#支持ext2，ext3，ext4等分区
 opkg install kmod-fs-ext4
```

 执行以下命令将4300被保留的90多M的空间加以利用：

 ```
 mkfs.ext4 /dev/mtdblock11
 mkdir /local
 mount -t ext4 /dev/mtdblock11 /local -o rw, sync

 ```

 上面的方法重启后保留的空间又不见了，故使用开机自动加载：
 把下面的语句加入到``/etc/rc.local``文件中使得每次重启都会加载保留的90M空间到/local目录。

 ```
 mount -t ext4 /dev/mtdblock11 /local -o rw, sync

 ```

 修改``/etc/opkg.conf``,增加以下语句：

 ```
 dest local /local
 ```

 修改``/etc/profule``增加以下语句：

 ```
 export PATH=/usr/bin:/usr/sbin:/bin:/sbin:/local/bin:/local/usr/bin
 export LD_LIBRARY_PATH=/local/lib:/local/usr/lib

 alias opkg='opkg -d local"
 ```

 执行以下命令全得上面的配置立刻生效：

 ```
 source /etc/profile

 #/local/usr 目录下建立链接：
ln -s /usr/share share
 #/locla目录下建立链接：
ln -s /etc etc 
ln -s /www www
```

 这样以后就可以使用opkg命令把工具默认安装local目录了。
 安装常用软件：

 ```
 opkg install kmod-nls-iso8859-1 kmod-nls-utf8 #安装语言组件
 opkg install libncurses
 /bin/opkg -d root install luci-app-commands #安装luci界面的shell执行工具，luci相关的内容必须在／目录下安装，然后重启路由器才能生效
 ```

## 挂载USB硬盘并实现windows共享访问
 安装USB驱动等：

 ```
 opkg update
 opkg install kmod-usb-core
 opkg install kmod-usb-ohci #安装usb ohci控制器驱动
 #opkg install kmod-usb-uhci #uhci usb控制器
 opkg install kmod-usb2 #安装usb2.0
 opkg install kmod-usb-storage #安装usb存储设置驱动
 opkg install usbutils #安装这个后可以用一些工具，如：lsusb
 opkg install kmod-fs-ntfs #ntfs内核驱动
 opkg install mount ntfs-3g #挂载ntfs的助手
 opkg install mount-utils #挂载卸载工具
 opkg install ntfs-3g #挂载ntfs
 opkg install kmod-fs-vfat #挂载fat
 opkg install fdisk #硬盘分区管理工具
 opkg install blkid #用于查看usb设备uuid信息 #blkid
 opkg install block-mount #安装block-mount，安装之后luci的System-mount points下可以直接查看挂载点信息。
```
 然后可以插上移动硬盘，可以通过以下命令加载

 ```
 lsusb #查看usb设置
 mkdir /mnt/g500 #创建加载目录
 ntfs-3g /dev/sda1 /mnt/g500 -o noatime, big_writes, async #加载移动硬盘，且使用big_writes等参数提高性能
```
 需要在/lib目录设置libpthread.so.0 --> /local/lib/libpthread-0.9.33.2.s9才能mount进行加载：

 ```
 cd /lib
 ln -s /local/lib/libpthread-0.9.33.2.so libpthread.so.0
 ```

 将加载移动硬盘的命令加入到rc.local中，以完成开机自动加载：
 
 ```
 mount -t ntfs-3g /dev/sda1 /mnt/g500 -o noatime, big_writes,async
 ```

 安装samba共享：

 ```
 opkg install luci-app-samba
 sed -i -e ' s@/usr/sbin/smbd@/local/usr/sbin/smbd@' -e 's@/usr/sbin/nmbd@/local/usr/sbin/nmbd@' /etc/init.d/samba #替换为正确的执行路径
 /etc/init.dsamba enable #该句我法成功使其开机自启动，因为此时/local目录还没有被加载，故实际要加入rc.local中。
 /etc/init.d/samba restart #启动samba
 ```
 注意： 因为samba是安装在local目录，故启动时执行init.d/samba时并没有完成加封local目录，不会成功。解决办法是将启动samba命令加入到rc.local中。
 然后重启路由器，再进入luci管理界面，就可以看到多出来一个“服务”菜单，下面有“网络共享”，即是samba的配置。

 修改模板以增加对中文目录的支持：

 ```
 #添加以下项
 unix charset = utf-i
 dos charset = cp936
 #去掉原来的unix charset项
 ```

 添加共享目录后即可在windows下访问，例如添加：

 ```
 共享名：openwrt
 目录：/tmp
 允许匿名用户访问：是
 其他：为空
 ```

 然后可以在windows下的explorer中输入如下路径访问路由器的/tmp目录（可读写）

 ```
 \\192.168.1.1\openwrt
 ```

 对于g500目录也做类似设置，以方便访问。

## 安装迅雷离线下载
 写到这不想写下去了。后面再写吧。
