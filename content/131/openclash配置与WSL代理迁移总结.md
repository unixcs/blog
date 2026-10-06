---
title: openclash配置与WSL代理迁移总结
publish: true
source: 000_Raw/20_对话笔记/2026-08-14/openclash配置与WSL代理迁移总结.md
origin: 对话笔记提纯
tags: [网络基建]
---

> 对话笔记直发：openclash配置与WSL代理迁移总结。分类：网络基建。

---
tags: [OpenClash, WSL, SSH, 路由器, 代理迁移, 免密登录, dropbear]
---



> 整理日期：2026-08-14
> 设备：兆能 ZN M2（OpenWrt，ipq60xx，aarch64_cortex-a53），局域网 IP ``局域网IP（已脱敏）``
> 本机环境：Windows + WSL2（Ubuntu-24.04），WSL 路径 `/mnt/vps`，本机即 `\\wsl.localhost\Ubuntu-24.04\mnt\vps`

---

## 一、最终目标

把代理从「Windows 本机 V2Ray（端口 10808）」整体迁移到「路由器 OpenClash 透明代理」，实现：

- Windows 电脑、WSL、Docker 默认都走路由器，关闭 Windows V2Ray 也不影响上网。
- 保留一条手动回退方案：哪天手动关掉路由器 OpenClash，可在 Windows 重新开 V2Ray 接管。
- 两个 vless 节点自动切换 + 手动切换都可用。
- 指定网站走直连（DIRECT）。

---

## 二、路由器 OpenClash 配置

### 1. 基本信息
- OpenClash 版本：`0.45.129-beta`，内核 mihomo / clash_meta
- 订阅源配置：`/etc/openclash/config/1元.yaml`
- 运行时配置：`/etc/openclash/1元.yaml`（由 OpenClash 从订阅源生成）
- 本地主配置（推送源，唯一真相）：`/mnt/vps/op/vless_config.yaml`
- 启用开关：`uci set openclash.config.enable=1`（注意是 `enable` 不是 `enabled`）

### 2. 两个 vless 节点
| 名称                        | 地址                   | 类型               |
| ------------------------- | -------------------- | ---------------- |
| `Reality-aws`             | ``你的服务器IP（已脱敏）`:1443`   | vless + reality  |
| `vless9443-CF-Tunnel-443` | `douy.indevs.in:443` | vless + ws + tls |
|                           |                      |                  |

### 3. 代理组（核心）
```yaml
proxy-groups:
  - name: "AUTO"          # 自动切换组
    type: fallback
    url: https://www.baidu.com
    interval: 120
    proxies:
      - "Reality-aws"
      - "vless9443-CF-Tunnel-443"
      - DIRECT            # 两个都挂 → 直连
  - name: "PROXY"         # 手动切换组（给用户用）
    type: select
    proxies:
      - "AUTO"
      - "Reality-aws"
      - "vless9443-CF-Tunnel-443"
      - DIRECT
```
- **自动切换逻辑**：`AUTO` 是 fallback 类型，每 120 秒测速，谁通走谁；两个都挂自动降级到 `DIRECT`。
- **手动切换逻辑**：用户切 `PROXY` 组，可选 `AUTO`（自动）/ 指定节点 / 直连。

### 4. 规则（关键片段）
- `GEOIP,CN,DIRECT`：国内 IP 直连。
- 来自 `z.md` 的 15 个域名 → `DOMAIN-SUFFIX,...,DIRECT`（需直连的网站）。
- `MATCH,PROXY`：其余走代理。

### 5. DNS
- 模式：`redir-host`（OpenClash 把我们写的 fake-ip 改写了）
- `nameserver`：`223.5.5.5` / `119.29.29.29`
- `fallback`：`8.8.8.8` / `1.1.1.1` / 阿里 DNS

### 6. 外部控制面板（yacd）
- 面板地址：`http://`局域网IP（已脱敏）`:9090/ui/yacd/`
- `external-controller`：`0.0.0.0:9090`
- `secret`：`123456`（**弱口令，建议改强**）

---

## 三、IPv6 冲突修复

**问题**：OpenWrt 装 Clash 常见 IPv6 冲突——WSL/设备拿到公网 IPv6 后，流量绕过路由器代理直接出去，导致部分网站不走代理或网络异常。

**排查**：检查 `br-lan` 是否还带公网 IPv6 地址。

**修复**：
- 关闭 `wan6` 接口。
- 关闭 LAN 口的 `ra`（路由通告）和 `dhcpv6`。
- 验证：`br-lan` 不再持有公网 IPv6 地址，IPv6 流量也回到路由器透明代理管控下。

---

## 四、yacd 面板访问问题（排错记录）

| 现象 | 根因 | 解决 |
|------|------|------|
| 面板打开报 `Oops, something went wrong!` | 用户误删了 yacd 里保存的后端，面板默认连 `localhost:9090`（连的是用户电脑，没有 clash） | 重新在 yacd 右下角【切换后端】填入 `http://`局域网IP（已脱敏）`:9090` + 密钥 `123456` |
| 填短链 `shturl.cc/...` 返回 `Not Found` | 把短链当 API 地址了；404 说明服务器在，但路径不是 clash API | 正确地址是 `http://`局域网IP（已脱敏）`:9090` |
| 打开 `http://`局域网IP（已脱敏）`:9090/` 报 `{"message":"Unauthorized"}` | 浏览器没带 Bearer Token，clash 正常拒绝（401 类） | 这是正常现象，让 yacd 带密钥访问即可，API 本身健康 |
| 切到 `AUTO` 报 `Selector update error: proxy not exist` | 原 `PROXY` 是 fallback 类型，成员里没有 `AUTO` | 重构为：`AUTO` 独立 fallback 组，`PROXY` 改为 select 且首成员为 `AUTO` |

**验证**：手动切到 `Reality-aws` 后，出口 IP 变为 ``你的服务器IP（已脱敏）``，生效。

---

## 五、旧订阅残留清理

清理了迁移前的历史残留（避免干扰）：
- 路由器上：`.bak`、`.bak2`、`vless.yaml` 等旧配置。
- 本机上：`sub_1yuan*`、`clean_sub.py` 等清理脚本与备份。

---

## 六、WSL / Windows 代理迁移（核心改造）

### 1. 问题
Windows 上的 V2Ray 监听 `127.0.0.1:10808`，WSL 通过 `localhost:10808` 转发走代理。一旦在 Windows 关掉 V2Ray，WSL 立刻没网。我们要让默认走路由器，V2Ray 只作为手动备用。

### 2. 改造动作
- **`/etc/systemd/system/pi-web.service`**：删掉写死的
  `HTTP_PROXY/HTTPS_PROXY=http://127.0.0.1:10808`，只保留
  `NO_PROXY=192.168.*,172.16.*,10.*,127.*,localhost`。
- **`/etc/systemd/system/docker.service.d/proxy.conf`**：同样删掉 10808 的 `HTTP_PROXY/HTTPS_PROXY`，只保留 `NO_PROXY`。
- **新建 `/root/proxy.sh`（手动回退脚本）**：
  ```bash
  #!/usr/bin/env bash
  V2RAY="http://127.0.0.1:10808"
  NOPROXY="192.168.*,172.16.*,10.*,127.*,localhost,<local>"
  case "$1" in
    v2ray) export http_proxy="$V2RAY" https_proxy="$V2RAY" HTTP_PROXY="$V2RAY" HTTPS_PROXY="$V2RAY"
           export no_proxy="$NOPROXY" NO_PROXY="$NOPROXY"; echo "[proxy] -> Windows V2Ray ($V2RAY)";;
    off|router) unset http_proxy https_proxy HTTP_PROXY HTTPS_PROXY no_proxy NO_PROXY
                echo "[proxy] -> 关闭(走路由器 OpenClash 透明代理)";;
    status) echo "http_proxy=$http_proxy"; echo "https_proxy=$https_proxy";;
    *) echo "用法: source ~/proxy.sh v2ray | off | status";;
  esac
  ```
- **Windows 桌面一键切换脚本**（已建在 `C:\Users\fengx\Desktop\`）：
  - `switch_auto.bat` / `switch_reality.bat` / `switch_vless9443.bat`
  - 原理：用 curl `PUT http://`局域网IP（已脱敏）`:9090/proxies/PROXY`，带 `Authorization: Bearer 123456`，一键切节点。

### 3. 默认行为
- 新开的终端 / 服务：无代理变量 → 直连 → 由路由器 OpenClash 透明代理接管。
- 手动回退：在终端 `source ~/proxy.sh v2ray` 切回 Windows V2Ray；`source ~/proxy.sh off` 切回路由器。

---

## 七、今日重点排错：死代理 10808 卡住运行中的进程

### 现象
改完 systemd 配置、重启 daemon 后，**Open Code CLI（opencode）还是没网**，codex 也报同样问题。

### 根因（关键认知）
> 改的是「配置文件」，但**已经在跑的进程**手里那份环境变量清不掉。进程不死，死代理就一直跟着它。

Windows V2Ray 关掉后，`127.0.0.1:10808` 成了死代理，所有继承了它的运行进程全部断网。而且这些 CLI 是**用户早先在 10808 还活着时独立开的终端**，不归 `pi-web.service` 管——所以单纯重启 `pi-web` 救不了它们。

### 排查发现
| 进程 | 角色 | 状态 |
|------|------|------|
| `9364` | `pi-web.service`（旧） | 攥着 10808（配置已改，但进程没重启） |
| `51276` | 旧 `codebuddy` CLI | 攥着 10808（独立 WSL 终端） |
| `194370` | `opencode --yolo` | 攥着 10808（**就是用户说的"Open Code CLI"**） |
| `106956` / `51253` | 上述 CLI 的父 bash 终端 | 攥着 10808 |
| `10517` | `dockerd` | 攥着 10808（只改了配置没重启，`docker pull` 会挂） |
| `280849` | `codex` | **环境已干净**，无 10808（23:08 新起，落在干净 shell） |

### 处置动作
1. `systemctl restart pi-web.service` → 新进程（281183）环境干净。
2. `kill` 掉卡死的 `opencode`（194370）及其终端 `106956`、孤儿 `51253`（先 SIGTERM 后 SIGKILL）。
3. `systemctl restart docker.service` → 新 dockerd（281702）丢掉 10808。
4. 全局核查：确认**全系统已无任何进程持有 `127.0.0.1:10808`**。

### codex 专项说明
- codex 配置指向局域网网关 `http://`局域网IP（已脱敏）`:8317/v1`（本地 "muyuan" 服务），不是直连外网。
- 当前 codex 进程环境零代理变量，网关在线可达，干净环境直连 google=200。
- 结论：codex 已被这次系统级清理顺带修好；若仍看到旧窗口没网，关掉重开即可。

### 验收
- 新干净 shell：`google=200`，出口 IP ``你的服务器IP（已脱敏）``（走 `Reality-aws` 节点），证明路由器透明代理生效。

---

## 八、给用户的最终操作清单

**日常（默认）**
- Windows 关掉 V2

---
**相关**：[[8.16-路由器OpenClash代理与hindsight单层化]] [[5.31-手动操作流程总结]] [[8.15-OpenClash节点延迟诊断]] [[DeepSeek-API问题与Nginx超时调整说明]] [[8.9 腾讯云装Codex-Claude CLI]] [[8.14 搜索OpenWrt兆能M2固件项目总结]] [[对话笔记全量梳理表]]
