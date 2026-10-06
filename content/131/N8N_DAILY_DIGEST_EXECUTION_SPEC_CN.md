---
title: N8N_DAILY_DIGEST_EXECUTION_SPEC_CN
publish: true
source: 000_Raw/20_对话笔记/2026-05-12/N8N_DAILY_DIGEST_EXECUTION_SPEC_CN.md
origin: 对话笔记提纯
tags: [网络基建]
---

> 对话笔记直发：N8N_DAILY_DIGEST_EXECUTION_SPEC_CN。分类：网络基建。

# N8N 每日资讯早报执行规范

## 1. 目标

基于自托管 `n8n` 构建一套每日自动资讯早报工作流，默认约束如下：

- 已完成部署：`n8n + SQLite`
- 推送渠道：`Telegram`
- LLM：`GPT-5.4`
- 允许的来源类型：
  - RSS
  - RSSHub
  - 少量 HTTP/API
  - 网页抓取仅作为兜底
- 优先级：
  - 低维护成本
  - 低 token 消耗
  - 稳定、可无人值守按日运行

目标结果：

- 每天早晨，Telegram 自动收到一份结构化科技/AR 早报
- 工作流可无人值守运行
- AI 仅在规则过滤之后介入

---

## 2. 范围

Phase 1 范围：

- 构建最小可用工作流
- 使用 `3 到 5` 个稳定来源
- 仅推送到 Telegram
- GPT-5.4 仅用于生成简短摘要
- 不做多渠道分发
- 不做复杂审批流
- 默认不做全文网页抓取

Phase 1 暂不包含：

- 企业微信 / 个人微信推送
- 大规模网页爬取
- 复杂数据库建模
- 向量数据库 / embeddings
- 全文文章入库
- 长篇 AI 改写

---

## 3. 架构

建议拆分为 3 个工作流：

1. `source-fetch-rss`
2. `daily-news-main`
3. `digest-publish-telegram`

用途如下：

### 3.1 `source-fetch-rss`

抓取多个 RSS / RSSHub / API 来源，并统一字段格式。

### 3.2 `daily-news-main`

每天定时运行，聚合所有条目，去重、过滤、排序，并选出最终进入早报的内容。

### 3.3 `digest-publish-telegram`

对最终入选内容生成简短摘要，并将格式化后的早报发送到 Telegram。

---

## 4. 实施顺序

建议按以下顺序落地：

1. 验证基础集成是否可用
2. 搭建 Telegram 测试工作流
3. 搭建 GPT-5.4 测试工作流
4. 搭建 RSS 抓取工作流
5. 搭建主工作流的去重 / 过滤 / 排序逻辑
6. 搭建 Telegram 早报发布工作流
7. 手动测试
8. 开启定时调度
9. 补充 RSSHub 来源
10. 仅在必要时加入网页抓取兜底

---

## 5. 必备凭证

优先在 n8n 中创建以下凭证：

1. Telegram Bot
2. GPT-5.4 API 凭证
3. 可选的 API/HTTP 认证凭证
4. 可选的 RSSHub 访问配置（如果 RSSHub 为自建或经反代暴露）

验证清单：

- Telegram 测试消息发送成功
- GPT-5.4 能返回一条简短中文句子
- RSS Read 能读取一个公开 RSS
- HTTP Request 能访问外部 URL

---

## 6. 工作流定义

## 6.1 工作流：`telegram-test`

用途：

- 验证 Telegram Bot 能正常发送消息

节点：

1. `Manual Trigger`
2. `Set`
3. `Telegram`

预期消息：

- 简单文本，例如：
  - `n8n Telegram test ok`

成功标准：

- 目标 Telegram 会话中能收到消息

---

## 6.2 工作流：`llm-test`

用途：

- 验证 GPT-5.4 能生成一条简短中文摘要

节点：

1. `Manual Trigger`
2. `Set`
3. `LLM 节点 / OpenAI 兼容节点`
4. `Set` 或 `Code` 用于检查输出

输入示例：

- title: `Apple reportedly expands Vision Pro ecosystem plans`
- source: `Example Source`
- summary: `Apple is said to be discussing broader content and hardware integrations around Vision Pro.`

Prompt 目标：

- 只返回一条简洁中文句子
- 长度控制在 `15 到 35` 个中文字符
- 仅保留事实，不加观点，不夸张

成功标准：

- 输出稳定的短中文摘要
- 不带 markdown 包裹
- 不附带多余解释

---

## 6.3 工作流：`source-fetch-rss`

用途：

- 读取所有选定来源
- 统一输出字段
- 返回干净的条目列表

建议首批来源数：

- `3 到 5` 个

建议来源类别：

1. 公开科技 RSS
2. AR/XR 垂直 RSS
3. Hacker News 或类似公开 feed
4. 1 个稳定的 RSSHub 路由

节点：

1. 开发阶段使用 `Manual Trigger`
2. 多个 `RSS Read`
3. 可选 `HTTP Request` 用于 API 来源
4. `Merge`
5. `Set` 或 `Code` 进行字段标准化

标准化输出结构：

```json
{
  "title": "",
  "summary": "",
  "url": "",
  "source": "",
  "platform": "",
  "publishedAt": "",
  "author": "",
  "content": "",
  "tags": []
}
```

标准化规则：

- `title`：必填
- `url`：必填
- `summary`：优先使用 RSS 自带 description / preview
- `source`：每个分支固定写明来源名
- `platform`：取值限定为 `rss`、`rsshub`、`api`、`web`
- `publishedAt`：尽量转为标准时间字符串
- `content`：可选，默认保持简短
- `tags`：可选数组

来源分支规则：

- 每个来源单独一条分支
- 不要把不同站点的抽取逻辑混在一起

成功标准：

- 所有来源统一输出同一字段结构
- 空条目和损坏条目在离开此工作流前被丢弃

---

## 6.4 工作流：`daily-news-main`

用途：

- 每天早晨自动运行
- 从来源工作流收集条目
- 去重
- 过滤无关内容
- 排序和截断
- 将最终结果传给发布工作流

节点：

1. `Schedule Trigger`
2. `Execute Workflow` -> `source-fetch-rss`
3. `Remove Duplicates`
4. `Code` -> 先标准化标题/URL，增强去重效果
5. `IF` 或 `Code` -> 关键词过滤
6. `Code` -> 打分排序
7. `Sort`
8. `Limit`
9. `IF` -> 如果最终数量为 0，则停止或告警
10. `Execute Workflow` -> `digest-publish-telegram`

调度建议：

- 每天早晨执行 1 次
- 推荐时间：
  - `08:00 Asia/Shanghai`

去重策略：

### 第一层：按 URL 去重

对 URL 做标准化后再比较：

- 去掉追踪参数（如果可行）
- 去掉 fragment
- 在安全前提下去掉尾部斜杠

### 第二层：按标题去重

对标题做标准化：

- 转小写
- 去首尾空格
- 合并连续空格
- 视情况去掉噪音标点

### 第三层：可选指纹去重

建议使用：

- 标准化后的标题 + 短摘要

过滤策略：

任何 LLM 调用前，必须先进行规则过滤。

首版保留关键词：

- `AR`
- `VR`
- `MR`
- `XR`
- `Spatial Computing`
- `Vision Pro`
- `Meta`
- `Quest`
- `Apple`
- `AI`
- `AIGC`
- `OpenAI`
- `Google`
- `NVIDIA`
- `chip`
- `robot`
- `robotics`
- `大模型`
- `芯片`
- `机器人`

首版排除关键词：

- `招聘`
- `促销`
- `抽奖`
- `广告`
- `直播预告`

基础过滤规则：

- 标题命中保留关键词则优先保留
- 标题或摘要明显属于目标领域则保留
- 标题或摘要命中排除关键词则丢弃
- 标题为空则丢弃
- URL 不可用则丢弃

排序策略：

建议打分项：

- 标题命中核心关键词：高权重
- 摘要命中关键词：中权重
- 来源优先级：中权重
- 发布时间新近程度：中权重
- 重复/近似重复倾向：负权重

建议最终数量上限：

- 每日报保留 `10 到 15` 条

成功标准：

- 只有相关条目能进入下一步
- 条目总量保持紧凑
- 被过滤掉的条目不得调用 LLM

---

## 6.5 工作流：`digest-publish-telegram`

用途：

- 为最终选中的内容生成极简摘要
- 组装 Telegram 早报正文
- 每天发送 1 次

节点：

1. `Execute Workflow Trigger` 或接收上游输入
2. `Split in Batches` 或逐条循环
3. `LLM`
4. `Code` 或 `Set` 组装单条展示内容
5. `Code` 将全部条目拼接为一份早报
6. `Telegram`

LLM 使用策略：

- 只处理已经过滤后的最终条目
- Phase 1 可以接受每条调用 1 次模型
- 每条只生成一句摘要
- 默认不要发送全文给模型
- 优先只传标题 + 短摘要 + 来源

建议传给 LLM 的字段：

- title
- source
- publishedAt
- summary preview
- url

默认不要发送：

- 全文正文
- 原始 HTML
- 大段抓取文本

Prompt 规则：

- 只返回一句简短中文摘要
- `15 到 35` 个中文字符
- 不要编号前缀
- 不要 markdown
- 不加个人观点
- 不做推测
- 尽量避免重复标题措辞
- 只保留事实核心

Telegram 输出模板：

```markdown
# 科技早报 | {{date}}

1. {{title}}
来源：{{source}}
摘要：{{short_summary}}
链接：{{url}}

2. {{title}}
来源：{{source}}
摘要：{{short_summary}}
链接：{{url}}
```

格式规则：

- 先保持简单
- 在稳定前，不要使用复杂 Telegram markdown
- 如果启用了 markdown parse mode，需要处理特殊字符转义
- 如果格式频繁报错，优先退回纯文本

成功标准：

- 每天成功发送 1 条完整早报
- 结构清晰易读
- 不要过度冗长
- 不要出现 markdown 解析失败

---

## 7. Token 控制规则

必须严格控制 LLM token 消耗。

规则如下：

1. 不允许对全部原始条目做摘要
2. 必须先去重
3. 必须先过滤，再调用 LLM
4. 最终进入摘要流程的条目限制为 `10 到 15` 条
5. 除非必要，不抓取全文
6. Prompt 必须短、小、确定性强
7. 每条只允许生成一句摘要
8. 如果来源自带摘要质量足够好，可以优先复用

可选优化：

- 对低优先级条目，不走 LLM，直接使用原摘要
- 仅对高分条目调用 LLM

---

## 8. 来源策略

优先级顺序：

1. 公开 RSS
2. RSSHub
3. API
4. 网页抓取兜底

原因：

- RSS 最稳定
- RSSHub 能扩展覆盖范围
- API 可用但可能限频
- 网页抓取维护成本最高

首批来源规则：

- 只使用稳定、公开、易维护的来源
- 初始来源数量保持较少
- 只有在连续几天稳定后再扩容

初期不要做：

- 一次性接入太多脆弱来源
- 反爬很强的网站
- 全站抓取
- 浏览器自动化抓取

---

## 9. 数据质量规则

每个条目至少必须满足：

- 标题不为空
- URL 可用
- 来源名明确
- 如果能提供时间，则时间字段合理

以下情况应直接丢弃：

- 标题为空
- URL 为空
- 标题明显无关
- 已判定为重复
- 内容明显是广告或促销噪音

---

## 10. 错误处理

Phase 1 最低要求：

1. 工作流失败时可发 Telegram 告警
2. 不同来源必须拆分分支，单个来源失败不应拖垮全部流程
3. 某个 RSS 来源失败时，主流程应尽量继续
4. 失败日志中要能定位到具体来源名

建议失败行为：

- 接受部分成功
- 只要剩余内容足够，就仍然生成并发送早报

可选后续增强：

- 如果某个来源连续多次失败，额外发送一条运维提醒

---

## 11. 日志与可观测性

Phase 1 最低日志要求：

- 执行时间
- 原始抓取总数
- 去重后的数量
- 最终保留数量
- 失败来源列表

存储方式可选：

- n8n 自带执行历史
- SQLite 支撑下的 n8n 内部数据
- 后续可接 Google Sheets
- 后续可接外部数据库

Phase 1 不要过度设计日志系统。

---

## 12. 操作者执行顺序

搭建时严格按以下顺序执行：

1. 搭建并验证 `telegram-test`
2. 搭建并验证 `llm-test`
3. 搭建并验证 `source-fetch-rss`
4. 首批只接入 `3 到 5` 个来源
5. 搭建 `daily-news-main`
6. 实现去重与过滤
7. 限制最终条目数
8. 搭建 `digest-publish-telegram`
9. 手动运行多次
10. 确认早报

---
**相关**：[[7.23-白嫖云盘cftc,tgState,tgNetDisc]] [[5.5-Cloudflare-AI内容流水线部署]] [[8.15-OpenClash节点延迟诊断]] [[5.5 Cloudflare AI内容流水线部署]] [[5.5 CloudflareSub Wrangler 部署]] [[5.8 JP NAS 本地部署 Kimi GLM 统一网关]] [[对话笔记全量梳理表]]
