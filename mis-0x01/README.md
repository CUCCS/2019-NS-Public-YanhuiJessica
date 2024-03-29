# OpenWrt on VirtualBox
## 实验过程
### 网络拓扑
![网络拓扑图](img/topology.jpg)
### 获得OpenWrt虚拟硬盘文件
```bash
# 以下命令在wsl(尽量不要使用)中执行

# 下载镜像文件
wget https://downloads.openwrt.org/releases/18.06.4/targets/x86/64/openwrt-18.06.4-x86-64-combined-squashfs.img.gz # 也可以直接前往官网下载
# 解压缩
gzip openwrt-18.06.4-x86-64-combined-squashfs.img.gz

# 直接将解压出来的.img文件转化为.vdi文件会出现VERR_VD_INVALID_SIZE错误, 根据官方安装教程的解决方案, 需要先用dd(主要功能为转换和复制文件)填充
dd if=openwrt-18.06.4-x86-64-combined-squashfs.img of=openwrt-18.06.4.img bs=128000 conv=sync
# bs: 输入/输出块大小为128000字节
# conv=sync: 将每个输入块填充到128000字节, 不足部分用空(NUL)字符补齐

VBoxManage convertfromraw --format VDI openwrt-18.06.4.img openwrt-18.06.4.vdi
# 调整至合适的大小, 在此之前需要在虚拟机上注册该虚拟硬盘
VBoxManage modifymedium openwrt-18.06.4.vdi --resize 128
```
### 虚拟机配置
- ![OpenWrt硬件配置](img/openwrt-network.jpg)
- ![host-only网卡配置信息](img/host-only-settings.jpg)
- 修改`/etc/config/network`配置信息(除了修改ipaddr还删除了接口`'wan6'`的配置):<br>
  ![修改后的配置](img/network-settings.jpg)<br>
  `lan`端口只支持静态配置IP
- 安装好`luci`后即可访问:<br>
  ![OpenWrt登录界面](img/openwrt-login-page.jpg)<br>
  初始登录是不需要密码的, 登入后一定要记得设置密码:<br>
  ![OpenWrt登录设置](img/openwrt-admin-settings.jpg)
### 开启AP功能
- 插入无线网卡并分配到OpenWrt虚拟机上, 在OpenWrt虚拟机中执行:
  ```bash
  opkg update

  # 为使用lsusb命令
  opkg install usbutils

  lsusb
  # Bus 001 Device 002: ID 0846:9030 NetGear, Inc. WNA1100 Wireless-N 150 [Atheros AR9271]
  # 可以看到无线网卡的芯片名称为AR9271

  # 前往官网查看 AR9271 对应驱动为kmod-ath9k-htc
  opkg install kmod-ath9k-htc

  # 查看 USB 外设的驱动加载情况
  lsusb -t
  # /:  Bus 01.Port 1: Dev 1, Class=root_hub, Driver=ehci-pci/12p, 480M
  #  |__ Port 1: Dev 2, If 0, Class=Vendor Specific Class, Driver=ath9k_htc, 480M
  # 成功安装

  # 让 OpenWrt 支持 WPA 系列更安全的无线安全机制
  opkg install wpa-supplicant hostapd
  ```
- 通过`OpenWrt`的`LuCI`完成无线网络的配置, 手工指定监听信道和信号强度:<br>
  ![手工指定监听信道和信号强度](img/general-setup.jpg)<br>如果`Status`显示`Wireless is disabled`, 可以重启(先`Disable`再`Enable`)`Wireless network`
- `Wireless Security`选择加密方式为`WPA-PSK/WPA2-PSK Mixed Mode`
### 其他人接入网络
- 当没有用户接入时:<br>
  ![无用户接入](img/wireless-none.jpg)
- 有用户接入时:<br>
  ![有两个用户接入](img/wireless-associations.jpg)
### AP配置
- [x] 重置和恢复AP到出厂默认设置状态
  ```bash
  firstboot && reboot now
  ```
- [x] 设置AP的管理员用户名和密码
  - [x] 设置AP管理员的用户名
    - 添加新用户:
      ```bash
      opkg update && opkg install shadow-useradd
      useradd acroot
      passwd acroot  #change the password of user acroot

      vi /usr/lib/lua/luci/controller/admin/index.lua
      # 将page.sysauth = "root" 修改为 page.sysauth = {"root","acroot"}

      vi /etc/config/rpcd
      # 添加
      # config login
      #   option username 'acroot'
      #   option password '$p$acroot'
      #   list read '*'
      #   list write '*'

      reboot
      ```
    - 可以成功使用`acroot`登录, 但此时`root`也能登录, 相当于增加了一个`root`类型的用户:<br>
      ![成功登录记录](img/acroot-success-login.jpg)
    - 将`/usr/lib/lua/luci/controller/admin/index.lua`中的`page.sysauth`修改为`page.sysauth = "acroot"`并删除`/etc/config/rpcd`中`root`的配置信息后再重启, 无法再使用`root`登录, `acroot`仍能正常登陆
  - [x] 设置AP管理员的密码
    - 可以通过`passwd`在虚拟机内设置, 也可以使用`luCI`:<br>
      ![设置/修改密码](img/admin-passwd.jpg)
- [x] 设置SSID广播和非广播模式
  - 默认是SSID广播
  - 设置SSID非广播模式, 修改配置文件`/etc/config/wireless`:<br>
    ![设置SSID为非广播模式](img/set-ssid-not-broadcast.jpg)<br>
    配置成功后, 使用`wifi down`和`wifi up`重启。
  - 关闭SSID广播后, 在AC-Pitfall正常的情况下, 无法扫描到网络:<br>
    ![无法扫描到AC-Pitfall](img/no-ac-pitfall.jpg)<br>
    但手机可以正常连接:<br>
    ![正常连接](img/ac-pitfall-connected.jpg)
    - Plus: 使用`date -R && iw dev wlan0 scan | grep SSID | sort | uniq && date -R`更好, 更精确的表明中间命令的执行时间
  - 再次开启广播, 可以将配置文件中`option hidden '1'`删除或者置为`0`
  - 也可以通过`OpenWrt`的图形界面进行管理:<br>
    ![图形界面设置SSID广播与非广播](img/hide-esside.jpg)
- [x] 配置不同的加密方式
  - 在`Interface Configuration`下的`Wireless Security`中修改加密方式:<br>
    ![配置不同的加密方式](img/ap-encryption.jpg)
- [x] 设置MAC地址过滤规则(ACL(访问控制表)地址过滤器)
  ![设置MAC地址过滤规则](img/MAC-Address-Filter.jpg)<br>
  可选有: `disable`/`Allow listed only`/`Allow all except listed`
## 其它问题
### 使用`opkg update`时遇到`wget returned 4`错误
- `opkg_download: Check your network settings and connectivity.`查看网络连接正常, `ping baidu.com`也可`ping`通, 多次尝试`opkg update`仍然失败。
- 解决办法: 先使用`wget`下载`opkg update`安装失败的安装包
### 新添加的Host-Only网卡无法使用
- 在[ns-0x01](https://github.com/CUCCS/2019-NS-Public-YanhuiJessica/tree/ns0x01/ns-0x01#gateway-debian)中提到通过使用管理员权限重新安装虚拟机来解决Host-Only网卡无法使用的问题, 但这个方法并不是解决方法的首选
- 查看宿主机网络连接, 找到该Host-Only网卡, 对比可以正常使用的Host-Only网卡并无明显差异, 禁用再重新启用, 虚拟机即可正常启动。
### 关闭USB口电量限制
- `设备管理器 -> 通用串行总线控制器 -> USB根集线器(USB 3.0) -> 电源管理`
## 实验总结
- 要在虚拟机中的`OpenWrt`开启无线网络支持或使用无线网络，需要借助无线网卡硬件设备
## 参考资料
- [SOLVED: Failed to download the package list Archer C7 v2 with LEDE 17.01.0](https://forum.openwrt.org/t/solved-failed-to-download-the-package-list-archer-c7-v2-with-lede-17-01-0/2557/5)
- [OpenWrt on VirtualBox HowTo](https://openwrt.org/docs/guide-user/virtualization/virtualbox-vm)
- [dd(1) - Linux man page](https://linux.die.net/man/1/dd)
- [Failsafe Mode, Factory Reset, and Recovery Mode](https://openwrt.org/docs/guide-user/troubleshooting/failsafe_and_factory_reset)
- [DNS and DHCP configuration examples](https://openwrt.org/docs/guide-user/base-system/dhcp_configuration)
- [Wi-Fi /etc/config/wireless](https://openwrt.org/docs/guide-user/network/wifi/basic)
- [SOLVED: LUCI add support user (in addition to root)](https://forum.openwrt.org/t/solved-luci-add-support-user-in-addition-to-root/17402)