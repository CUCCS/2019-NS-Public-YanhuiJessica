# 基于 Scapy 编写端口扫描器
## 实验要求
- [x] 完成以下扫描技术的编程实现
  - [x] [`TCP connect scan`](#tcp-connect-scan) / [`TCP stealth scan`](#tcp-stealthsyn-scan)
  - [x] [`TCP Xmas scan`](#tcp-xmas-scan) / [`TCP FIN scan`](#tcp-fin-scan) / [`TCP NULL scan`](#tcp-null-scan)
  - [x] [`UDP scan`](#udp-scan)
## 实验过程
### 网络拓扑
![网络拓扑图](img/topology.jpg)
### 端口状态的模拟
- 关闭状态<br>
对应端口没有开启监听, 防火墙没有开启
- 开启状态<br>
对应端口开启监听, `80`端口可以使用`service apache2 start`, `53`端口可以使用`service dnsmasq start`。防火墙处于关闭状态
- 过滤状态<br>
对应端口开启监听, 防火墙开启
- 本次实验中防火墙的开启与关闭使用的是`ufw`，操作较为简单。当然，直接使用`iptables`也是可以的
### `TCP connect scan`
- 代码
    ```py
    #! /usr/bin/python

    import logging
    logging.getLogger("scapy.runtime").setLevel(logging.ERROR)
    # Used before importing the Scapy module inside the Python program
    # Sets logging only for errors that occur during the program execution and not the warnings.
    from scapy.all import *

    dst_ip = "172.16.111.148"   # ACKAli
    src_port = RandShort()  # 生成一个随机数
    st_port = 80

    tcp_connect_scan_resp = sr1(IP(dst=dst_ip)/TCP(sport=src_port,dport=dst_port,flags="S"),timeout=10)
    if(str(type(tcp_connect_scan_resp))=="<type 'NoneType'>"):  #端口没有响应
        print "Filtered"
    elif(tcp_connect_scan_resp.haslayer(TCP)):
        if(tcp_connect_scan_resp.getlayer(TCP).flags == 0x12):  #Flags: 0x012 (SYN, ACK)
            send_rst = sr(IP(dst=dst_ip)/TCP(sport=src_port,dport=dst_port,flags="AR"),timeout=10)
            print "Open"
        elif (tcp_connect_scan_resp.getlayer(TCP).flags == 0x14):   #Flags: 0x014 (RST, ACK)
            print "Closed"
    ```
- 端口关闭状态<br>
  ![端口关闭扫描结果](img/tcp-connect-scan-closed.jpg)
- 端口开放状态<br>
  ![端口开放扫描结果](img/tcp-connect-scan-open.jpg)
- 端口过滤状态<br>
  ![端口过滤扫描结果](img/tcp-connect-scan-filtered.jpg)
### `TCP stealth/SYN scan`
- 并不打开一个完整的链接, 当得到的是一个`SYN/ACK`包时通过发送一个`RST`包立即拆除连接。
- 代码
    ```py
    #! /usr/bin/python

    import logging
    logging.getLogger("scapy.runtime").setLevel(logging.ERROR)
    from scapy.all import *

    dst_ip = "172.16.111.148"   # ACKAli
    src_port = RandShort()
    dst_port = 80

    stealth_scan_resp = sr1(IP(dst=dst_ip)/TCP(sport=src_port,dport=dst_port,flags="S"),timeout=10)
    if(str(type(stealth_scan_resp))=="<type 'NoneType'>"):
        print "Filtered"
    elif(stealth_scan_resp.haslayer(TCP)):
        if(stealth_scan_resp.getlayer(TCP).flags == 0x12):
            send_rst = sr(IP(dst=dst_ip)/TCP(sport=src_port,dport=dst_port,flags="R"),timeout=10)   #拆除链接
            print "Open"
        elif (stealth_scan_resp.getlayer(TCP).flags == 0x14):
            print "Closed"
    ```
- 端口关闭状态<br>
  ![端口关闭扫描结果](img/tcp-stealth-scan-closed.jpg)
- 端口开启状态<br>
  ![端口开启扫描结果](img/tcp-stealth-scan-open.jpg)
- 端口过滤状态<br>
  ![端口过滤扫描结果](img/tcp-stealth-scan-filtered.jpg)
### `TCP Xmas scan`
- 代码
    ```py
    #! /usr/bin/python

    import logging
    logging.getLogger("scapy.runtime").setLevel(logging.ERROR)
    from scapy.all import *

    dst_ip = "172.16.111.148"   # ACKAli
    src_port = RandShort()
    dst_port = 80

    xmas_scan_resp = sr1(IP(dst=dst_ip)/TCP(dport=dst_port,flags="FPU"),timeout=10) # FIN, PUSH, URG
    if (str(type(xmas_scan_resp))=="<type 'NoneType'>"):
        print "Open|Filtered"
    elif(xmas_scan_resp.haslayer(TCP)):
        if(xmas_scan_resp.getlayer(TCP).flags == 0x14):
            print "Closed"
    ```
- 端口关闭状态<br>
  ![端口关闭扫描结果](img/tcp-xmas-scan-closed.jpg)
- 端口开启状态<br>
  ![端口开启扫描结果](img/tcp-xmas-scan-open.jpg)
- 端口过滤状态<br>
  ![端口过滤扫描结果](img/tcp-xmas-scan-filtered.jpg)
### `TCP FIN scan`
- 代码
    ```py
    #! /usr/bin/python

    import logging
    logging.getLogger("scapy.runtime").setLevel(logging.ERROR)
    from scapy.all import *

    dst_ip = "172.16.111.148"   # ACKAli
    src_port = RandShort()
    dst_port = 80

    fin_scan_resp = sr1(IP(dst=dst_ip)/TCP(dport=dst_port,flags="F"),timeout=10)
    if (str(type(fin_scan_resp))=="<type 'NoneType'>"):
        print "Open|Filtered"
    elif(fin_scan_resp.haslayer(TCP)):
        if(fin_scan_resp.getlayer(TCP).flags == 0x14):
            print "Closed"
    ```
- 端口关闭状态<br>
  ![端口关闭扫描结果](img/tcp-fin-scan-closed.jpg)
- 端口开启状态<br>
  ![端口开启扫描结果](img/tcp-fin-scan-open.jpg)
- 端口过滤状态<br>
  ![端口过滤扫描结果](img/tcp-fin-scan-filtered.jpg)
### `TCP NULL scan`
- 代码
    ```py
    #! /usr/bin/python

    import logging
    logging.getLogger("scapy.runtime").setLevel(logging.ERROR)
    from scapy.all import *

    dst_ip = "172.16.111.148"   # ACKAli
    src_port = RandShort()
    dst_port = 80

    null_scan_resp = sr1(IP(dst=dst_ip)/TCP(dport=dst_port,flags=""),timeout=10)
    if (str(type(null_scan_resp))=="<type 'NoneType'>"):
        print "Open|Filtered"
    elif(null_scan_resp.haslayer(TCP)):
        if(null_scan_resp.getlayer(TCP).flags == 0x14):
            print "Closed"
    ```
- 端口关闭状态<br>
  ![端口关闭扫描结果](img/tcp-null-scan-closed.jpg)
- 端口开启状态<br>
  ![端口开启扫描结果](img/tcp-null-scan-open.jpg)
- 端口过滤状态<br>
  ![端口过滤扫描结果](img/tcp-null-scan-filtered.jpg)
### `UDP scan`
- 代码
    ```py
    #! /usr/bin/python

    import logging
    logging.getLogger("scapy.runtime").setLevel(logging.ERROR)
    from scapy.all import *

    dst_ip = "172.16.111.148"   # ACKAli
    src_port = RandShort()
    dst_port = 53
    dst_timeout = 10

    def udp_scan(dst_ip,dst_port,dst_timeout):
        udp_scan_resp = sr1(IP(dst=dst_ip)/UDP(dport=dst_port),timeout=dst_timeout)
        if (str(type(udp_scan_resp))=="<type 'NoneType'>"):
            return "Open|Filtered"
        elif(udp_scan_resp.haslayer(ICMP)):
            if(int(udp_scan_resp.getlayer(ICMP).type)==3 and int(udp_scan_resp.getlayer(ICMP).code)==3):
            # ICMP Type: 3 (Destination unreachable)
            # ICMP Code: 3 (Port unreachable)
                return "Closed"

    print(udp_scan(dst_ip,dst_port,dst_timeout))
    ```
- 端口关闭状态<br>
  ![端口关闭扫描结果](img/udp-scan-closed.jpg)
- 端口开启状态<br>
  ![端口开启扫描结果](img/udp-scan-open.jpg)
- 端口过滤状态<br>
  ![端口过滤扫描结果](img/udp-scan-filtered.jpg)
## 实验总结
- 每一次扫描测试的抓包结果与课本中的扫描方法原理基本相符
### `Scapy`
- `TCP`中的`flags`参数值填写需要置`1`字段名的首字母(顺序任意), 如`[RST, ACK]`对应`AR`, 也可以填写对应的字段值, 如`flags=0x2`:<br>
  ![TCP flags](img/tcp-flags.jpg)
- ```py
  sr()
  #for sending packets and receiving answers
  #returns a couple of packet and answers, and the unanswered packets

  sr1()
  #only returns one packet that answered the packet (or the packet set) sent
  ```
### `Kali`防火墙`ufw`的使用
```bash
ufw enable  #开启防火墙
ufw disable #关闭防火墙

#ufw deny <port>/<optional: protocol>
ufw deny 80 #过滤80端口上所有包
ufw deny 80/tcp #过滤传入80端口的TCP包

ufw status  #查看当前防火墙的状态和现有规则
```
## 参考资料
- [2018-NS-Public-jckling/ns-0x05](https://github.com/CUCCS/2018-NS-Public-jckling/blob/master/ns-0x05/5.md)
- [Usage — Scapy 3 documentation](https://scapy.readthedocs.io/en/latest/usage.html)
- [Port Scanning using Scapy](https://resources.infosecinstitute.com/port-scanning-using-scapy/)
- [UFW](https://help.ubuntu.com/community/UFW)