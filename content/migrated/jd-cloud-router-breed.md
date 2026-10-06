---
title: 京东云无线宝（一代 128G）刷机：编程器刷 Breed 上老毛子/集客
date: 2023-06-05
tags: [软路由, 刷机, Breed, OpenWrt]
description: 京东云无线宝拆机用 CH341A 刷 Breed：备份、eeprom 提取改 MAC、老毛子/集客固件与 mesh 实测。
origin: 博客园迁移
---

# 京东云无线宝（一代 128G）刷机

> 原载博客园（2023-06-05）。为垃圾佬群开车整理的整合教程，仅供学习研究。图片已略，步骤保留。

## 准备

- 硬件：CH341A 土豪金编程器 + SOP16 转 DIP8
- 软件：AsProgrammer / NeoProgrammer
- 接线：夹子红线对 FLASH 右下角⑦脚，对应关系按 1-8→7/8/9/10/15/16/1/2 夹好

## 刷 Breed

标准九步：检测 → 读取 → 校验 → 保存（32M 备份！）→ 擦除 → 查空 → 选固件 → 写入 → 校验。

- 检测不到 flash：查驱动/夹子，或换 Win7 机器。
- 校验报错：换 USB 口，或上电 10 秒再自动跑。
- 假死无响应：划重点——**上电 10 秒**。

Breed 刷完撤夹子，长按 reset 上电 3 秒松开，蓝灯快闪后浏览器开 `192.168.1.1` 即 Breed（Recovery 概念，Breed 在就不怕变砖）。


![刷breed](assets/jd-cloud-router-breed/img10.png)

![刷breed](assets/jd-cloud-router-breed/img11.png)

![刷breed](assets/jd-cloud-router-breed/img12.png)

![刷breed](assets/jd-cloud-router-breed/img13.png)

![刷breed](assets/jd-cloud-router-breed/img14.png)

![刷breed](assets/jd-cloud-router-breed/img15.png)

![刷breed](assets/jd-cloud-router-breed/img16.png)

![Breed](assets/jd-cloud-router-breed/img17.png)

![Breed](assets/jd-cloud-router-breed/img18.png)

![Breed](assets/jd-cloud-router-breed/img19.png)

![Breed](assets/jd-cloud-router-breed/img20.png)

![Breed](assets/jd-cloud-router-breed/img21.png)

![Breed](assets/jd-cloud-router-breed/img22.png)

![Breed](assets/jd-cloud-router-breed/img23.png)

![Breed](assets/jd-cloud-router-breed/img24.png)

![Breed](assets/jd-cloud-router-breed/img25.png)

![Breed](assets/jd-cloud-router-breed/img26.png)
## eeprom 提取与改 MAC

Breed 里：开环境变量（内部）保存重启 → 开 breed 保存。用 WinHex 打开 32M 备份，取 `40000~4FFFF` 另存 `eeprom.bin`（64K），倒数第 2 行 `0000FFE0` 起按无线宝背面 MAC 改写保存，Breed 里分别上传固件 + eeprom。

## 固件实测

- **老毛子**：管理 IP `192.168.123.1`（admin/admin）。`mkfs.ext4 /dev/mmcblk0` 格 128G，开 Telnet/SSH，开 Entware。已知：有线最高 800M 后掉到 500M，5G 最高 300M，疑固件问题。
- **集客**：管理 IP `6.6.6.6`（admin）。遇 MAC 冲突用 mac 工具查 2.4/5G 地址回填 Breed。已知：内存显示 240、不支持 IPv6。
- **mesh**：OP 主路由 DHCP，下面两台集客 AP 同 SSID/密码、同信道、覆盖重合 30% 最佳。

![设备连接](assets/jd-cloud-router-breed/img1.png)

![设备连接](assets/jd-cloud-router-breed/img2.png)

![设备连接](assets/jd-cloud-router-breed/img3.png)

![设备连接](assets/jd-cloud-router-breed/img4.png)

![设备连接](assets/jd-cloud-router-breed/img5.png)

![设备连接](assets/jd-cloud-router-breed/img6.png)

![设备连接](assets/jd-cloud-router-breed/img7.png)

![设备连接](assets/jd-cloud-router-breed/img8.png)

![设备连接](assets/jd-cloud-router-breed/img9.png)

![集客](assets/jd-cloud-router-breed/img27.png)

![集客](assets/jd-cloud-router-breed/img28.png)

![集客](assets/jd-cloud-router-breed/img29.png)

![集客](assets/jd-cloud-router-breed/img30.png)

![集客](assets/jd-cloud-router-breed/img31.png)

![集客](assets/jd-cloud-router-breed/img32.png)

---
**相关**： [[00-索引-玩机NAS索引]] [[华为悦盒 EC6180V9 刷 ubuntu20.04 做 NAS]]
