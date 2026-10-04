---
title: OpenCode 斜杠命令在 MindFS 下失效？三条绕法
date: 2026-10-05
tags: [AI, OpenCode, MindFS, 上下文工程]
description: MindFS 里 OpenCode 的 / 命令触发不了：根因是 ACP 实现缺内置命令，附三条实测绕法。
---

# OpenCode 斜杠命令在 MindFS 下失效？三条绕法

## 现象

同样跑在 MindFS 里，Codex 下 `/compact` 一切正常，OpenCode 下按 `/` 压根触发不了内置命令（压缩上下文等）。第一反应是配置问题，找开发者确认后：**这是 agent 自身 ACP 实现有关，内置命令没开发**，换配置没用。

## 绕法一：借 Codex 的压缩用（最稳）

OpenCode 上下文攒到 120K 切到 Codex，Codex 里 `/compact` 压到 10K 左右，再切回 OpenCode。实测可行，间接绕过缺失的内置命令。注意切过去时工具上下文会膨胀一点（120K 变 130K 级别），属正常。

## 绕法二：调小上限 forced 自动压缩

模型上下文上限如 1M，手动改成 200K，超限即触发自动压缩。简单粗暴，适合长会话懒人。

## 绕法三：MindFS 内直接换 CLI 搬运

上下文跟着会话走，换 CLI 即换压缩能力。代价是每次切换都有膨胀与格式损耗，适合救急不适合常规。

## 一句话

不是你的问题，是实现缺失。在官方补齐前，绕法一是主力：让有压缩能力的 CLI 当“压缩机”，OpenCode 只管写代码。
