---
author: Ryo
title: Emby 弹幕插件
date: 2022-08-08T18:44:25+0800
lastmod: 2022-08-09T10:26:11+0800
description: "Emby danmaku extension"
tags: ["Emby","弹幕", "插件","Tampermonkey","油猴","弹弹play"]
projects: ["Homelab"]
---
![截图](https://s1.ax1x.com/2022/08/09/vltmKs.png)

### 项目主页
[RyoLee/emby-danmaku](https://github.com/RyoLee/emby-danmaku)

### 安装
任意一种方式安装即可

#### 浏览器方式(推荐)
1. [Tampermonkey](https://www.tampermonkey.net/)
2. [添加脚本](https://cdn.jsdelivr.net/gh/RyoLee/emby-danmaku@gh-pages/ede.user.js)

#### 服务端方式
路径 /system/dashboard-ui/index.html
在```</body>```前添加如下标签
```
<script src="https://cdn.jsdelivr.net/gh/RyoLee/emby-danmaku@gh-pages/ede.user.js" defer></script>
```

#### 客户端方式
类似服务端方式,解包后修改dashboard-ui/index.html再重新打包即可,iOS需要通过类似AltStore方式自签,请自行Google解决

### 界面
插件运行后会在界面左下侧追加4个新按钮,分别为:
- 弹幕开关
- 手动匹配
- 简繁转换
- 弹幕信息(触发右下角消息)

### 补充说明
弹幕来源为[弹弹play](https://www.dandanplay.com/) 已开启弹幕聚合(A/B/C 站等弹幕融合)

因播放文件在emby中与上游弹弹play的名称可能存在不同(如"彻夜之歌"<-->"夜曲"),或因为存在多季/剧场版/OVA导致搜索匹配错误,首次播放时请检查当前弹幕信息是否正确匹配,若匹配错误可尝试手动匹配

匹配完成后对应关系会保存在浏览器本地存储中,后续播放(包括同季的其他集)会优先按照保存的匹配记录载入弹幕

### TODO
- [ ] 屏蔽规则导入
- [x] 简繁转换开关
- [ ] 弹幕加载失败自动重试

### 参考项目

[susundingkai/emby-danmaku](https://github.com/susundingkai/emby-danmaku)

主要差异:
- 配置存储改为 localstorage
- 弹幕数据跨域问题通过 cloudflare 的 workers 处理绕过
- 通过播放对象的 item id 从 emby 的 api 获取数据,不再使用页面对象提取,更新导致出错概率大幅下降(顺便修好了播放下一集不能加载的问题)

### License
[MIT License](https://github.com/RyoLee/emby-danmaku/blob/master/LICENSE)