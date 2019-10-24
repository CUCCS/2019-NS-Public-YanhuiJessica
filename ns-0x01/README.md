# 基于 VirtualBox 的网络攻防基础环境搭建
## 实验要求
- [x] [虚拟硬盘配置成多重加载](#虚拟硬盘多重加载)
- [x] 网络拓扑图与搭建
- [x] 网络连通性测试
  - [x] 靶机可以**直接**访问攻击者主机
    - 靶机与攻击者主机不在同一个网段, 访问显然需要通过网关
  - [x] 攻击者主机无法**直接**访问靶机
  - [x] 网关可以**直接**访问攻击者主机和靶机
  - [x] 靶机的所有对外上下行流量必须经过网关
  - [x] 所有节点均可以访问互联网
## 实验过程
### 虚拟硬盘多重加载
- 需要先将下载的vdi(普通)文件与本地虚拟机关联,否则会在修改时出现以下报错信息:<br>
  ![多重加载只能用在与创建在4.0及之后版本的虚拟机有关联的介质上](img/errorMessage-1.jpg)
- 管理 -> 虚拟介质管理, 选中需要修改的虚拟硬盘, 修改类型为"多重加载"并应用, 将会先从原来关联的虚拟电脑中释放, 修改成功后再与原来关联的虚拟机建立联系:<br>
  ![添加虚拟硬盘](img/addVdi.jpg)
  可以看到虚拟硬盘类型显示为"多重加载":<br>
  ![多重加载](img/multiattach.jpg)
  ![虚拟硬盘与多个虚拟机建立联系](img/multiAttach.png)
- 可以将一个多重加载类型的虚拟硬盘用于多个虚拟机上, 免去了每次新建的虚拟机都要安装操作系统的烦恼
  - 有一个安装了增强功能的虚拟机, 新建一个与之共用一个多重加载虚拟硬盘的虚拟机, 增强功能并不会沿用到新的虚拟机上
### 网络拓扑
- 网络节点: 攻击者主机、网关、靶机
  ![网络拓扑图](img/topology.png)
### 为内网模式虚拟机分配IP地址
- 刚开始看到内网模式的WinXP虚拟机有IPv4地址而且和Debian网关(除了基本网卡其他都没有配置)可以互通, 就以为内网模式的Kali也应该有, 然而没有:<br>
  ![没有IPv4地址](img/NoIPv4.png)<br>
  以为是~~内网模式Kali的网卡配置问题或者是因为其他原因IPv4地址无法显示~~, 实际上内网模式需要手动将**虚拟网卡配置为使用静态IP地址**, 或者使用DHCP服务器(可以使用Debian的`dnsmasq`, 参考[官方文档](https://wiki.debian.org/dnsmasq#Basic_DHCP_Setup)的操作没有成功, 仅收到了DHCP REQUEST的广播包, 并没有发出DHCP OFFER)来分配IP地址
- 实际上网关已配置好`/etc/network/interfaces`, 由于硬编码, 只要按照规定的顺序配置网关的网卡, 就可以为内网靶机自动分配IP地址
### 各虚拟机网络配置
#### 靶机
##### Victim-XP-1
<!-- ![内网1靶机XP](img/victim-xp-1-network.png) -->
![内网1靶机XP](img/victim-xp-1-network2.png)
##### Victim-Kali-1
<!-- ![内网1靶机Kali](img/victim-kali-1-network.png) -->
![内网1靶机Kali](img/victim-kali-1-network2.png)
##### Victim-XP-2
![内网2靶机XP](img/victim-xp-2-network.png)
#### 攻击者主机
##### Attacker-Kali
- 需要注意的是网卡设置为NAT网络后, 如: 网络CIDR是10.0.2.0/24, 那么将默认设置10.0.2.1为网关。但是攻击者主机只需要与网关互通, 无需访问靶机, 只需要保证与网关的一个端口在同一个网段中, 所以无需修改默认网关。
![攻击者主机Kali](img/attacker-kali-network.png)
#### 网关
##### Gateway-Debian
- 最好不要使用自己添加的Host-Only网卡, 选择已有的Host-Only网卡, 否则会出现以下问题:<br>
  ![VERR_INTNET_FLT_IF_NOT_FOUND](img/hostOnlyError.png)
  <br>尽管可以使用管理员权限重新安装虚拟机来获得正确启动, 但是查看该网卡的IP地址仍然是不正确的。
![网关Debian](img/gateway-debian-network.png)
### 配置端口转发
<!-- - 重启Debian后查看iptables发现内网2的靶机竟然已经自动加入了！查看该靶机发现所有网络配置都是自动获得的, 但是内网1的靶机却不能做到。 查看/etc/network/interfaces配置文件发现两者的规则是不同的:<br>
  ![两个内网不同规则](img/difRule.png)<br>
- 如果没有在网关上配置 DNS 服务器，Internal Network模式主机的 DNS 解需要外部 DNS 服务器。网关只要能实现 DNS 流量的正常 NAT 即可确保 Internal Network模式主机的 DNS 解析。
- iptables.rules配置如下:<br>
  ![iptables.rules](img/iptablesRules.png) -->
- 查看`/etc/network/interfaces`发现enp0s9和enp0s10都已经有配置了:<br>
  ![interfaces配置](img/network-interfaces.png)
- 由于硬编码, 网卡的数量、类型与顺序非常重要, 如果网关不是四张网卡并且顺序依次为: NAT网络、仅主机网络、内部网络1、内部网络2, 那么就需要修改配置文件。
### 连通性测试
#### Victim-XP-1
![Victim-XP-1已连通](img/victim-XP-1-vis.png)
#### Victim-Kali-1
![Victim-Kali-1已连通](img/victim-Kali-1-vis.png)
#### Victim-XP-2
![Victim-XP-2已连通](img/victim-XP-2-vis.png)
#### Attacker-Kali
![Attacker-Kali访问](img/attacker-kali-vis.png)
#### Gateway-Debian
- ![ping和ARP表](img/GWvis.png)
- ![网关可上网](img/gateway-Debian-vis.png)
## 其它问题
### Debian: `arp: command not found`
尝试使用`apt-get`安装`arp`包, 发现不能定位包, `apt-get update`之后也没有用, 使用`whereis`可以找到arp.7.gz。
- 解决方法: 安装`net-tools`包, 包里包含`arp`、`ifconfig`等命令
### Debian: Host-Only网络
- 为网关虚拟机添加Host-Only网络的网卡后, 在虚拟机内查看该网卡对应的IP地址, 不在网卡DHCP服务器对应的网段中, 导致无法使用`ssh`连接虚拟机
- 解决方法: ~~修改虚拟机该网卡对应的IP地址使其属于有效网段(永久修改需要修改配置文件), 或修改DHCP服务器对应的网段~~这是由于配置文件硬编码导致的问题, 应该修改网关网卡的顺序
## 实验总结
### 注意
- 以169.254开头的IP地址是link-local address, 是当DHCP分配失败时自动生成的
### 各个模式的区别
Mode|VM->Host|VM<-Host|VM1<->VM2|VM->Net/LAN|VM<-Net/LAN
-|-|-|-|-|-
Host-Only|+|+|+|-|-
Internal|-|-|+|-|-
NAT|+|Port forward|**-**|+|Port forward
NAT service|+|Port forward|+|+|Port forward
### 端口转发
尽管本次实验中网关已经配置好了端口转发, 还是可以了解一下如何使用`iptables`配置。
- 使用`iptables`设置端口转发(enp0s3端口对应NAT网络的网卡)
  ```
  iptables -t nat -L #选择NAT表并列出所有规则
  iptables-save -c > iptables.rules #将规则写入iptables.rules
  vi iptables.rules #编辑
  -A POSTROUTING -o enp0s3 -s 172.16.111.0/24 -j MASQUERADE #添加到COMMIT前
  #-o enp0s3	this rule is valid for packets that leave on the second network interface (-o stands for "output")
  #-j MASQUERADE	the action that should take place is to 'masquerade' packets, i.e. replacing the sender's address by the router's address.
  iptables-restore < iptables.rules #将规则写回iptables
  ```
- 配置文件`/etc/network/interfaces`中使用`echo 1 > /proc/sys/net/ipv4/ip_forward`的方式开启IPv4端口转发, 也可以将`/etc/sysctl.conf`文件中`# net.ipv4.ip_forward=1`的注释去除再重启, 效果是一样的。
## 参考资料
- [基于VirtualBox的网络攻防基础环境搭建 - 林淑琪](https://github.com/CUCCS/2018-NS-Public-jckling/blob/master/ns-0x01/%E5%9F%BA%E4%BA%8EVirtualBox%E7%9A%84%E7%BD%91%E7%BB%9C%E6%94%BB%E9%98%B2%E5%9F%BA%E7%A1%80%E7%8E%AF%E5%A2%83%E6%90%AD%E5%BB%BA.md)
- [Windows 10 - Host Only Adapters](https://forums.virtualbox.org/viewtopic.php?f=6&t=94277)
- [Chapter 6. Virtual networking](https://www.virtualbox.org/manual/ch06.html#network_nat)
- [-bash: netstat: command not found - Debian/Ubuntu Linux](https://linuxconfig.org/bash-netstat-command-not-found-debian-ubuntu-linux)
- [Example: Connect a private subnet to the internet using NAT](https://www.karlrupp.net/en/computer/nat_tutorial)
- [How to Disable/Enable IP forwarding in Linux](https://linuxconfig.org/how-to-turn-on-off-ip-forwarding-in-linux)
