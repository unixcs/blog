---
title: AI 多服务器统一运维与自动部署 Plan V1
publish: true
source: 000_Raw/20_对话笔记/2026-08-31/AI 多服务器统一运维与自动部署 Plan V1.md
origin: 对话笔记提纯
tags: [小程序接单]
---

> 对话笔记直发：AI 多服务器统一运维与自动部署 Plan V1。分类：小程序接单。

可以。这一轮我们实际上把 **“MindFS 统一入口 → Tailscale 私网 → 多服务器 AI 运维 → 权限分层 → 后续自动部署”** 这条链路基本定下来了。

下面我把这条对话整理成一份可以直接交给 Codex / Trellis 执行的 **Plan V1**。我会把已经确定的和暂时不做的东西分开，避免后面执行时跑偏。

# 《AI 多服务器统一运维与自动部署 Plan V1》

## 1. 总目标

建立一套以 **腾讯云 4G VPS + MindFS** 为核心的个人 AI 基础设施。

最终效果：

```text
手机 / Windows
       │
       ▼
   MindFS Web
       │
       ▼
腾讯云 4G VPS
AI 主控 / Codex / OpenCode / 其他 CLI
       │
       │ Tailscale 私网
       ▼
┌──────────┬──────────┬──────────┬──────────┐
│ NAS      │ 国内 VPS │ 国内 VPS │ 国外 VPS │
│          │          │          │          │
│ 重要节点 │ 测试节点 │ 测试节点 │ 业务节点 │
└──────────┴──────────┴──────────┴──────────┘
```

核心原则：

> **MindFS 是统一入口，腾讯云 VPS 是 AI 主控，Tailscale 是私网网络，SSH 是服务器运维通道。**

以后不管使用 Codex CLI、OpenCode、其他 CLI，原则上都从 MindFS 进入。

---

# 2. 当前已经确定的基础架构

## 2.1 MindFS

已经在腾讯云 VPS 上部署并测试成功。

MindFS 作为：

* 手机端统一入口
* Windows 端统一入口
* CLI/Web 会话入口
* 多个 AI CLI 的统一工作入口

暂时不再增加 Telegram Bot。

原因：

* MindFS 本身已经有任务完成提醒
* 手机浏览器可以直接访问
* 后续可以封装成 PWA
* Telegram 单纯作为通知入口价值不大

因此：

> **MindFS Web/PWA = 主入口**

Telegram 暂不进入 V1。

---

# 3. Tailscale 网络

## 3.1 所有设备加入同一个 Tailnet

计划纳入：

1. 腾讯云 4G VPS（主控）
2. NAS
3. Windows
4. 两台国内 VPS
5. 两台国外 VPS
6. 后续新增服务器

最终形成一个私有 AI 基础设施网络。

---

## 3.2 为什么使用 Tailscale

目的不是替代 SSH，而是：

> **让 SSH、HTTP、Docker Web 服务等全部跑在私有网络里面。**

例如：

```text
公网：
Windows → 公网IP:2288 → SSH

未来：
Windows
   ↓
Tailscale
   ↓
100.x.x.x:22
   ↓
SSH
```

SSH 仍然是 TCP。

Tailscale/WireGuard 主要使用 UDP 建立加密隧道。

如果无法直接连接，可以通过 DERP 中继。

---

# 4. 自建 DERP

当前已经验证过：

* 手机使用移动网络
* 访问腾讯云 VPS 上的 MindFS
* 原来延迟可能达到 500ms+
* 自建/近端 DERP 后
* 延迟降低到几十～100ms 左右

因此：

> **保留自建 DERP 的思路。**

之前开放的高位 UDP 端口属于 Tailscale/DERP 网络通信，不等于开放 SSH。

两者必须区分：

```text
UDP 高位端口
    ↓
Tailscale / DERP / WireGuard

TCP 22/2288 等
    ↓
SSH
```

---

# 5. SSH 网络策略

最终目标：

> **公网 SSH 关闭，SSH 只通过 Tailscale 私网访问。**

但不要直接一次性关闭。

正确执行顺序：

### 阶段 1

目前：

* SSH 使用高位端口，例如 2288
* 禁止密码登录
* 允许 SSH Key
* 当前保留 Root + Key

### 阶段 2

部署并验证 Tailscale：

```text
Windows
手机
腾讯云 VPS
NAS
其他 VPS
```

全部可以稳定互通。

### 阶段 3

确认 Tailscale 出问题时还有：

* 云厂商控制台
* Web Console
* Rescue / VNC / Serial Console 等救援手段

然后：

> **关闭公网 SSH 端口。**

以后：

```text
公网 → SSH ❌

Tailscale → SSH ✅
```

---

# 6. SSH Key 策略

这是本次讨论确定的重要原则。

不要复用 Windows 当前个人 SSH Key。

建立：

> **AI 专用 SSH Key**

例如：

```text
Windows 个人 Key
        ↓
你本人使用

AI 运维 Key
        ↓
腾讯云 MindFS 使用
```

这样：

* 人和 AI 身份分开
* AI Key 泄露可以单独撤销
* 不影响 Windows
* 后续可以单独轮换
* 更容易控制服务器权限

---

# 7. 服务器权限模型

目前计划有四类节点。

## A. AI 主控服务器

腾讯云 4G VPS：

```text
MindFS
Codex CLI
OpenCode
其他 AI CLI
```

这是整个系统的控制中心。

---

# 8. 非重要服务器

目前两台测试服务器。

用途：

* 测试
* Demo
* AI Coding
* Docker 测试
* 部署实验
* 自动化实验
* 仿真环境

这类服务器：

> **允许 MindFS 使用 Root。**

原因：

这些机器可以重装、恢复。

如果 AI：

* 安装软件
* 修改系统
* Docker
* 重启
* 删除测试文件
* 部署
* 调试

不希望因为权限不足频繁卡住。

所以：

```text
MindFS
  ↓
Tailscale
  ↓
SSH
  ↓
root
```

方便 AI 自动化。

---

# 9. 重要服务器

例如：

* NAS
* 正式业务服务器
* 正式数据服务器
* 数据库
* 重要 Docker 服务

原则：

> **不直接给 MindFS Root。**

先创建普通用户。

例如：

```text
ai-worker
```

然后：

```text
MindFS
   ↓
Tailscale
   ↓
SSH
   ↓
ai-worker
```

第一版暂时：

> **普通用户先不做非常复杂的权限限制。**

先正常使用。

实际遇到：

```text
权限不足
```

再针对具体需求增加权限。

这样避免一开始设计一大堆复杂 sudo 规则。

---

# 10. 普通用户未来的权限演进

V1：

```text
普通用户
↓
正常 SSH
↓
正常文件操作
↓
正常项目运行
```

以后根据实际需求逐步增加：

```text
sudo 白名单
```

例如：

```text
允许：
docker logs
docker restart xxx

不允许：
sudo -i
sudo su
修改防火墙
修改用户
磁盘操作
```

但这些暂时不提前做复杂化。

原则：

> **不是先设计一万条权限规则，而是 AI 真正遇到权限问题，再解决具体权限问题。**

---

# 11. NAS 特殊处理

NAS：

* ARM64
* RK3566
* CPU 较弱
* 8GB RAM
* 当前约使用一半 RAM
* 剩余存储 <10GB
* 已经运行 Hermes
* Hermes 执行任务时 CPU 有时达到 90～99%

因此：

> **不建议在 NAS 上再常驻一套 MindFS + AI CLI。**

NAS 更适合：

```text
存储
Docker
服务
文件
API
```

AI 计算仍然放：

```text
腾讯云 VPS
```

腾讯云 VPS：

```text
MindFS
Codex
OpenCode
AI 工作
```

然后：

```text
MindFS
 ↓
SSH
 ↓
NAS ai-worker
 ↓
执行 NAS 本地命令
```

SSH 本身几乎不产生明显 CPU 压力。

真正消耗资源的是 SSH 后面执行的任务。

---

# 12. NAS 权限

NAS 建立：

```text
ai-worker
```

不直接给 Root。

暂时也不急着做复杂 sudo。

以后如果出现：

> “AI 因为权限不足无法重启某个 Docker”

再增加：

```text
sudo 白名单
```

例如只允许：

```text
restart xxx
status xxx
logs xxx
```

而不是：

```text
sudo ALL=(ALL) ALL
```

---

# 13. AI 与个人权限分离

最终形成：

### 你本人

```text
Windows
 ↓
Tailscale
 ↓
SSH
 ↓
你的个人 Key
 ↓
必要时 Root
```

### AI

```text
MindFS
 ↓
Tailscale
 ↓
SSH
 ↓
AI 专用 Key
 ↓
不同服务器对应不同权限
```

两套身份完全分开。

---

# 14. 五台/多台服务器的管理模式

最终不要让 AI 到处猜 IP。

建立一个简单的：

```text
server-inventory
```

记录：

```yaml
servers:
  controller:
    role: ai-controller

  nas:
    role: storage

  test-01:
    role: test

  test-02:
    role: test

  production-01:
    role: production

  production-02:
    role: production
```

以后进一步记录：

* Tailscale 地址
* SSH 用户
* 服务
* Docker
* 用途
* 权限等级

AI 先看 Inventory，再决定：

> “我要去哪个服务器？”

而不是自己猜 IP。

---

# 15. 未来自动化部署

这是下一阶段重点。

你以后写了一个小项目：

```text
AI Coding
     ↓
测试服务器
     ↓
运行测试
     ↓
成功
     ↓
部署
```

部署目标根据项目类型决定。

---

## 15.1 本地/私网项目

如果只是自己使用：

```text
部署服务器
 ↓
Docker
 ↓
Tailscale
 ↓
手机 / Windows
```

不需要公网域名。

---

## 15.2 需要公网访问

可以：

```text
项目
 ↓
Cloudflare
 ↓
域名
 ↓
公网服务
```

---

# 16. Cloudflare Workers

后续可以给 AI 一定程度的 Cloudflare 权限。

但原则：

> **不要直接给 Cloudflare 全局 Root/API Token。**

应该根据实际用途创建：

* 指定 Zone
* 指定项目
* 指定 Workers
* 指定 DNS
* 指定必要权限

让 AI 只能完成：

```text
部署
更新
查看
```

而不是：

```text
整个 Cloudflare 账号随便操作
```

---

# 17. 自动部署判断逻辑

未来可以让 AI 自动判断：

### 静态网站

```text
Cloudflare Pages
```

### Serverless/API

```text
Cloudflare Workers
```

### Docker 服务

```text
VPS
```

### 大数据/数据库/持久化服务

```text
VPS / NAS
```

不要强行把所有项目都塞进 Workers。

---

# 18. 当前明确暂时不做

为了防止项目越来越重，V1 暂时不做：

### ❌ Telegram Bot

因为：

* MindFS 已经有通知
* 手机浏览器已经能用
* 后续 PWA 足够
* Telegram 暂时没有明显收益

### ❌ 第二套 NAS MindFS

因为 NAS CPU 太弱。

### ❌ ELK / Loki 等日志系统

目前没有必要。

### ❌ 复杂权限中心

先普通用户跑起来。

### ❌ 复杂资产管理系统

先简单 Inventory。

### ❌ OpenMemory

目前先不引入。

跨 CLI 的长期记忆继续采用：

```text
项目目录
+
Markdown
+
AGENTS.md
+
Git
```

以后真正有需求再引入专门 Memory 系统。

---

# 19. CLI 跨工具记忆

统一采用：

```text
project/
├── AGENTS.md
├── memory/
│   ├── CURRENT_STATE.md
│   ├── DECISIONS.md
│   ├── NEXT.md
│   └── CHANGELOG.md
└── ...
```

任务结束时：

```text
OpenCode
Codex

---
**相关**：[[4.27-Bitwarden-+-Vaultwarden+-Tailscale]] [[5.4-VPS-安全加固与-SSH-Fail2ban-配置]] [[7.23-白嫖云盘cftc,tgState,tgNetDisc]] [[2026年08月09日]] [[8.9 首页导航路由改造（apex-dashboard版）]] [[8.11 vikunja-任务分层-v4]] [[对话笔记全量梳理表]]
