---
author: Ryo
title: RouterOS 通过 OSPF IP分流至 OpenWrt
date: 2022-04-20T17:59:11+0800
lastmod: 2023-03-09T15:31:37+0800
description: "ROS ip 分流方案"
tags: ["RouterOS","ROS", "OpenWrt","OP","OSPF","BRID","ip分流"]
projects: ["Homelab"]
---

### 更新
- 2022-05-17 修改后的方案(带配置) [RouterOS 通过 OSPF IP分流非CN流量](https://blog.9-ch.com/post/homelab-no-cn-routes/)

### 组网参考

Reflist:
- [使用RouterOS，OSPF 和OpenWRT给国内外 IP 分流](https://www.truenasscale.com/2021/12/13/195.html)
- [RouterOS路由OSPF协议+树莓派分流国外流量](https://www.kn1f4.com/news/767.html)

主要适用于不想折腾PS/XBOX/Switch设备网关配置,又不希望出口网关为卵漏油/AIB的,或者怕折腾网络时家里断网影响家人的

遇到一些问题记录一下

### ROS DDNS失效

表现: 域名解析ip不正确

原因: 分流后ROS内置ip cloud(DDNS)流量同样会被指向OpenWrt导致错误,尝试打标签bypass对应udp流量未成功,因为有公网IP所以直接改为通过域名服务商API更新对应解析为PPPOE接口地址

解决: 修改以下脚本中"CloudFlare variables"段内容即可.原始脚本见[Mikrotik_CF_DDNS](https://github.com/mike6715b/Mikrotik_CF_DDNS)

<details>
  <summary>Cloudflare DDNS ROS脚本(IPV4)</summary>
  
```
#########################################################################
#         ==================================================            #
#         $ Mikrotik RouterOS update script for CloudFlare $            #
#         ==================================================            #
#                                                                       #
# - You need a CloudFlare account & api key (look under settings),      #
#   a zone and A record in it                                           #
# - All variables in first section are obvious,                         #
#   except CFid and CFzoneid                                            #
# - Obtain CFzoneid from Cloudflare Dashboard,                          #
#   on Overview tab scroll down                                         # 
#   To obtain CFid use following command in any unix shell:             #
#    curl -X GET "https://api.cloudflare.com/client/v4/zones/YOUR_ZONE_ID/dns_records?name=YOUR_DOMAIN" -H "Authorization:Bearer $CFtkn" -H "Content-Type: application/json" | python -mjson.tool
# - You can use my Postman script to get those variables                #
# - Enable CFDebug if needed - it'll print some info to logs            #
# - Enable CFcloud if you don't get a public IP on interface            #
# - Put script under /system scripts giving "read,write,ftp" policy access.       #
#   For 6.29 and older "test" policy is also needed.                    #
# - Add script to /system scheduler using it's name in "on-event"       #
# - Requires at least RouterOS 6.44beta75 for multiple header support   #
#                                                                       #
#              Credits for Samuel Tegenfeldt, CC BY-SA 3.0              #
#                        Modified by kiler129                           #
#                        Modified by viritt                             #
#                        Modified by asuna                              #
#                        Modified by mike6715b                          #
#                                                                       #
#               Tested and working as of February 22, 2021              #
#########################################################################

################# CloudFlare variables #################
:local CFDebug "false"
:local CFcloud "false"

:global WANInterface "[YOUR_WAN_INTERFACE_NAME]"

:local CFdomain "[YOUR_DOMAIN]"

:local CFtkn "[YOUR_CF_TOKEN]"

:local CFzoneid "[YOUR_CF_ZONE_ID]"
:local CFid "[YOUR_CF_ID]"

:local CFrecordType ""
:set CFrecordType "A"

:local CFrecordTTL ""
:set CFrecordTTL "60"

#########################################################################
########################  DO NOT EDIT BELOW  ############################
#########################################################################

:log info "Updating $CFDomain ..."

################# Internal variables #################
:local previousIP ""
:global WANip ""

################# Build CF API Url (v4) #################
:local CFurl "https://api.cloudflare.com/client/v4/zones/"
:set CFurl ($CFurl . "$CFzoneid/dns_records/$CFid");
 
################# Get or set previous IP-variables #################
:if ($CFcloud = "true") do={
    :set WANip [/ip cloud get public-address]
};

:if ($CFcloud = "false") do={
    :local currentIP [/ip address get [/ip address find interface=$WANInterface ] address];
    :set WANip [:pick $currentIP 0 [:find $currentIP "/"]];
};

:if ([/file find name=ddns.tmp.txt] = "") do={
    :log error "No previous ip address file found, createing..."
    :set previousIP $WANip;
    :execute script=":put $WANip" file="ddns.tmp";
    :log info ("CF: Updating CF, setting $CFDomain = $WANip")
    /tool fetch http-method=put mode=https output=none url="$CFurl" http-header-field="Authorization:Bearer $CFtkn,content-type:application/json" http-data="{\"type\":\"$CFrecordType\",\"name\":\"$CFdomain\",\"ttl\":$CFrecordTTL,\"content\":\"$WANip\"}"
    :error message="No previous ip address file found."
} else={
    :if ( [/file get [/file find name=ddns.tmp.txt] size] > 0 ) do={ 
    :global content [/file get [/file find name="ddns.tmp.txt"] contents] ;
    :global contentLen [ :len $content ] ;  
    :global lineEnd 0;
    :global line "";
    :global lastEnd 0;   
            :set lineEnd [:find $content "\n" $lastEnd ] ;
            :set line [:pick $content $lastEnd $lineEnd] ;
            :set lastEnd ( $lineEnd + 1 ) ;   
            :if ( [:pick $line 0 1] != "#" ) do={   
                #:local previousIP [:pick $line 0 $lineEnd ]
                :set previousIP [:pick $line 0 $lineEnd ];
                :set previousIP [:pick $previousIP 0 [:find $previousIP "\r"]];
            }
    }
}

######## Write debug info to log #################
:if ($CFDebug = "true") do={
 :log info ("CF: hostname = $CFdomain")
 :log info ("CF: previousIP = $previousIP")
 :log info ("CF: currentIP = $currentIP")
 :log info ("CF: WANip = $WANip")
 :log info ("CF: CFurl = $CFurl&content=$WANip")
 :log info ("CF: Command = \"/tool fetch http-method=put mode=https url=\"$CFurl\" http-header-field="Authorization:Bearer $CFtkn,content-type:application/json" output=none http-data=\"{\"type\":\"$CFrecordType\",\"name\":\"$CFdomain\",\"ttl\":$CFrecordTTL,\"content\":\"$WANip\"}\"")
};
  
######## Compare and update CF if necessary #####
:if ($previousIP != $WANip) do={
 :log info ("CF: Updating CF, setting $CFDomain = $WANip")
 /tool fetch http-method=put mode=https url="$CFurl" http-header-field="Authorization:Bearer $CFtkn,content-type:application/json" output=none http-data="{\"type\":\"$CFrecordType\",\"name\":\"$CFdomain\",\"ttl\":$CFrecordTTL,\"content\":\"$WANip\"}"
 /ip dns cache flush
    :if ( [/file get [/file find name=ddns.tmp.txt] size] > 0 ) do={
        /file remove ddns.tmp.txt
        :execute script=":put $WANip" file="ddns.tmp"
    }
} else={
 :log info "CF: No Update Needed!"
}
```
</details>

### TCP流量握手缓慢

表现: 访问网站时打开缓慢,F12查看SSL阶段时间长度接近甚至超过10秒

原因: OP与ROS在同一网段,进出流量路由不对称,触发ROS防火墙动作

解决: 禁用ROS防火墙drop invalid规则(位于ip-firewall-filter rules)

### ICMP(Ping)异常

表现: ping 被分流网段ip时黑洞

原因: 出口Op通过TProxy只能处理TCP/UDP流量

解决:
- A: Op添加 ICMP劫持(假响应,可规避一些软件的报错)
- B: ROS给ICMP流量打上bypass的route标签,再创建一个bypass路由表,该表不包含OSPF宣告的分流规则,所以如果没有被ISP之类的劫持就是裸连的真响应

### Openwrt无法与被分流网段通信(循环)

表现: OpenWrt与被分流网段通信时不通

原因: ROS接到宣告的表后循环Op->ROS->Op->...

解决: 给Op流量打上标签,直接走bypass表

防火墙规则(Mangle)可参考

```
chain=prerouting action=mark-routing new-routing-mark=bypass passthrough=yes src-address=[BYPASS_DEVICE_IP] dst-address-list=!private log=no log-prefix=""
```

### DNS CDN优化

表现: 不需要分流区域域名CDN解析为分流区域

原因: Op全局流量隧道,DNS解析也被隧道处理,导致CDN失效

解决: dnsmasq-full分流list外走国内DNS服务,list内走DoH,Op故障时回落全国内DNS服务

### OpenWrt访问内网异常

表现: Op所在子网广播域外的其他子网访问Op时,无法正常响应

原因: Op走了bypass表,未添加其他子网路由,下一跳直接从pppoe-out出去了

解决: 防火墙匹配src address为op的ip,dst address为需要支持访问子网流量打上main标签走main路由表
