---
title: Hermes Telegram Bot无回复排查SOP
publish: true
source: tx 171会话最高频主题 + NAS hermes-telegram-repair.md
origin: 原创初稿
---

# Hermes Telegram Bot无回复排查SOP（初稿）

按顺序查：网关进程活着吗→限流/rate-limit→鉴权token过期→模型过期切换→Honcho记忆迁移问题。
这是全网会话里出现频率最高的主题（~25%），修一次写成SOP， liability终结。
（待补：gateway service与bot binding检查清单，HANDOFF-zen-rotator需补读。）


> 已发布：https://blog.fengx.eu.org/migrated/telegram-bot-noreply-sop（2026-10-05）
