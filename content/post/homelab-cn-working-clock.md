---
author: Ryo
title: iOS中国大陆工作日闹钟
description: "iOS中国大陆工作日闹钟"
tags: ["iOS","快捷指令"]
projects: ["homelab"]
---

使用数据来自[cn_working_day](https://github.com/RyoLee/cn_working_day)项目

基于[workalendar](https://github.com/workalendar/workalendar) 生成近30天工作日信息

使用Github action自动更新

iOS工作日闹钟[快捷指令-工作日闹钟](https://www.icloud.com/shortcuts/20c84b95071d4ac489f59dc8307ba279)

使用该快捷指令需配置2个闹钟,分别对应工作日和休息日,再编辑快捷指令找到如下部分
- [打开]闹钟"工作日"
- 将变量[工作日]设为[闹钟]
- [打开]闹钟"休息日"
- 将变量[休息日]设为[闹钟]

因闹钟可能不存在或未配置闹钟名,[打开]闹钟可能会显示异常,请手动修改打开闹钟"工作日"/"休息日"为对应闹钟

如果休息日想睡懒觉不需要闹钟,则删除以下三项
- [打开]闹钟"休息日"
- 将变量[休息日]设为[闹钟]
- 否则(询问是否同时删除"否则"内的操作时,选择"删除操作")

自动化可以自行选择方案,比较常见的是在 快捷指令-自动化-创建个人自动化-打开"勿扰模式"模式时 添加运行本快捷指令利用勿扰模式触发自动运行

勿扰模式在 设置-专注模式-勿扰模式-自动打开 处添加 每天 00:10-06:00 配置,即可实现全自动触发