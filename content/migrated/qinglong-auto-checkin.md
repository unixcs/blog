---
title: 青龙平台自动签到脚本踩坑记录
date: 2021-12-07
tags: [青龙, 签到, Docker, 羊毛]
description: 青龙平台部署 Sitoi 签到脚本：apk 依赖报错、cookie 转义、smzdm 脚本更换与防封定时策略。
origin: 博客园迁移
---

# 青龙平台自动签到脚本踩坑记录

> 原载博客园（2021-12-07，阅读 14466），工具版本已老，仅留思路与坑位参考。

## 起因

在大妈签到大半年，身为懒癌患者，手动签到太不得劲，遂上青龙 + Sitoi 开源脚本（dailycheckin）。

## 坑 1：安装依赖报错

```bash
apk add --no-cache gcc g++ python python-dev py-pip mysql-dev linux-headers libffi-dev openssl-dev
```

第一条就报错，换 apk 源无用。解法：**先删除青龙平台里的 linux 依赖**，再执行；`python-dev` 包名在新版 Alpine 里改为 `python3-dev`：

```bash
apk add --no-cache gcc g++ python3-dev py-pip mysql-dev linux-headers libffi-dev openssl-dev
```

## 坑 2：cookie 报错

提取的 cookie 直接粘贴报错，需要先做转义（可用 json.cn 等在线工具处理后再填环境变量）。

## 更新：smzdm 签到脚本更换（2022-06-26）

原脚本 smzdm 签到失效，换新脚本（my_script 仓库的 `smzdm_signin.js`）：

```bash
docker cp smzdm_signin.js <容器id>:/ql/scripts
```

青龙面板：环境变量加 `SMZDM_COOKIE`，定时任务加 `task smzdm_signin.js`。

## 防封：随机定时思路

Cron 不支持随机，折中办法：同一任务建多个，分别设不同时间点（如每周 1/3/5/7 跑 A 时间，2/4/6 跑 B 时间），错峰执行。
