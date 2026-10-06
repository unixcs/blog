---
title: M.d 8.3 Hermes记忆迁移Hindsight与Docker清理
publish: true
source: 000_Raw/20_对话笔记/2026-08-03/M.d 8.3 Hermes记忆迁移Hindsight与Docker清理.md
origin: 对话笔记提纯
tags: [AI工程]
---

> 对话笔记直发：M.d 8.3 Hermes记忆迁移Hindsight与Docker清理。分类：AI工程。

---
date: 2026-08-03
tags: [对话笔记]
---

# M.d 8.3 Hermes记忆迁移Hindsight与Docker清理

## 目标与背景
- jp 服务器（`局域网IP（已脱敏）`，Linux 6.1 rockchip aarch64）Hermes 记忆体系从 Honcho 切换到 Hindsight，并解决磁盘紧张问题
- 磁盘 29G 仅剩 3.7G（88%），Docker 镜像/缓存占大头
- 用户要求：删除 glm-free-api + kimi-free-api；用子代理 review 计划后执行；Hindsight 最终定案部署在 jp 本地（云服务器 yun1 仅 1.6G 内存/40G 盘，跑不动）
- 前置：Nous auth 修复已完成（root 生成 auth.json 属主不对导致 gateway 读不到，chown hermes:hermes 修复）

## 关键结论
- **根因教训**：gateway 侧所有配置文件必须属主 `hermes:hermes`（600 权限），root 写的文件会导致 hermes 用户读不到（Nous auth.json 曾因此报 "not logged into Nous Portal"）
- **Hindsight 部署方案**：local_external 模式，Docker 容器跑在 jp 本地，LLM 走 metapi(:4000) 的 deepseek-v4-flash（长寿命 key，替代被删的 glm/kimi）
- **honcho 移除关键坑**：honcho 是 systemd 服务（`/etc/systemd/system/honcho.service`，Restart=always、User=root），**裸 kill 会 5 秒复活**，必须 `systemctl stop honcho.service`；honcho-pg 容器 `restart=unless-stopped` 需 `docker rm` 而非 stop
- **磁盘根因**：/var/lib/docker 21G + 缓存垃圾；真正的存储大户（HA/n8n/FNS/vaultwarden/网关）都绑定 jp 局域网不能迁，能上云的只有冷备份
- **云服务器定位**：yun1(`你的服务器IP（已脱敏）`) 1.6G 内存/40G 盘，跑 Hindsight 峰值 1.5-2G 必 OOM，改做冷备份库
- **Hindsight 容器网络坑**：容器访问 huggingface.co 需代理（HTTP_PROXY=172.17.0.1:7890，宿主 sing-box）；NO_PROXY 不能用 CIDR（``局域网IP（已脱敏）`/24` 不被 httpx 识别），必须写显式 IP ``局域网IP（已脱敏）``，否则 LAN 的 metapi 请求误走代理导致 502
- **metapi 502 排查**：初判上游慢，实测是代理配置问题；修复后 host 与容器内调用全部 200
- **hindsight-client 必须手动装**：`uv pip install hindsight-client==0.6.1`（local_external 的 is_available() 无条件返回 True，不装会在首次 retain/recall 时 ImportError）

## 执行记录
- Nous 修复：`chown hermes:hermes /opt/data/auth.json /opt/data/shared/nous_auth.json && chmod 600` → Telegram 实测 api_calls=1 成功
- 阶段 A 磁盘清理（回收 ~1.6G，3.7G→5.2G）：
  - `rm -rf /root/.npm /root/.nvm`（619M，root 的 node 开发缓存，无进程依赖）
  - `rm -rf /home/openclaw/.npm /home/openclaw/.cache`（467M，保留 v22.22.1，openclaw-gateway 仍 active）
  - `/root/backups`(519M) rsync 到 yun1 `/home/admin/backups`（6385 文件校验一致）后删本地；jp 生成 `~/.ssh/id_ed25519` 并加入 yun1 admin 的 authorized_keys
  - komari metrics.db(817M) VACUUM 跳过（freelist 仅 18 页，856M 是真实数据，收益 ~72KB）
- 阶段 B Hindsight 部署：
  - `systemctl stop honcho.service` → `docker rm honcho-pg` → `rm -rf /opt/data/honcho`(796M) + `rm honcho.json`
  - `docker rmi pgvector/pgvector:pg15 redis:7-alpine`（redis:7 孤儿，sub2api 用的是 redis:8）
  - `docker stop/rm/rmi glm-free-api kimi-free-api`（先确认无消费者：honcho 是 llm-gateway:19000 唯一客户端，已先撤）
  - `/opt/data/hindsight/config.json`：`{"mode":"local_external","api_url":"http://127.0.0.1:8888","llm_base_url":"http://`局域网IP（已脱敏）`:4000/v1"}`（属主 hermes:hermes 600）
  - 容器启动：`docker run -d --name hindsight -p 8888:8888 -p 9999:9999 -v hindsight-data:/home/hindsight/.pg0 -e HTTP_PROXY=http://172.17.0.1:7890 -e HTTPS_PROXY=http://172.17.0.1:7890 -e NO_PROXY=localhost,127.0.0.1,`局域网IP（已脱敏）` -e HINDSIGHT_API_LLM_PROVIDER=openai -e HINDSIGHT_API_LLM_API_KEY=<metapi id=1 key> -e HINDSIGHT_API_LLM_BASE_URL=http://`局域网IP（已脱敏）`:4000/v1 -e HINDSIGHT_API_LLM_MODEL=deepseek-v4-flash ghcr.io/vectorize-io/hindsight:latest`
  - config.yaml:398 `provider: honcho` → `hindsight`，`systemctl restart hermes-gateway`
- 验证：`hermes memory status` 显示 hindsight installed/available；插件工具 hindsight_retain/hindsight_recall/hindsight_reflect 加载；端到端 sync_turn→recall 返回 "部署代号是JP-MIGRATION-TEST-12"

## 状态与待办
- ✅ Hindsight healthy（db connected），17 容器运行，磁盘 6.9G 剩余（76%），hermes-gateway/openclaw active
- ⚠️ metapi 上游首个请求慢（7-9s），曾触发 retain_extract_facts 502 重试，后续留意 hindsight 日志
- ⏭ llm-gateway(:19000) 现已无消费者（honcho 已撤），是下一步可清理项，本次未动
- ⏭ 后续思路 A：Hindsight 管会话记忆 + llm-wiki(FNS)/Obsidian 管知识沉淀，FNS vault feng 2111 篇笔记


---
**相关**：[[8.16-路由器OpenClash代理与hindsight单层化]] [[5.8-JP-NAS-本地部署-Kimi-GLM-统一网关]] [[7.25-在-yun1-部署-new-api-并通过-Cloudflare-域名-HTTPS-访问-—-实操记录]] [[8.2 jp CPA workbuddy qoderwork 插件故障排查]] [[8.3 AI工具链收口与自媒体内容生产—123阶段执行规划]] [[8.3 AI工具链收口与自媒体内容生产—执行规划]] [[对话笔记全量梳理表]]

> 注：本文涉及的服务器公网 IP / Tailnet / 局域网 IP 已脱敏，请替换为你自己的地址。
