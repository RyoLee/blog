---
author: Ryo
title: 通过DNS与Nginx锁定并加速Steam下载(CDN)
date: 2023-01-28T17:08:11+0800
lastmod: 2024-09-24T04:21:33+0800
description: "通过DNS与Nginx锁定并加速Steam下载(CDN)"
tags: ["Steam", "CDN", "Nginx", "DNS"]
projects: ["Homelab"]
---
### 更新记录

- 2024-09-24: 简化配置, 通过lancache机制直接实现对CDN的控制

### 为什么要手动锁定

当使用透明网关对内网进行加速时(包括部分游戏加速器的特定工作模式), 会因为Steam根据当前ip判断自动切换到ip对应地区的下载CDN, 造成隧道流量的浪费甚至产生反效果导致下载缓慢

### 如何判断是否需要使用该方案

最简单的办法就是不做任何其他配置的情况下(若使用加速器需要先打开加速器), 打开 ```Steam--设置--下载``` 检查下载地区, 若显示为任意大陆地区城市则大概率不需要本方案修改(这个选项城市其实是假的, 只要是任意大陆城市本质都一样)

若显示为透明网关对应隧道/加速器ip地区, 则可以考虑使用本方案

另外也可以通过 ```Win+R``` 执行 ```steam://open/console``` 调出Steam控制台来查看信息判断
1. 执行 ```user_info``` 查看IPCountry是否为CN
2. 执行 ```download_sources``` 直接查看当前下载服务器是否为国内CDN

常见国内服务器域名可以在 [CDN清单](https://github.com/v2fly/domain-list-community/blob/master/data/steam) 查到

### 相关配置

本方案由两部分组成

- DNS
- Nginx

#### DNS

下面以ROS为例, 假设Nginx服务所在IP为内网192.168.1.100, 需要劫持的域名见如下配置

```
/ip dns static
add address=192.168.1.100 name=lancache.steamcontent.com
```

配置完毕后将PC的DNS指向ROS后即可, 另外也可以通过dstnat将dst port 53的udp流量直接劫持给ROS

以ROS ip 192.168.1.1, PC网段192.168.1.0/24为例, 解析劫持配置如下

```
/ip firewall nat
add action=dst-nat chain=dstnat comment="Hijacking for DNS" dst-port=53 protocol=udp src-address=192.168.1.0/24 to-addresses=192.168.1.1 to-ports=53
```

也可以直接在Steam运行机器通过Hosts文件配置

#### Nginx转发配置

可选的CDN很多, 这里以阿里云 ```xz.pphimalayanrt.com``` 为例, 若使用其他CDN修改配置中对应 ```server``` 与 ```Host``` 即可

```
upstream steam-upstream {
    server xz.pphimalayanrt.com;
}
server {
    listen 80;
    server_name lancache.steamcontent.com *.steamcontent.com;
    slice 8m;
    proxy_set_header Range $slice_range;
    location / {
        proxy_pass http://steam-upstream;
        proxy_set_header Host "xz.pphimalayanrt.com";
    }
}
```

若阿里云速度不理想可以参考 [dogfight360 的这篇 blog](https://www.dogfight360.com/blog/knowledge-base/%E5%A6%82%E4%BD%95%E6%8F%90%E9%AB%98steam%E7%9A%84%E4%B8%8B%E8%BD%BD%E9%80%9F%E5%BA%A6%E4%B8%AD%E5%9B%BD%E5%A4%A7%E9%99%86%E5%9C%B0%E5%8C%BA/) 自行逐个测试, 根据自身网络选出最佳CDN

### 检查是否生效

随便下载个游戏, 检查Nginx日志, 会发现有大量的 ```Valve/Steam HTTP Client 1.0``` 访问记录, 即说明当前运行正常

### 其他方案

Dogfight360的 [Steam下载CDN重定向](https://www.dogfight360.com/blog/1531/) 非常好用的锁定工具 ~~除了每次要手动启动~~
