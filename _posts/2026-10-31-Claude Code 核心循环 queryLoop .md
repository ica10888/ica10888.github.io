---
layout:     post
title:      Claude Code 核心循环 queryLoop 
subtitle:   Claude Code 核心循环 queryLoop 
date:       2026-10-31
author:     ica10888
catalog: true
tags:
    - AI 
    - Prompts engineering
---

# Claude Code 核心循环 queryLoop

queryLoop方法是 Claude Code 的核心循环，其代码经历多次迭代，但是我们仍然可以提取出其核心逻辑，主要精炼之后，就是以下两点：
- 7+ 个 continue 站点，每个对应不同的恢复场景
- 4 级压缩管道（Compaction Pipeline），用于上下文的压缩和保存上下文

### 7+ 个 continue 站点
```text
Continue Site 1: Proactive Compaction
触发: token 超过阈值
动作: autocompact → 新消息 → continue

Continue Site 2: Prompt Too Long
触发: API 返回 prompt-too-long 错误
动作: context-collapse → reactive compact

Continue Site 3: Max Output Tokens
触发: 模型输出截断
动作: 升级 8k→64k → 多轮重试（最多3次）

Continue Site 4: Fallback Model
触发: FallbackTriggeredError
动作: 切换模型 → 重试请求

Continue Site 5: Stop Hook Blocking
触发: 用户 Hook 要求额外轮次
动作: 注入 Hook 消息 → continue

Continue Site 6: Image/Media Errors
触发: ImageSizeError / ImageResizeError
动作: 反应式压缩（移除图片）→ continue

Continue Site 7: Tool Execution
触发: 正常工具执行完成
动作: 收集结果 → 更新状态 → continue

return Terminal — 唯一的退出点
条件: 无工具调用 + Stop Hook 不阻止
```

### 4 级压缩管道
```text
Level 1: Snip
历史截断成本: 极低 | 延迟: ~0ms

释放少量 token

Level 2: Microcompact
老化工具结果缩减
成本: 低 | 延迟: ~1ms

边界消息延迟

Level 3: Context-Collapse
读时投射（不修改数组）
成本: 中 | 延迟: ~5ms

如果仍超阈值                            如果低于阈值

Level 4: Autocompact                   跳过 Autocompact
LLM 全对话摘要                          保留粒
成本: 高 | 延迟: ~2s
```










