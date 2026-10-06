---
title: OpenCode斜杠命令在MindFS下失效？三条绕法
publish: true
source: Dell MindFS会话（开发者回复核查）
origin: 原创初稿
---

# OpenCode斜杠命令在MindFS下失效？三条绕法（初稿）

现象：MindFS里用OpenCode，/压根触发不了内置命令（压缩上下文等），Codex下一切正常。
根因：和agent的ACP实现有关，OpenCode内置命令没开发，不是配置问题。
绕法一：切到Codex里压缩上下文到10K，再切回OpenCode，间接绕过。
绕法二：把模型上下文上限（如1M）手动改小到200K，超限自动压缩。
绕法三：MindFS内直接切CLI搬运上下文（注意工具上下文会膨胀，120K过去变130K）。


> 已发布：https://blog.fengx.eu.org/migrated/opencode-slash-mindfs-workaround（2026-10-05）

---

---
**相关**： [[00-索引-AI工程索引]] [[OpenCode 斜杠命令在 MindFS 下失效？三条绕法]]
