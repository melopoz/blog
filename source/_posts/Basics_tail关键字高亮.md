---
title: tail关键词高亮
tags: note
categories: basics
date: 2021/01/01 20:46:25
updated: 2021/01/01 20:46:25
---



```
tail -f system.log |grep "/write-down" |grep "payload=" | perl -pe 's/(visitType)|(visitTerminal)|(operateType)/\e[1;33m$1\e[0m\e[1;33m$2\e[0m\e[1;33m$3\e[0m/g'
```
`perl`命令 进行动态替换，格式：
```
perl -pe 's/([关键词1])|([关键词2])/\e
```
关键词部分：`s/([关键词1])|([关键词2])/`
颜色部分：`\e[1;颜色1$1\e[背景颜色` `\e[1;颜色2$2\e[背景颜色`


```
字体颜色：
30m：黑
31m：红
32m：绿
33m：黄
34m：蓝
35m：紫
36m：青
37m：白

背景颜色设置
40-47 黑、红、绿、黄、蓝、紫、青、白
40：黑
41：红
42：绿
43：黄
44：蓝
45：紫
46：青
47：白

```