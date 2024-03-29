---
author: Ryo
title: RouterOS 通过 OSPF IP分流非CN流量
date: 2022-05-17T120:47:11+0800
lastmod: 2023-03-09T15:31:37+0800
description: "ROS ip 分流方案"
tags: ["RouterOS","ROS", "Debian","OSPF","BRID","ip分流"]
projects: ["Homelab"]
---

主要适用于不想折腾PS/XBOX/Switch设备网关配置,又不希望出口网关为卵漏油/AIB的,或者怕折腾网络时家里断网影响家人的场景

### 更新    
- 2022-06-18: 高版本ROS建立邻居困难问题建议尝试PTP/PTMP模式,并且对齐两侧参数(hello、dead count等),并检查224.0.0.0组播地址范围有无防火墙/路由规则影响导致的不可达问题
- 2023-01-28: 更新探针脚本, 调整文档结构

### 相关硬件
- Mikrotik AC^2 (ros stable 7.2.3*)
- NUC5i5MYBE (debian 11)

    **该版本OSPF似乎有bug,测试邻居离线时出现过一次不能自动删除路由,建议还是LT版本*

### 网络结构
```
AC^2--ETH1(192.168.0.254)--(192.168.0.1)光猫--Internet
    |
    |-ETH2-| birdge:lan
    |-ETH3-| ip:192.168.1.1
    |-ETH4-|----> 192.168.1.0/24 SW
    |
    |-ETH5(192.168.255.1)--(192.168.255.254)NUC
```

相关网段划分:
- 192.168.0.0/24 --光猫/WAN口网段
- 192.168.1.0/24 --普通设备网段
- 192.168.255.0/24 --NUC网段
### 基础防火墙配置
不建议无脑抄,大部分配置有comment,建议按需添加
 
LAN包含lan桥ether5,WAN包括pppoe-out
<details>
  <summary>ROS IPV4防火墙(较长已折叠)</summary>

```
/ip firewall address-list
add address=0.0.0.0/8 comment="defconf: RFC6890" list=no_forward_ipv4
add address=169.254.0.0/16 comment="defconf: RFC6890" list=no_forward_ipv4
add address=224.0.0.0/4 comment="defconf: multicast" list=no_forward_ipv4
add address=255.255.255.255 comment="defconf: RFC6890" list=no_forward_ipv4
add address=192.168.233.233 list=BOT
add address=192.168.0.0/16 list=private
add address=10.0.0.0/8 list=private
add address=172.16.0.0/12 list=private
/ip firewall filter
add action=drop chain=input comment=ANTI-BOT disabled=yes src-address-list=\
    BOT
add action=accept chain=input comment=Accept-OSPF protocol=ospf
add action=accept chain=input comment=Accept-LAN-DNS dst-address=192.168.1.1 \
    dst-port=53 protocol=udp
add action=accept chain=input comment=Accept-NTP dst-port=123 protocol=udp
add action=drop chain=input comment=BLOCK-DNS dst-port=53 in-interface-list=\
    WAN protocol=udp
add action=drop chain=input comment=BLOCK-DoT dst-port=53 in-interface-list=\
    WAN protocol=tcp
add action=drop chain=input comment=BLOCK-HTTP dst-port=80 in-interface-list=\
    WAN protocol=tcp
add action=drop chain=input comment=BLOCK-HTTPS dst-port=443 \
    in-interface-list=WAN protocol=tcp
add action=accept chain=input comment=\
    "defconf: accept established,related,untracked" connection-state=\
    established,related,untracked
add action=drop chain=input comment="defconf: drop invalid" connection-state=\
    invalid
add action=accept chain=input comment="defconf: accept ICMP" log-prefix=\
    IPv4-ping protocol=icmp
add action=accept chain=input comment=\
    "defconf: accept to local loopback (for CAPsMAN)" dst-address=127.0.0.1
add action=drop chain=input comment="defconf: drop all not coming from LAN" \
    in-interface-list=!LAN
add action=accept chain=forward comment="defconf: accept in ipsec policy" \
    ipsec-policy=in,ipsec
add action=accept chain=forward comment="defconf: accept out ipsec policy" \
    ipsec-policy=out,ipsec
add action=fasttrack-connection chain=forward comment="defconf: fasttrack" \
    connection-state=established,related hw-offload=yes
add action=accept chain=forward comment=\
    "defconf: accept established,related, untracked" connection-state=\
    established,related,untracked
add action=drop chain=forward comment="defconf: drop invalid" \
    connection-state=invalid
add action=drop chain=forward comment=\
    "defconf: drop all from WAN not DSTNATed" connection-nat-state=!dstnat \
    connection-state=new in-interface-list=WAN
add action=drop chain=forward comment="defconf: drop bad forward IPs" \
    src-address-list=no_forward_ipv4
add action=drop chain=forward comment="defconf: drop bad forward IPs" \
    dst-address-list=no_forward_ipv4
/ip firewall nat
add action=masquerade chain=srcnat comment="Hairpin NAT" dst-address=\
    192.168.1.0/24 out-interface=lan src-address=192.168.1.0/24
add action=masquerade chain=srcnat comment=ONU dst-address=192.168.0.0/24 \
    src-address=192.168.1.0/24
add action=masquerade chain=srcnat comment="defconf: masquerade" \
    ipsec-policy=out,none out-interface-list=WAN
```
</details>

### DNS解析

#### ROS配置
使用netwatch检查NUC状态,离线时自动将dns切换为公共dns
```routeros
/system script
add dont-require-permissions=yes name=use-default-dns owner=admin policy=\
    ftp,reboot,read,write,policy,test,password,sniff,sensitive,romon source="/\
    ip/dns/set allow-remote-requests=yes cache-size=4096KiB servers=119.29.29.\
    29,223.5.5.5 \r\
    \n/ip dns cache flush"
add dont-require-permissions=yes name=use-sgw-dns owner=admin policy=\
    ftp,reboot,read,write,policy,test,password,sniff,sensitive,romon source="/\
    ip/dns/set allow-remote-requests=yes cache-size=4096KiB servers=192.168.25\
    5.254\r\
    \n/ip dns cache flush"
/tool netwatch
add down-script=use-default-dns host=192.168.255.254 interval=30s up-script=use-sgw-dns
```
#### (可选)192.168.1.0/24网段dns劫持
```routeros
/ip firewall nat
add action=dst-nat chain=dstnat comment="Hijacking for DNS" dst-port=53 \
    protocol=udp src-address=192.168.1.0/24 to-addresses=192.168.1.1 \
    to-ports=53
```
#### Debian配置
SmartDNS分流解析
```bash
apt install smartdns
```
配置/etc/smartdns/smartdns.conf 如下
```bash
bind :53
cache-size 0
prefetch-domain no
serve-expired no
speed-check-mode ping,tcp:80

log-level info
log-file /var/log/smartdns.log
server 223.5.5.5:53
server 119.29.29.29:53
server-https https://dns.google/dns-query -check-edns -blacklist-ip -group GFW -exclude-default-group
server-https https://1.1.1.1/dns-query -blacklist-ip -group GFW -exclude-default-group
server-https https://cloudflare-dns.com/dns-query -blacklist-ip -group GFW -exclude-default-group
conf-file /etc/smartdns/gfw.conf
```
/etc/smartdns/gfw.conf 使用crontab更新
```bash
0 2 * * * curl -s https://raw.githubusercontents.com/RyoLee/gfwlist2smartdns/master/run.sh | bash -s -- -o /etc/smartdns/gfw.conf && /etc/init.d/smartdns reload
```
最终效果list内走223.5.5.5/119.29.29.29,list外走DoH

### OSPF路由

#### ROS配置
添加防火墙放行
```routeros
/ip firewall mangle
add action=mark-routing chain=prerouting new-routing-mark=main passthrough=\
    yes protocol=ospf
add action=mark-routing chain=prerouting dst-address-list=!private \
    in-interface=ether5 new-routing-mark=bypass passthrough=yes
```
Bypass路由表 & OSPF
```routeros
/routing ospf instance
add disabled=no name=sgw router-id=192.168.255.1 routing-table=main
/routing ospf area
add disabled=no instance=sgw name=sgw
/routing table
add disabled=no fib name=bypass
/ip route
add comment="BYPASS" disabled=no distance=1 dst-address=0.0.0.0/0 \
    gateway=pppoe-out pref-src=0.0.0.0 routing-table=bypass scope=30 \
    suppress-hw-offload=no target-scope=10
/routing ospf interface-template
add area=sgw disabled=no interfaces=ether5
/routing rule
add action=lookup disabled=no routing-mark=bypass table=bypass
add action=lookup disabled=no table=main
```
#### Debian配置
安装bird2
```bash
apt install bird2
```

配置/etc/bird/bird.conf (网卡为enp0s25)
```
log syslog all;

router id 192.168.255.254;


protocol device {
	scan time 60;
}

protocol kernel {
	ipv4 {
	      import none;
	      export all;
	};
}

protocol static {
	ipv4;
	include "routes4.conf";
}

protocol ospf v2 {
    ecmp no;
  	ipv4 {
		export all;
	};
	area 0.0.0.0 {
		interface "enp0s25" {
			type broadcast;
		};
	};
}
```
非CN流量出口为虚拟设备为utun,若使用TProxy则配置为对应的以太网网卡(如本例中enp0s25)

`/etc/bird/routes4.conf` 使用crontab更新
```bash
0 2 * * * curl -s https://api.9-ch.com/ncr?device=utun  -o /etc/bird/routes4.conf
0 3 */1 * * /etc/init.d/bird reload
```

心跳检查,若google 204服务检查失败则停止bird触发回落

探针脚本路径 `/opt/check/tun.sh` (已删除bark推送部分, 可自行按需添加)
```bash
#!/usr/bin/bash
COUNT=0
MAX_COUNT=6
while [ $COUNT -lt $MAX_COUNT ]
do
    SER=0
    NET=0
    if [ $(curl --connect-timeout 10 --interface utun -w "%{http_code}" -s https://www.google.com/generate_204) -eq 204 ];then
        NET=1
    fi
    if /etc/init.d/bird status|grep Active|grep -q running;then
        SER=1
    fi
    if [ $NET -eq 1 ] && [ $SER -eq 0 ];then
        /etc/init.d/bird start
        # curl -X "POST" "https://******/push" -H 'Content-Type: application/json; charset=utf-8' -d $'{"body": "***隧道已上线","device_key": "******","level": "passive","title": "网络探针"}'
        exit 0
    fi
    if [ $NET -eq 0 ] && [ $SER -eq 1 ];then
        let COUNT+=1
        if [ $COUNT -eq $MAX_COUNT ];then
            /etc/init.d/bird stop
            # curl -X "POST" "https://******/push" -H 'Content-Type: application/json; charset=utf-8' -d $'{"body": "***隧道已离线","device_key": "******","level": "passive","title": "网络探针"}'
            exit 0
        fi
        continue
    fi
    exit 0
done
```
```bash
* * * * * bash /opt/check/tun.sh
```
### ROS DDNS失效问题
见前文[ros-ddns失效](https://blog.9-ch.com/post/homelab-ros-ospf/#ros-ddns%e5%a4%b1%e6%95%88)

### 参考信息

#### 同网段分流openwrt方案

见前文[RouterOS 通过 OSPF IP分流至 OpenWrt](https://blog.9-ch.com/post/homelab-ros-ospf/)

零散问题比较多, 本方案为基于以上测试经验改进出新方案,非CN IP流量出口改为使用运行debian的一台NUC
