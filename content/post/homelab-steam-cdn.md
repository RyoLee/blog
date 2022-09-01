---
author: Ryo
title: 通过DNS与Nginx锁定并加速Steam下载(CDN)
description: "通过DNS与Nginx锁定并加速Steam下载(CDN)"
tags: ["Steam","CDN", "Nginx","DNS"]
projects: ["Homelab"]
---

### 为什么要手动锁定

当使用透明网关对内网进行加速时(包括部分游戏加速器的特定工作模式),会因为Steam根据当前ip判断自动切换到ip对应地区的下载CDN,造成隧道流量的浪费甚至产生反效果导致下载缓慢

### 如何判断是否需要使用该方案

最简单的办法就是不做任何其他配置的情况下(若使用加速器需要先打开加速器),打开 ```Steam--设置--下载``` 检查下载地区,若显示为任意大陆地区城市则大概率不需要本方案修改(这个选项城市其实是假的,只要是任意大陆城市本质都一样)

若显示为透明网关对应隧道/加速器ip地区,则可以考虑使用本方案

另外也可以通过 ```Win+R``` 执行 ```steam://open/console``` 调出Steam控制台来查看信息判断
1. 执行 ```user_info``` 查看IPCountry是否为CN
2. 执行 ```download_sources``` 直接查看当前下载服务器是否为国内CDN

常见国内服务器域名可以在 [CDN清单](https://github.com/v2fly/domain-list-community/blob/master/data/steam) 查到

### 相关配置

本方案由两部分组成

- DNS劫持
- Nginx转发

#### DNS劫持

下面以ROS为例,假设Nginx服务所在IP为内网192.168.1.100,需要劫持的域名见如下配置

```
/ip dns static
add address=192.168.1.100 regexp=".*\\.steamcontent\\.com"
add address=192.168.1.100 name=cdn-ws.content.steamchina.com
add address=192.168.1.100 name=cdn-qc.content.steamchina.com
add address=192.168.1.100 name=cdn.mileweb.cs.steampowered.com.8686c.com
add address=192.168.1.100 name=dl.steam.clngaa.com
add address=192.168.1.100 name=st.dl.eccdnx.com
add address=192.168.1.100 name=st.dl.bscstorage.net
add address=192.168.1.100 name=st.dl.pinyuncloud.com
add address=192.168.1.100 name=steampipe.steamcontent.tnkjmec.com
add address=192.168.1.100 name=edge.steam-dns.top.comcast.net
add address=192.168.1.100 name=steam.eca.qtlglb.com
add address=192.168.1.100 name=steam.naeu.qtlglb.com
add address=192.168.1.100 name=steam.ru.qtlglb.com
add address=192.168.1.100 name=steampipe-kr.akamaized.net
add address=192.168.1.100 name=steampipe-partner.akamaized.net
add address=192.168.1.100 name=steampipe.akamaized.net
add address=192.168.1.100 name=f3b7q2p3.ssl.hwcdn.net
add address=192.168.1.100 name=steam.cdn.on.net
add address=192.168.1.100 name=steam.cdn.orcon.net.nz
add address=192.168.1.100 name=steam.cdn.slingshot.co.nz
add address=192.168.1.100 name=steam.cdn.webra.ru
```

若使用的DNS服务不支持通配符模式(比如基于hosts文件劫持),则需要手动补齐 ```*.steamcontent.com``` 相关的全部域名,具体清单可以Google到当前最新,建议还是使用支持通配符的DNS服务

配置完毕后将PC的DNS指向ROS后即可,另外也可以通过dstnat将dst port 53的udp流量直接劫持给ROS

以ROS ip 192.168.1.1,PC网段192.168.1.0/24为例,解析劫持配置如下

```
/ip firewall nat
add action=dst-nat chain=dstnat comment="Hijacking for DNS" dst-port=53 protocol=udp src-address=192.168.1.0/24 to-addresses=192.168.1.1 to-ports=53
```

#### Nginx转发配置

可选的CDN很多,这里以阿里云 ```xz.pphimalayanrt.com``` 为例,若使用其他CDN修改配置中对应域名即可


```
server {
    listen 80;
    server_name _;
    return 301 http://xz.pphimalayanrt.com$request_uri;
}
```

若阿里云速度不理想可以在 [CDN清单](https://github.com/v2fly/domain-list-community/blob/master/data/steam) 自行测试选出最佳CDN

### 检查是否生效

随便下载个游戏,检查Nginx日志,会发现有大量的 ```Valve/Steam HTTP Client 1.0``` 访问记录,即说明当前运行正常

### 其他方案

Dogfight360的 [Steam下载CDN重定向](https://www.dogfight360.com/blog/1531/) 非常好用的锁定工具 ~~除了每次要手动启动~~
