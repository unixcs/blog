---
title: AI-chat-项目操作规范说明书
publish: true
source: 000_Raw/20_对话笔记/2026-08-05/AI-chat-项目操作规范说明书.md
origin: 对话笔记提纯
tags: [网络基建]
---

> 对话笔记直发：AI-chat-项目操作规范说明书。分类：网络基建。

# AI-chat 项目操作规范说明书

## 1. 文档目的

本文档用于统一 AI-chat 项目的开发、部署、升级、备份与运维操作规范，确保以下目标长期成立：

1. 线上更新代码时不影响真实业务数据。
2. 两台服务器可以按统一原则执行升级，即使当前数据库位置不同。
3. 后续可以将操作过程沉淀为稳定可复用的 Skill 或脚本。
4. 运维人员在新窗口或新会话中也能按文档独立完成操作。

## 2. 项目范围

本文档适用于以下对象：

1. GitHub 仓库：`https://github.com/unixcs/AI-chat`
2. 服务器：`121.41.192.80`
3. 服务器：`121.41.206.32`
4. Docker 部署场景
5. DeepSeek API 配置与提示词文件管理

## 2.1 服务器连接信息

本项目当前涉及两台生产服务器，默认使用 `admin` 用户通过 SSH 登录。

1. `121.41.192.80`
   - SSH：`ssh admin@121.41.192.80`
   - Hostname：`iZbp15kdyy928z0o7i8p1nZ`
   - 项目目录：`/opt/AI-chat`
2. `121.41.206.32`
   - SSH：`ssh admin@121.41.206.32`
   - Hostname：`iZbp1d00qv2i2wyo5nj6hbZ`
   - 项目目录：`/opt/AI-chat`

连接说明：

1. 默认使用当前操作机已配置的 SSH key 登录
2. 如登录时提示输入密码，说明当前环境未预配密钥，需要先确认登录凭据
3. 执行任何升级命令前，必须先确认自己已登录到目标服务器，避免误操作另一台机器

首次登录后，建议先执行以下命令确认服务器身份与项目目录：

```bash
hostname
pwd
ls -la /opt
ls -la /opt/AI-chat
```

如需快速确认 AI-chat 相关容器状态，建议执行：

```bash
docker ps --format '{{.Names}}\t{{.Image}}\t{{.Status}}'
```

## 3. 当前已确认的项目事实

### 3.1 代码侧事实

当前 GitHub 最新代码已具备以下能力：

1. 后端优先读取环境变量 `SQLITE_PATH`。
2. 未设置 `SQLITE_PATH` 时，自动回退到 `backend/data.sqlite`。
3. Docker 部署默认使用容器内固定路径：`/app/data/data.sqlite`。
4. Docker Compose 默认挂载宿主机数据目录到容器内 `/app/data`。
5. Docker 构建已通过 `.dockerignore` 排除 `*.sqlite`、`*.sqlite-wal`、`*.sqlite-shm`。
6. 提示词文件通过 `DEEPSEEK_SYSTEM_PROMPT_FILE` 指定，默认值为 `prompts/Prompt.md`。
7. 最新模板默认模型已更新为：`DEEPSEEK_MODEL=deepseek-v4-flash`。
8. 最新模板默认上游地址已统一为：`DEEPSEEK_BASE_URL=https://api.deepseek.com`。

### 3.2 服务器侧事实

两台服务器当前都已统一到同一套 Docker 生产基线：

1. 代码目录统一为：`/opt/AI-chat`
2. 宿主机数据库目录统一为：`/srv/ai-chat/data`
3. 容器内数据库路径统一为：`/app/data/data.sqlite`
4. 两台服务器当前都显式使用：`SQLITE_PATH=/app/data/data.sqlite`
5. 两台服务器后端当前都通过目录挂载将 `/srv/ai-chat/data` 挂载到容器内 `/app/data`
6. 两台服务器前端对外端口当前统一为：`8181`
7. 两台服务器容器命名当前统一为：`ai-chat-backend`、`ai-chat-frontend`
8. 两台服务器当前统一使用以下后端关键环境变量：

```env
NODE_ENV=production
SQLITE_PATH=/app/data/data.sqlite
ALLOW_JSON_SEED=
DEEPSEEK_API_KEY=<server-local-secret>
DEEPSEEK_MODEL=deepseek-v4-flash
DEEPSEEK_BASE_URL=https://api.deepseek.com
DEEPSEEK_SYSTEM_PROMPT_FILE=prompts/Prompt.md
DEEPSEEK_TIMEOUT_MS=90000
MODEL_CONCURRENCY=6
MODEL_QUEUE_MAX=50
MODEL_RETRY_MAX=2
MODEL_RETRY_BASE_MS=500
MODEL_KEY_COOLDOWN_MS=60000
```

两台服务器当前的代码维护方式也已收敛为同一原则：

1. GitHub 仓库仍是标准代码来源
2. 线上默认优先采用方案 B，即手动上传最新源码包到 `/tmp/ai-chat-update` 后做受控同步
3. 只有在现场再次确认 Git 状态、网络条件和本地部署差异都允许时，才考虑切换到方案 A

### 3.3 当前统一部署基线

当前线上 AI-chat 的统一部署基线如下：

1. 项目目录：`/opt/AI-chat`
2. 宿主机数据库目录：`/srv/ai-chat/data`
3. SQLite 数据文件：`data.sqlite`、`data.sqlite-wal`、`data.sqlite-shm`
4. 容器内数据库路径：`/app/data/data.sqlite`
5. 前端对外端口：`8181`
6. 后端宿主机端口：`3001`
7. 提示词文件：`backend/prompts/Prompt.md`
8. 本地敏感配置：`backend/.env`
9. 默认更新口径：保留 `backend/.env` 和真实数据库文件，受控替换应用代码、Dockerfile、Nginx 配置与提示词文件
10. 当前生产模型：`deepseek-v4-flash`
11. 当前生产上游地址：`https://api.deepseek.com`
12. 当前生产 `docker-compose.yml` 需要继续保留 `8181:80`，不得直接套用 GitHub 默认的 `80:80`

### 3.4 最近一次双机统一结果

最近一次双机升级已完成以下动作：

1. `121.41.206.32` 已从 `/opt/AI-chat-data/db` 迁移到 `/srv/ai-chat/data`
2. `121.41.192.80` 已从 `/opt/AI-chat/backend/data.sqlite*` 迁移到 `/srv/ai-chat/data`
3. 两台服务器当前都已切换为目录挂载：`/srv/ai-chat/data:/app/data`
4. 两台服务器当前都已验证健康接口正常
5. 两台服务器当前都已验证核心表可读，且能正常处理真实请求流量

注意：
每次实际升级前，必须再次检查线上真实状态，不能只依赖历史记录。

## 4. 总体运维原则

### 4.1 第一原则

线上更新时，只更新应用代码和镜像，不覆盖真实数据库内容。

### 4.2 第二原则

GitHub 仓库是代码来源，不是线上数据库内容来源。

### 4.3 第三原则

任何升级动作前，必须先识别当前生效数据库路径，再决定备份和更新方案。

### 4.4 第四原则

服务器本地配置默认优先级高于 GitHub 默认文件，尤其是以下内容：

1. `backend/.env`
2. 当前真实数据库文件
3. 与部署强相关的服务器本地配置
4. 已经人工维护过的提示词文件

### 4.5 第五原则

两台服务器禁止同时升级，必须先完成一台服务器的升级与验证，再执行另一台。

### 4.6 第六原则

当前默认升级方式仍是方案 B，即手动上传最新源码包后做受控同步；只有在现场确认满足条件时，才允许切换到方案 A。

## 5. 角色与责任

### 5.1 开发负责人

负责：

1. 本地修改项目代码
2. 推送 GitHub 最新版本
3. 明确本次是否涉及提示词、环境变量、模型参数调整

### 5.2 运维执行人

负责：

1. 检查线上实际运行状态
2. 备份数据库
3. 更新代码与镜像
4. 验证容器、健康接口和数据库完整性

### 5.3 文档维护人

负责：

1. 将每次确认后的流程固化进 Skill
2. 维护本说明书与 Skill 一致
3. 在服务器布局发生变化后及时更新文档

## 6. 环境规范

### 6.1 开发环境规范

开发环境默认行为如下：

1. 不配置 `SQLITE_PATH`
2. 自动使用 `backend/data.sqlite`
3. 如需本地空库初始化，可按需使用 `ALLOW_JSON_SEED=true`

开发环境目标：

1. 零配置快速运行
2. 不依赖生产绝对路径
3. 不影响生产数据库策略

### 6.2 非 Docker 生产环境规范

非 Docker 生产环境必须显式设置数据库路径，例如：

```env
SQLITE_PATH=/var/lib/ai-chat/data.sqlite
```

要求：

1. 生产环境不允许依赖默认回退路径
2. 路径必须是完整文件路径，不允许只写目录
3. 数据目录必须独立于代码目录

### 6.3 Docker 生产环境规范

Docker 部署统一采用以下逻辑：

1. 容器内固定数据库路径：`/app/data/data.sqlite`
2. 宿主机挂载真实数据目录到 `/app/data`
3. `backend/.env` 或 compose 中显式指定：

```env
SQLITE_PATH=/app/data/data.sqlite
```

当前双机线上还应同时满足：

1. `NODE_ENV=production`
2. `ALLOW_JSON_SEED=`
3. `DEEPSEEK_MODEL=deepseek-v4-flash`
4. `DEEPSEEK_BASE_URL=https://api.deepseek.com`
5. 前端对外端口继续统一使用 `8181`

### 6.4 推荐长期目标

两台生产服务器当前已经统一到隔离数据目录布局，即：

1. 代码目录仅放应用代码
2. 数据目录仅放 SQLite 数据文件
3. Skill 以后只负责更新代码、镜像和验证

## 7. 关键文件约定

### 7.1 代码目录

默认代码目录：

```text
/opt/AI-chat
```

### 7.2 数据目录

推荐隔离数据目录：

```text
/srv/ai-chat/data
```

或非 Docker 生产目录：

```text
/var/lib/ai-chat
```

### 7.3 数据文件

SQLite 相关文件至少包含：

1. `data.sqlite`
2. `data.sqlite-wal`
3. `data.sqlite-shm`

### 7.4 配置文件

关键配置文件包括：

1. `backend/.env`
2. `docker-compose.yml`
3. `backend/Dockerfile`
4. `frontend/Dockerfile`
5. `frontend/nginx.default.conf`
6. `backend/prompts/Prompt.md`

## 8. DeepSeek 与提示词管理规范

### 8.1 关键环境变量

后端相关变量包括：

```env
DEEPSEEK_API_KEY=
DEEPSEEK_MODEL=deepseek-v4-flash
DEEPSEEK_BASE_URL=https://api.deepseek.com
DEEPSEEK_SYSTEM_PROMPT_FILE=prompts/Prompt.md
DEEPSEEK_TIMEOUT_MS=90000
MODEL_CONCURRENCY=6
MODEL_QUEUE_MAX=50
MODEL_RETRY_MAX=2
MODEL_RETRY_BASE_MS=500
MODEL_KEY_COOLDOWN_MS=60000
```

### 8.2 DeepSeek API 管理规则

1. `DEEPSEEK_API_KEY` 属于服务器本地敏感配置，不入库。
2. 更新 GitHub 代码时，默认不覆盖 `backend/.env`。
3. 如需切换模型、代理地址或 Key，必须先备份 `.env`。

### 8.3 提示词管理规则

1. `backend/prompts/Prompt.md` 可能属于业务配置，而不是纯代码文件。
2. 如果线上提示词经过人工定制，不应盲目用 GitHub 版本覆盖。
3. 如需长期稳定，建议后续将提示词迁移为服务器外挂文件，并通过 `DEEPSEEK_SYSTEM_PROMPT_FILE` 指向独立路径。

### 8.4 是否替换提示词的判断标准

满足以下条件时，才建议直接替换线上提示词：

1. 本次业务明确要求切换到 GitHub 最新提示词
2. 已确认线上当前业务目标与 GitHub 提示词一致
3. 已备份旧提示词文件

## 9. 升级前检查规范

每次操作前必须完成以下检查。

### 9.1 容器状态检查

检查：

```bash
cd /opt/AI-chat
docker compose ps
```

### 9.2 健康接口检查

检查：

```

---
**相关**：[[7.11美的空调midea_ac_lan插件迁移记录]] [[7.13 HA巴法云插件兼容修复]] [[7.23 白嫖云盘cftc,tgState,tgNetDisc]] [[对话笔记全量梳理表]]
