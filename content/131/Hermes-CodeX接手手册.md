---
title: Hermes-CodeX接手手册
publish: true
source: 000_Raw/20_对话笔记/2026-08-10/Hermes-CodeX接手手册.md
origin: 对话笔记提纯
tags: [AI工程]
---

> 对话笔记直发：Hermes-CodeX接手手册。分类：AI工程。

# 腾讯云 CodeX 远程执行 · Hermes 上手与对齐手册

> 用途:无论**你本人**还是**未来的 Hermes 会话**接手,读完这篇就能在 5 分钟内对齐全貌、知道怎么用、知道什么不能碰。

## 一、一句话说清这是啥

你在 **Telegram 发一句话** → 家里 NAS 上的 **Hermes** → 腾讯云服务器 **tx** 上的 **CodeX** 自动写代码/跑命令 → 结果回给你。

```
Telegram ──⟩ Hermes(jp: 192.168.2.223) ──⟩ SSH强制命令 ──⟩ codex-run(tx: 124.222.144.181)
                                                                      │
                                                                   Docker容器(隔离笼子)
                                                                      │
                                                                   OpenCode Zen 模型
                                                                      │
                                                              写文件到 /workspace/work
```

## 二、核心事实速查表(接手必读)

| 项目 | 值 | 备注 |
|---|---|---|
| tx(腾讯云) | `124.222.144.181`, Ubuntu 22.04 | 跑着你的商城 **yoshop**,**绝不能动** |
| jp(家里 NAS) | `192.168.2.223` | Hermes 进程在这 |
| 入口脚本 | `tx:/usr/local/sbin/codex-run` | 唯一入口,SSH 强制命令调用 |
| CodeX 版本 | **锁定 0.94.0** | 见第四节,别升级 |
| 模型 | OpenCode Zen,`/v1/chat/completions` | 免费小模型 deepseek-v4-flash-free |
| 密钥 | `tx:/etc/codex-runner/zen.key`(root:root 0600) | **本地别留明文副本** |
| Hermes 白名单用户 | `TELEGRAM_ALLOWED_USERS=5359999791` | 只有你一个,私聊 |
| 容器资源上限 | 内存 512M / 硬盘 4G / CPU 0.8 核 / 无特权 / 独立网络 | 碰不到商城 DB |

## 三、Hermes 对齐三步(放完技能文件后必做)

新技能文件已部署到 jp,但**光放文件不生效**——Hermes 有进程内技能缓存,不看文件时间,所以老进程发现不了新技能。

1. **Telegram 发 `/restart`** —— 等它回"已重启"。*别用 SSH 硬重启,会打断在跑的对话。*
2. **发 `/new`** —— 开新会话,老会话复用旧提示,没有新技能。
3. **发 `/tx-codex-remote 写个 bash 脚本算 1 到 100 的和,跑一下给我看结果`** —— 确认链路通。

> 现在重启安全:5 个定时任务最早明天 03:00,不撞车、不补跑。

## 四、为什么是现在这副样子(关键技术决策)

1. **CodeX 锁 0.94.0**:zen 的 `/v1/responses` 是半成品壳(事件发不全),0.146 版本必报 `OutputTextDelta without active item`。二分 9 个版本定位到 **0.94.0 是最后一个支持老 `wire_api="chat"` 的**,0.95 砍了。
2. **必须隔离进容器**:CodeX 是"不问直接执行命令"模式。关进 Docker 独立网络后,MySQL/Redis 只绑 127.0.0.1,**6 条路径实测全不通**。
3. **SSH 强制命令白名单**:`restrict,command="sudo -n /usr/local/sbin/codex-run $SSH_ORIGINAL_COMMAND"`,参数走允许列表,`$SSH_ORIGINAL_COMMAND` 不引号展开是安全的(分号/换行都变字面词被拒)。
4. **5 个安全补丁**(已审查通过):
   - P1 密钥不进 `cmdline`(改成 `-e ZEN_API_KEY` 继承,已实测商城 PHP 用户 `ps aux` 看不到)
   - P2 防软链接逃逸(容器内 `mkdir` + `readlink -f` 校验,模拟攻击验证过)
   - P3 超时清理孤儿进程
   - P4 `flock` 并发锁(一次只跑一个任务)
   - P5 status 挂载检查

## 五、日常使用

**方式 A:说人话(推荐)**
> 帮我在腾讯云上写个脚本,统计某日志里出现最多的 10 个 IP

**方式 B:点名(更保险,不依赖系统提示)**
```
/tx-codex-remote 你的需求
```

**注意(免费小模型,别为难它):**

| 注意                   | 原因                      |
| -------------------- | ----------------------- |
| 一次一件事,别给一长串          | 小模型扛不住复杂多步              |
| 别写 **Python / Node** | 容器里只有 bash/git/curl     |
| 别从 **GitHub** 拉代码    | 国内 GFW 连不通(npm/pypi 可以) |
| **一次一个任务**           | 并发锁会拒第二个                |
| 每个任务用不同目录名           | 不然产物互覆盖                 |
| 碰不到商城数据库             | 故意设计,非 bug              |
| 命令统一用 `bash -lc` 开头  | 防止重定向符号被当参数             |

## 六、安全边界(红线)

- 不动 tx 上的 yoshop(Nginx / PHP8.3-FPM / MySQL8 / Redis / systemd timer)
- 不升级 CodeX(会破坏 chat 协议兼容)
- 不改 SSH 白名单 / UFW / 挂载 docker.sock
- 不在本地留明文密钥文件
- 所有写操作都发生在容器 `/workspace/work` 内,撑爆也只影响自己

## 七、文件清单

| 位置 | 是什么 |
|---|---|
| `tx:/usr/local/sbin/codex-run` | 服务器唯一入口 |
| `tx:/etc/codex-runner/zen.key` | 密钥(root 0600) |
| `tx:/var/log/codex-run.log` | 每次调用日志 |
| `jp:/opt/data/skills/devops/tx-codex-remote/SKILL.md` | Hermes 实际读的技能文件 |
| 本地 `/mnt/vps/tencent/codex-run.sh` | 入口脚本源码(同线上) |
| 本地 `/mnt/vps/tencent/tx-codex-remote-SKILL.md` | 技能文件源码(同 jp) |
| 本地 `/mnt/vps/tencent/hermes-codex-plan.md` | 完整方案+18 项验收记录 |

## 八、排查与救助

Telegram 里:
```
/tx-codex-remote 帮我看下 status
```
或 SSH:`ssh root@124.222.144.181 '/usr/local/sbin/codex-run status'`

| 现象 | 处理 |
|---|---|
| `已有任务在执行中` | 正常,等前一个跑完 |
| `执行超时` | 拆小任务,或加 `--timeout` |
| 硬盘满 | 让它执行 `reset` 清空 |
| 只回字不动手 | 提示加"用 bash -lc"重发 |
| 容器挂了 | **找我,别自己 SSH 乱重启** |


---
**相关**：[[8.13-Hermes-网关免费模型自动切换与报错修复]] [[4.30-Cloudflare-Tunnel-0基础手动安装教程]] [[腾讯云环境快速上手]] [[8.9 公众号自动发布工作流流程图]] [[8.9 公众号自动发布工作流部署]] [[8.10 vikunja-任务-计划]] [[对话笔记全量梳理表]]
