---
author: Ryo
title: RouterOS 端口扫描防御
description: "RouterOS 入侵检测"
tags: ["RouterOS", "Mikrotik", "ROS", "Firewall", "防火墙", "Bark"]
projects: ["Homelab"]
---

通过ROS防火墙检测并屏蔽端口扫描行为,并使用bark将屏蔽事件发送至手机

### ROS配置
#### 屏蔽列表IP

屏蔽BOT列表中的数据包,建议将这一条的优先级提高到最高(即编号#1)

```routeros
/ip firewall raw
add action=drop chain=prerouting comment=ANTI-BOT src-address-list=BOT
```
#### 指定端口方式

对于常见家用环境下不使用的高危TCP端口进行记录,在连续尝试3次(1分钟超时)后拉黑IP 7天

```routeros
/ip firewall raw
add action=add-src-to-address-list address-list=BOT address-list-timeout=1w chain=prerouting comment=ANTI-SCAN dst-port=20-23,53,80,135,139,443,445,1433,2049,3389,5900,8080,8089 log=yes log-prefix=ANTI-SCAN protocol=tcp src-address-list=SCAN_S2
add action=add-src-to-address-list address-list=SCAN_S2 address-list-timeout=1m chain=prerouting comment=ANTI-SCAN dst-port=20-23,53,80,135,139,443,445,1433,2049,3389,5900,8080,8089 protocol=tcp src-address-list=SCAN_S1
add action=add-src-to-address-list address-list=SCAN_S1 address-list-timeout=1m chain=prerouting comment=ANTI-SCAN dst-port=20-23,53,80,135,139,443,445,1433,2049,3389,5900,8080,8089 in-interface-list=WAN protocol=tcp
```
#### PSD方式

如果觉得搞一堆端口配置太麻烦,也可以用内置的PSD来实现,默认规则为: 3秒内低位端口计3分,高位端口计1分,合计21分触发

实测默认规则针对端口批量扫描检出率较高,针对定向端口弱口令爆破识别效果稍差,建议自行调整参数,或者结合指定端口方式配置

```routeros
/ip firewall raw
add action=add-src-to-address-list address-list=BOT address-list-timeout=1w chain=prerouting comment=ANTI-SCAN in-interface-list=WAN log=yes log-prefix=ANTI-SCAN protocol=tcp psd=21,3s,3,1
add action=add-src-to-address-list address-list=BOT address-list-timeout=1w chain=prerouting comment=ANTI-SCAN in-interface-list=WAN log=yes log-prefix=ANTI-SCAN protocol=udp psd=21,3s,3,1
```

#### Rsyslog配置

如果不需要推送屏蔽完成上述配置即可

假设接收日志机器ip为192.168.1.100

```routeros
/system logging action
set remote=192.168.1.100 remote
/system logging
add action=remote topics=info
```

如果不需要其他消息可以在topics中加上firewall,仅过滤并转发防火墙的log

### Linux配置

假设接收设备ip为192.168.1.100,ROS ip为192.168.1.1

修改/etc/rsyslog.conf如下部分,去除最前面的"#"解除注释,启用udp模式rsyslog接收ros发送过来的日志

```
module(load="imudp")
input(type="imudp" port="514")                              # 可按需修改端口,需要同步修改ros配置端口
```

再添加如下规则

```
:msg, contains, "ANTI-SCAN"     ^/opt/bark/router.sh        # 调用脚本处理并发送bark消息
:FROMHOST-IP, isequal,"192.168.1.1" /var/log/router.log     # 汇总ros日志
:FROMHOST-IP, isequal,"192.168.1.1" ~                       # 防止重复收集到message中
```
修改/etc/logrotate.d/rsyslog 在花括号前添加一行,配置日志老化

```
/var/log/router.log
```


创建脚本 /opt/bark/router.sh 并添加权限(chmod +x /opt/bark/router.sh),消息格式可自行按需调整

YOUR_BARK_SERVER 替换为BARK服务器ip/域名

YOUR_DEVICE_ID 替换为接收设备ID

```bash
#!/usr/bin/bash
msg=$(echo "$*" |tr " " "\n"|grep '\->'|grep -E "(25[0-5]|2[0-4][0-9]|[01]?[0-9][0-9]?)\.(25[0-5]|2[0-4][0-9]|[01]?[0-9][0-9]?)\.(25[0-5]|2[0-4][0-9]|[01]?[0-9][0-9]?)\.(25[0-5]|2[0-4][0-9]|[01]?[0-9][0-9]?)"|tr -d ",")
curl -X "POST" "https://YOUR_BARK_SERVER/push" \
     -H 'Content-Type: application/json; charset=utf-8' \
     -d $'{
  "body": "已封禁:\n'$msg'",
  "device_key": "YOUR_DEVICE_ID",
  "level": "passive",
  "title": "IDS"
}'
```

重启rsyslog与logrotate服务使配置生效

```bash
systemctl restart rsyslog
systemctl restart logrotate
```