---
title: 旧笔记本当服务器：通电自启动与真熄屏
date: 2026-10-05
tags: [Linux, 服务器, 笔记本, 省电, 折腾]
description: Dell 旧笔记本跑 Debian 当服务器：BIOS 通电自启第一性原理，以及 brightness=0 仍漏光的真熄屏根治。
---

# 旧笔记本当服务器：通电自启动与真熄屏

> 机型 Dell Inspiron 5557 / Debian 13 / 纯 SSH 无桌面。本文是实战记录，可直接复用。

## 一、通电自启动：第一性原理

链路只有一条：`交流电 → PSU → EC/PMIC → 启动序列`。关机不断电时 EC 还在跑，默认行为是等开机键，想上电即启就得改这个默认。

- **正解**：BIOS 里 `AC Power Recovery / After Power Loss` 设为 On（Dell 一般 F2 进 Power Management）。
- **死路**：消费级机型若无此项，只能外接继电器短接电源键，属于硬件魔改。
- **误区**：`rtcwake`、cron 定时对**完全关机**无效，只对睡眠/休眠有效，别在这上面浪费时间。

## 二、真熄屏：brightness=0 不等于断电

现象：`/sys/class/backlight/intel_backlight/brightness=0`，屏幕却还发着光。

排查结论：

1. `brightness=0` 只是 PWM 占空比为 0，老 i915 面板照样漏光；
2. `systemd-backlight` 有最低亮度钳位（日志：`Saved brightness 0 is too low; increasing to 46`），重启必亮；
3. 真熄屏必须断背光电源域：写 `bl_power=1` 或 `drm dpms off`。

根治命令（可逆，写 0 即恢复）：

```bash
echo 1 > /sys/class/backlight/intel_backlight/bl_power
cat /sys/class/backlight/intel_backlight/bl_power        # 期望 1
cat /sys/class/backlight/intel_backlight/actual_brightness  # 期望 0
```

持久化：写进开机启动脚本（systemd unit 或 rc.local），否则重启后钳位又把亮度提回来。合盖不休眠另需 `logind.conf` 三项 `ignore` + mask 睡眠 target，本文不展开。

## 三、效果

屏幕彻底黑、整机功耗降一档，远程 SSH 毫无影响。一台吃灰笔记本，就这样变成 7×24 的开发机。
