---
layout:     post
title:      Codex 会话记忆与上下文压缩：基于固定 Rust 源码的完整请求分析
subtitle:  Codex 会话记忆与上下文压缩：基于固定 Rust 源码的完整请求分析
date:       2026-10-26
author:     ica10888
catalog: true
tags:
    - AI 
    - Prompts engineering
---

# Codex 会话记忆与上下文压缩：基于固定 Rust 源码的完整请求分析

## 研究边界

本文是独立重写版本。分析只依据本次会话已经确认的本地 Git 源码，不读取或改写上一版文章，不使用博客、搜索摘要或第三方资料。研究仓库为 `openai/codex`，固定 commit：

`2351d9e1b608e6f9d9a3699b71d7eb39ee41cfa4`

调研日期：2026-10-24。本文中的 GitHub 链接都固定到这个 commit。文章重点分析 Rust CLI/core 的会话历史和上下文压缩；不把该开源实现推断成所有 Codex 产品的内部实现。

为了让每个模型请求都能展开，使用一个教学 fixture：

- 普通 Responses API；
- 不展开权限、沙箱、Guardian、MCP 和文件存储细节；
- 模型名、session/thread/turn ID、token 数和时间是伪值；
- 只使用 `exec_command` 作为示意工具；
- 假设历史已经增长到自动压缩阈值；
- 压缩完成后，用户继续要求验证 README.md；
- `tools: []`、可选字段省略规则和 developer message 位置，按固定源码的序列化逻辑书写。

“记忆”在本文中不是向量数据库或永久人格记忆，而是 Codex 在会话生命周期内维护的结构化历史、上下文片段、压缩摘要和窗口/token 状态。

---

## 一、源码中的会话、turn、history 和 Prompt

### 1. Session 是会话级运行容器

`Session` 持有线程级状态和服务，包括会话历史、输入队列、模型客户端、工具运行时、token 使用状态、world-state checkpoint 等。用户的一次请求进入一个 turn，但一个 session/thread 可以包含多个 turn。

核心源码入口：

- [`run_turn`](https://github.com/openai/codex/blob/2351d9e1b608e6f9d9a3699b71d7eb39ee41cfa4/codex-rs/core/src/session/turn.rs)
- [`Session` 历史和用户输入记录](https://github.com/openai/codex/blob/2351d9e1b608e6f9d9a3699b71d7eb39ee41cfa4/codex-rs/core/src/session/mod.rs)
- [`History`](https://github.com/openai/codex/blob/2351d9e1b608e6f9d9a3699b71d7eb39ee41cfa4/codex-rs/core/src/context_manager/history.rs)

### 2. turn 是一次目标编排生命周期

`run_turn` 的注释把一次 turn 描述为循环：一次 sampling 可能返回 function call 或 assistant message；若返回 function call，Codex 在本地执行工具，再把工具输出放入下一次 sampling；如果返回 assistant message，则进入结束判定。

实际结束还受 `Completed`、`end_turn`、pending input 和未完成工具 future 影响。因此不能把“收到 assistant message”机械地等同于立刻结束 turn。

抽象流程：

```text
run_turn
  -> 捕获 step context、历史和 world state
  -> 记录用户输入与上下文片段
  -> clone_history().for_prompt(...)
  -> build_prompt(...)
  -> client_session.stream(...)
  -> 处理模型 response events
  -> FunctionCall 时执行工具
  -> ToolOutput 写回 history
  -> 下一次 sampling
  -> assistant message + end_turn 条件满足
  -> turn 完成
```

### 3. History 是结构化序列

`History` 内部不是一段拼接字符串，而是带 envelope metadata 的 `ResponseItem` 序列。一个 item 可能是：

- `Message`：user、developer 或 assistant message；
- `Reasoning`；
- `FunctionCall`；
- `FunctionCallOutput`；
- `CustomToolCall` 和对应 output；
- `AdditionalTools`；
- context/world-state update；
- compaction summary；
- 其他协议 item。

调用 `clone_history().for_prompt(input_modalities)` 时，源码会复制、规范化历史，并根据当前模型能力处理图片/音频和工具调用输出配对。

实现位置：

[`History::for_prompt`](https://github.com/openai/codex/blob/2351d9e1b608e6f9d9a3699b71d7eb39ee41cfa4/codex-rs/core/src/context_manager/history.rs#L597-L614)

### 4. Prompt 不是 wire JSON

`core/src/client_common.rs` 的 `Prompt` 大致包含：

```rust
pub struct Prompt {
    pub input: Vec<ResponseItem>,
    pub(crate) tools: Arc<[ToolSpec]>,
    pub(crate) parallel_tool_calls: bool,
    pub base_instructions: BaseInstructions,
    pub output_schema: Option<Value>,
    pub output_schema_strict: bool,
    pub(crate) cyber_access_program: Option<CyberAccessProgram>,
}
```

源码链接：

[`client_common.rs::Prompt`](https://github.com/openai/codex/blob/2351d9e1b608e6f9d9a3699b71d7eb39ee41cfa4/codex-rs/core/src/client_common.rs#L23-L116)

它之后还要经过 `build_responses_request`，才变成 `ResponsesApiRequest`，最后由 HTTP/WebSocket endpoint 序列化为 wire JSON。

因此要区分：

```text
History
  -> Prompt
  -> ResponsesApiRequest
  -> HTTP/WebSocket JSON
```

---

## 二、一个会逐渐变长的 README 会话

为便于说明，假设会话经历以下步骤。

### 第一个用户目标

```text
请创建 README.md，内容如下：

# Codex Demo

这是一个使用 Rust 编写的 Agent 项目。
```

模型先调用 `exec_command` 检查文件，然后调用同一个工具写入文件，再调用工具读取文件验证。用户期间还追加了一条 steer：

```text
补充限制：只修改 README.md，不要覆盖其他文件。
```

经过这些步骤，历史大致包含：

```text
developer base instructions
user: 创建 README.md
assistant/function_call: 检查 README.md
function_call_output: 文件不存在
user: 只修改 README.md
assistant/function_call: 写入 README.md
function_call_output: 写入成功
assistant/function_call: cat README.md
function_call_output: README 内容
assistant message: 已完成
```

这里的每个工具调用和工具输出都是结构化 item。工具执行本身没有产生新的模型请求；只有工具 output 被记录后，下一次 sampling 才会使用它。

### 历史为什么会触发压缩

每次普通 sampling 前，`run_turn` 会检查当前上下文状态。代码会读取 token 使用量、模型上下文窗口和自动压缩配置，判断是否需要：

- 继续普通采样；
- 自动压缩；
- context window rollover；
- 在错误或重试路径中重新尝试。

相关代码位于：

- [`run_pre_sampling_compact`](https://github.com/openai/codex/blob/2351d9e1b608e6f9d9a3699b71d7eb39ee41cfa4/codex-rs/core/src/session/turn.rs#L1304-L1331)
- [`run_auto_compact`](https://github.com/openai/codex/blob/2351d9e1b608e6f9d9a3699b71d7eb39ee41cfa4/codex-rs/core/src/session/turn.rs#L1435-L1508)
- turn 主循环中的 token 状态和压缩判断（同一 `turn.rs` 的主循环部分）

重要边界是：

```text
普通 sampling request
    -> 历史增加模型输出和工具结果
    -> token/context 状态检查
    -> 触发 compaction
    -> compaction sampling request
    -> 压缩模型返回摘要
    -> 结构化替换 history
    -> 下一次普通 sampling request
```

---

## 三、自动压缩的入口和分支

`run_auto_compact` 会根据当前 provider 和配置选择不同实现。固定源码中可以看到：

- token budget 专用路径；
- remote compaction 路径；
- 普通 Responses inline compaction 路径；
- 不同 provider capability 对应的实现。

本文固定使用普通 Responses 的 `run_inline_auto_compact_task`，因为它最直接地展示“复制历史 → 加入摘要提示 → 发模型请求 → 用摘要替换历史”。

```rust
pub(crate) async fn run_inline_auto_compact_task(
    sess: Arc<Session>,
    turn_context: Arc<TurnContext>,
    replacement_step_context: Arc<StepContext>,
    world_state: Arc<WorldState>,
    reason: CompactionReason,
    phase: CompactionPhase,
) -> CodexResult<()> {
    let prompt = turn_context
        .config
        .compact_prompt
        .as_deref()
        .unwrap_or(SUMMARIZATION_PROMPT)
        .to_string();

    let input = vec![UserInput::Text {
        text: prompt,
        text_elements: Vec::new(),
    }];

    run_compact_task_inner(...).await?;
    Ok(())
}
```

源码：

[`compact.rs::run_inline_auto_compact_task`](https://github.com/openai/codex/blob/2351d9e1b608e6f9d9a3699b71d7eb39ee41cfa4/codex-rs/core/src/compact.rs#L113-L148)

这段代码说明两件事：

1. 压缩 prompt 是一条合成的 `UserInput::Text`，不是用户刚刚输入的普通消息；
2. `config.compact_prompt` 可以覆盖默认模板，否则使用 `SUMMARIZATION_PROMPT`。

---

## 四、压缩模板的真实内容

### 1. `SUMMARIZATION_PROMPT`

`codex-rs/prompts/src/compact.rs`：

```rust
pub const SUMMARIZATION_PROMPT: &str =
    include_str!("../templates/compact/prompt.md");
```

固定文件内容是：

```text
You are performing a CONTEXT CHECKPOINT COMPACTION. Create a handoff summary for another LLM that will resume the task.

Include:
- Current progress and key decisions made
- Important context, constraints, or user preferences
- What remains to be done (clear next steps)
- Any critical data, examples, or references needed

Be concise, structured, and focused on helping the next LLM continue the task.
```

文件链接：

[`prompts/templates/compact/prompt.md`](https://github.com/openai/codex/blob/2351d9e1b608e6f9d9a3699b71d7eb39ee41cfa4/codex-rs/prompts/templates/compact/prompt.md)

这段文本会被包装成一个临时 user message，和已有历史一起进入 compaction sampling request。

### 2. `SUMMARY_PREFIX`

`codex-rs/prompts/src/compact.rs` 还定义：

```rust
pub const SUMMARY_PREFIX: &str =
    include_str!("../templates/compact/summary_prefix.md");
```

固定文件内容是：

```text
Another language model started to solve this problem and produced a summary of its thinking process. You also have access to the state of the tools that were used by the other language model. Use this information to assist you in continuing the task. Avoid duplicating work already done.
```

文件链接：

[`prompts/templates/compact/summary_prefix.md`](https://github.com/openai/codex/blob/2351d9e1b608e6f9d9a3699b71d7eb39ee41cfa4/codex-rs/prompts/templates/compact/summary_prefix.md)

需要注意：`SUMMARY_PREFIX` 不一定作为独立输入发送给压缩模型。它是在压缩模型返回摘要后，由：

```rust
let summary_text = format!("{SUMMARY_PREFIX}\n{summary_suffix}");
```

拼接成最终的 compaction summary，再写进替换后的 history。也就是说：

```text
prompt.md
  -> 压缩请求中的临时 user input

summary_prefix.md + 模型摘要
  -> 压缩完成后的 CompactionSummary history item
```

### 3. 前缀文本的版本事实

固定 commit 中的 `summary_prefix.md` 内容应以仓库文件为准。文章中的伪请求会把文件内容完整放入 JSON 字符串；如果仓库文本在不同 commit 中变化，不能把其他版本的前缀混入当前分析。

---

## 五、`run_compact_task_inner_impl` 如何构造压缩请求

核心函数：

[`compact.rs::run_compact_task_inner_impl`](https://github.com/openai/codex/blob/2351d9e1b608e6f9d9a3699b71d7eb39ee41cfa4/codex-rs/core/src/compact.rs#L259-L430)

源码顺序如下。

### 第一步：创建压缩生命周期事件

```rust
let compaction_item =
    TurnItem::ContextCompaction(ContextCompactionItem::new());
sess.emit_turn_item_started(&turn_context, &compaction_item).await;
```

这是本地 turn item 生命周期事件，不是模型请求。

### 第二步：把压缩提示加入临时 history

```rust
let initial_input_for_turn: ResponseInputItem =
    ResponseInputItem::from(input);

let mut history = sess.clone_history().await;
let input_goal_ids = UserGoalUpdate::message_ids(history.raw_items());
history.record_items(
    &[initial_input_for_turn.into()],
    turn_context.model_info().truncation_policy.into(),
);
```

关键点：

- `sess.clone_history()` 得到当前历史副本；
- 原始 session history 此时还没有被替换；
- 合成的 compaction prompt 被追加到临时副本；
- `input_goal_ids` 记录压缩前的用户目标信息。

因此，压缩请求看到的是“旧历史 + 压缩指令”，而不是只有压缩指令。

### 第三步：为每次尝试重新生成 turn input

```rust
let mut turn_input = history
    .clone()
    .for_prompt(&turn_context.model_info().input_modalities);

sess.services
    .executed_tool_calls
    .attach_to_compaction_prompt(&mut turn_input);
```

`for_prompt` 会执行历史规范化。压缩请求可能还会附加已执行工具调用的受限 metadata。

### 第四步：避免重复 base instructions

```rust
let base_instructions = if turn_input
    .iter()
    .any(BaseInstructionsFragment::matches_item)
{
    BaseInstructions {
        text: String::new(),
        provenance: None,
    }
} else {
    sess.get_prompt_base_instructions().await
};
```

如果临时 history 已经有带 `model.base_instructions` 标记的 developer item，就把 `Prompt.base_instructions.text` 设为空，避免 `build_responses_request` 再插入一份重复 developer message。

如果没有该 item，就从 session 获取 base instruction，由后续 `build_responses_request` 转成 developer input item。

### 第五步：构造压缩 Prompt

```rust
let prompt = Prompt {
    input: turn_input,
    base_instructions,
    cyber_access_program: turn_context.cyber_access_program,
    ..Default::default()
};
```

这里的 `..Default::default()` 很重要：

- `tools` 为空；
- `parallel_tool_calls` 为 false；
- `output_schema` 为 None；
- `output_schema_strict` 为 true；
- compaction 本身不要求模型调用工具。

所以压缩请求与普通 coding turn 请求不同：它主要提供历史和摘要指令，不提供 `exec_command` 等普通工具。

### 第六步：调用 `drain_to_completed`

```rust
let responses_metadata = sess
    .compaction_responses_metadata(turn_context.as_ref(), compaction_metadata)
    .await;

let attempt_result = drain_to_completed(
    &sess,
    turn_context.as_ref(),
    &mut client_session,
    &responses_metadata,
    &prompt,
    compaction_metadata.phase(),
)
.await;
```

`drain_to_completed` 内部才调用 `client_session.stream(...)`。到这里才是一次真正的 compaction sampling request。

---

## 六、压缩请求的完整教学伪 JSON

下面假设压缩前历史已经包含以下结构化 item：

- developer base instruction；
- 原始 user 请求；
- 检查 README 的 FunctionCall 和 FunctionCallOutput；
- steer user message；
- 写入 README 的 FunctionCall 和 FunctionCallOutput；
- 读取 README 的 FunctionCall 和 FunctionCallOutput；
- assistant 已完成消息。

为避免将前置动态配置写成源码固定值，以下是教学 fixture。模型名、service tier、metadata ID 和时间均为伪值。工具列表为空；没有把普通工具 schema 误放进 compaction 请求。

### 1. 压缩前最后一次普通请求

这次请求的目的仍是普通 coding turn，不是压缩。它包含完整的增长历史和 `exec_command` 工具定义：

```json
{
  "model": "gpt-5.2-codex",
  "stream": true,
  "service_tier": "priority",
  "input": [
    {
      "type": "message",
      "id": "msg_base_001",
      "role": "developer",
      "content": [
        {
          "type": "input_text",
          "text": "You are a coding agent in the Codex CLI.\nUse the available tools when needed.\nKeep the requested file scope unchanged."
        }
      ]
    },
    {
      "type": "message",
      "role": "user",
      "content": [
        {
          "type": "input_text",
          "text": "请创建 README.md，内容如下：\n\n# Codex Demo\n\n这是一个使用 Rust 编写的 Agent 项目。"
        }
      ]
    },
    {
      "type": "function_call",
      "id": "fc_001",
      "name": "exec_command",
      "arguments": "{\"cmd\":\"test -f README.md && cat README.md || printf 'README absent\\n'\",\"yield_time_ms\":1000}",
      "call_id": "call_001"
    },
    {
      "type": "function_call_output",
      "call_id": "call_001",
      "output": "Chunk ID: chunk_001\nWall time: 0.0040 seconds\nProcess exited with code 1\nOutput:\nREADME absent"
    },
    {
      "type": "message",
      "role": "user",
      "content": [
        {
          "type": "input_text",
          "text": "补充限制：只修改 README.md，不要覆盖其他文件。"
        }
      ]
    },
    {
      "type": "function_call",
      "id": "fc_002",
      "name": "exec_command",
      "arguments": "{\"cmd\":\"cat > README.md <<'EOF'\\n# Codex Demo\\n\\n这是一个使用 Rust 编写的 Agent 项目。\\nEOF\",\"yield_time_ms\":1000}",
      "call_id": "call_002"
    },
    {
      "type": "function_call_output",
      "call_id": "call_002",
      "output": "Chunk ID: chunk_002\nWall time: 0.0180 seconds\nProcess exited with code 0\nOutput:"
    },
    {
      "type": "function_call",
      "id": "fc_003",
      "name": "exec_command",
      "arguments": "{\"cmd\":\"cat README.md\",\"yield_time_ms\":1000}",
      "call_id": "call_003"
    },
    {
      "type": "function_call_output",
      "call_id": "call_003",
      "output": "Chunk ID: chunk_003\nWall time: 0.0030 seconds\nProcess exited with code 0\nOutput:\n# Codex Demo\n\n这是一个使用 Rust 编写的 Agent 项目。"
    },
    {
      "type": "message",
      "role": "assistant",
      "content": [
        {
          "type": "output_text",
          "text": "README.md 已创建并完成初步验证。"
        }
      ]
    }
  ],
  "tools": [
    {
      "type": "function",
      "name": "exec_command",
      "description": "Runs a command in a PTY, returning output or a session ID for an ongoing command.",
      "strict": false,
      "parameters": {
        "type": "object",
        "properties": {
          "cmd": {
            "type": "string",
            "description": "Shell command to execute."
          },
          "workdir": {
            "type": "string",
            "description": "Working directory for the command. Defaults to the turn cwd."
          },
          "tty": {
            "type": "boolean",
            "description": "True allocates a PTY; false or omitted uses plain pipes."
          },
          "yield_time_ms": {
            "type": "number",
            "description": "Wait before yielding output."
          },
          "max_output_tokens": {
            "type": "number",
            "description": "Output token budget."
          }
        },
        "required": ["cmd"],
        "additionalProperties": false
      }
    }
  ],
  "tool_choice": "auto",
  "parallel_tool_calls": true,
  "reasoning": {
    "effort": "medium"
  },
  "store": false,
  "include": ["reasoning.encrypted_content"],
  "prompt_cache_key": "session_001",
  "client_metadata": {
    "x-codex-installation-id": "install_001",
    "session_id": "session_001",
    "thread_id": "thread_001",
    "x-codex-window-id": "window_001",
    "turn_id": "turn_001"
  }
}
```

该 JSON 是普通请求的完整 fixture。它说明压缩不是在 `build_prompt` 阶段凭空发生，而是在一次普通请求结束后，由 turn 的 token/context 检查决定是否调用压缩任务。

### 2. 真正的 compaction sampling request

`run_compact_task_inner_impl` 将压缩模板作为临时 user input 追加到复制的 history 后，再调用 `drain_to_completed`。下面是完整的逻辑请求伪内容：

```json
{
  "model": "gpt-5.2-codex",
  "stream": true,
  "service_tier": "priority",
  "input": [
    {
      "type": "message",
      "id": "msg_base_001",
      "role": "developer",
      "content": [
        {
          "type": "input_text",
          "text": "You are a coding agent in the Codex CLI.\nUse the available tools when needed.\nKeep the requested file scope unchanged."
        }
      ]
    },
    {
      "type": "message",
      "role": "user",
      "content": [
        {
          "type": "input_text",
          "text": "请创建 README.md，内容如下：\n\n# Codex Demo\n\n这是一个使用 Rust 编写的 Agent 项目。"
        }
      ]
    },
    {
      "type": "function_call",
      "id": "fc_001",
      "name": "exec_command",
      "arguments": "{\"cmd\":\"test -f README.md && cat README.md || printf 'README absent\\n'\",\"yield_time_ms\":1000}",
      "call_id": "call_001"
    },
    {
      "type": "function_call_output",
      "call_id": "call_001",
      "output": "Chunk ID: chunk_001\nWall time: 0.0040 seconds\nProcess exited with code 1\nOutput:\nREADME absent"
    },
    {
      "type": "message",
      "role": "user",
      "content": [
        {
          "type": "input_text",
          "text": "补充限制：只修改 README.md，不要覆盖其他文件。"
        }
      ]
    },
    {
      "type": "function_call",
      "id": "fc_002",
      "name": "exec_command",
      "arguments": "{\"cmd\":\"cat > README.md <<'EOF'\\n# Codex Demo\\n\\n这是一个使用 Rust 编写的 Agent 项目。\\nEOF\",\"yield_time_ms\":1000}",
      "call_id": "call_002"
    },
    {
      "type": "function_call_output",
      "call_id": "call_002",
      "output": "Chunk ID: chunk_002\nTime: 0.0180 seconds\nProcess exited with code 0\nOutput:"
    },
    {
      "type": "function_call",
      "id": "fc_003",
      "name": "exec_command",
      "arguments": "{\"cmd\":\"cat README.md\",\"yield_time_ms\":1000}",
      "call_id": "call_003"
    },
    {
      "type": "function_call_output",
      "call_id": "call_003",
      "output": "Chunk ID: chunk_003\nTime: 0.0030 seconds\nProcess exited with code 0\nOutput:\n# Codex Demo\n\n这是一个使用 Rust 编写的 Agent 项目。"
    },
    {
      "type": "message",
      "role": "assistant",
      "content": [
        {
          "type": "output_text",
          "text": "README.md 已创建并完成初步验证。"
        }
      ]
    },
    {
      "type": "message",
      "role": "user",
      "content": [
        {
          "type": "input_text",
          "text": "You are performing a CONTEXT CHECKPOINT COMPACTION. Create a handoff summary for another LLM that will resume the task.\n\nInclude:\n- Current progress and key decisions made\n- Important context, constraints, or user preferences\n- What remains to be done (clear next steps)\n- Any critical data, examples, or references needed\n\nBe concise, structured, and focused on helping the next LLM continue the task."
        }
      ]
    }
  ],
  "tools": [],
  "tool_choice": "auto",
  "parallel_tool_calls": false,
  "reasoning": {
    "effort": "medium"
  },
  "store": false,
  "include": ["reasoning.encrypted_content"],
  "prompt_cache_key": "session_001",
  "client_metadata": {
    "x-codex-installation-id": "install_001",
    "session_id": "session_001",
    "thread_id": "thread_001",
    "x-codex-window-id": "window_001",
    "turn_id": "compact_turn_001"
  }
}
```

这里有几个源码级结论：

- 压缩请求的 `input` 仍包含旧历史；摘要指令只是最后追加的临时 user message；
- `Prompt::default()` 使 tools 为空，`build_responses_request` 对普通 Responses 路径会得到空工具数组；
- `parallel_tool_calls` 来自默认 Prompt，因此为 false；
- 普通 Responses 下 `reasoning.context` 通常省略，Lite 模式才可能设置 `all_turns`；
- `prompt_cache_key` 和 `client_metadata` 是服务端请求元数据，不是摘要内容；
- 该请求也通过流式 `client_session.stream` 发送，但 `drain_to_completed` 会一直消费到 `response.completed`。

### 3. 压缩模型返回

压缩模型返回的关键 output item 可以是：

```json
{
  "type": "message",
  "role": "assistant",
  "content": [
    {
      "type": "output_text",
      "text": "当前任务是创建 README.md。用户要求文件包含标题 # Codex Demo 和中文说明。已完成文件写入，并通过 cat README.md 验证内容正确。用户还要求只修改 README.md。当前没有未完成的文件修改；后续如需继续，应先读取现有 README.md，再遵守文件范围限制。"
    }
  ]
}
```

该响应不是用户最终看到的普通 assistant 答案，而是 compaction 的摘要候选。`drain_to_completed` 会收集 `OutputItemDone`，等待 `Completed`，记录 response ID 和 token usage，然后返回 `CompactionResponse`。

源码：

[`compact.rs::drain_to_completed`](https://github.com/openai/codex/blob/2351d9e1b608e6f9d9a3699b71d7eb39ee41cfa4/codex-rs/core/src/compact.rs#L792-L858)

压缩阶段如果遇到 context window exceeded，源码会从临时 history 开头移除旧 item，并重新尝试；如果是普通流错误且 provider 允许重试，则按 retry/backoff 重试。这个处理仍属于 compaction task，不是普通 coding turn 的 tool loop。

---

## 七、摘要如何变成新的历史

### 1. 提取摘要文本

`run_compact_task_inner_impl` 在压缩请求成功后取得 `history_snapshot`，根据 compaction phase 选择摘要来源：

- post-turn compaction：从本次 compaction response output 中读取最后一个 assistant message；
- 其他 phase：从当前 history 中读取最后一个 assistant message。

然后执行：

```rust
let summary_text = format!("{SUMMARY_PREFIX}\n{summary_suffix}");
```

### 2. 保留用户消息

```rust
let user_messages = collect_annotated_user_messages(history_items);
let mut new_history =
    build_compacted_history(Vec::new(), &user_messages, &summary_text);
```

`build_compacted_history_with_limit` 的行为是：

- 从用户消息集合的尾部倒序选择；
- 在 `COMPACT_USER_MESSAGE_MAX_TOKENS` 预算内保留消息；
- 如果最后一条消息超过预算，则按 token policy 截断为文本；
- 保留必要的 harness metadata；
- 最后追加一个 `CompactionSummary`。

这说明压缩不是“删掉前 N 个字符串”。它有消息类型、用户目标、metadata 和 token 预算的结构化规则。

源码：

- [`build_compacted_history`](https://github.com/openai/codex/blob/2351d9e1b608e6f9d9a3699b71d7eb39ee41cfa4/codex-rs/core/src/compact.rs#L673-L760)
- [`CompactionSummary`](https://github.com/openai/codex/blob/2351d9e1b608e6f9d9a3699b71d7eb39ee41cfa4/codex-rs/core/src/context/compaction_summary.rs)

### 3. `CompactionSummary` 的角色

`CompactionSummary` 实现 `ContextualUserFragment`：

```rust
impl ContextualUserFragment for CompactionSummary {
    fn content_kind(&self) -> ContentItemKind {
        ContentItemKind("compaction.summary".to_string())
    }

    fn role(&self) -> &'static str {
        "user"
    }

    fn body(&self) -> String {
        self.summary.clone()
    }
}
```

所以最终历史中的压缩摘要是一个带 `compaction.summary` 标记的 user message，而不是 assistant message，也不是隐藏的本地变量。

### 4. 重新加入 initial context

```rust
let (new_history, world_state_baseline) =
    build_compaction_replacement_history(
        sess.as_ref(),
        &replacement_step_context,
        &world_state,
        new_history,
    )
    .await;
```

这个函数会重新生成 initial context/world-state，拆分 prefix 和普通 context，并调用 `assemble_compaction_history` 把它们放回新历史。这样 compaction 后不会只剩摘要而丢掉必要的模型上下文。

### 5. 替换 session history

```rust
sess.replace_compacted_history(
    new_history,
    replacement_step_context.to_turn_context_item(),
    world_state_baseline,
    CompactedHistoryMetadata {
        input_goal_ids,
        message: summary_text,
        window_number,
        window_ids,
        compaction_response_id: Some(compaction_response.response_id),
        compaction_model_hash: turn_context.model_info().comp_hash.clone(),
        reviewer_compaction_hash: None,
    },
)
.await;

sess.recompute_token_usage(&replacement_step_context.turn).await;
```

保存的元数据包括：

- 压缩前识别到的 user goal IDs；
- summary message；
- compact window number 和 IDs；
- compaction response ID；
- compaction model hash。

最后重新计算 token usage，并发出 compaction completed 生命周期事件。

---

## 八、压缩后的下一次普通模型请求

假设压缩完成后，用户追加新的请求：

```text
请再确认 README.md 只包含要求的标题和说明，如果正确就告诉我结果。
```

此时普通 turn 不再把所有旧的 function call/output 逐条发送。`clone_history().for_prompt(...)` 得到的历史主要包括：

1. initial context；
2. 仍在压缩预算内保留的用户目标消息；
3. `CompactionSummary` user message；
4. 压缩后新追加的 user message。

压缩后的普通请求伪内容如下：

```json
{
  "model": "gpt-5.2-codex",
  "stream": true,
  "service_tier": "priority",
  "input": [
    {
      "type": "message",
      "id": "msg_base_001",
      "role": "developer",
      "content": [
        {
          "type": "input_text",
          "text": "You are a coding agent in the Codex CLI.\nUse the available tools when needed.\nKeep the requested file scope unchanged."
        }
      ]
    },
    {
      "type": "message",
      "role": "user",
      "content": [
        {
          "type": "input_text",
          "text": "请创建 README.md，内容如下：\n\n# Codex Demo\n\n这是一个使用 Rust 编写的 Agent 项目。"
        }
      ]
    },
    {
      "type": "message",
      "role": "user",
      "content": [
        {
          "type": "input_text",
          "text": "Another language model started to solve this problem and produced a summary of its thinking process. You also have access to the state of the tools that were used by the other language model. Use this information to assist you in continuing the task. Avoid duplicating work already done.\nHere is the summary produced by the other language model, use the information in this summary to assist with your own analysis:\n\n当前任务是创建 README.md。用户要求文件包含标题 # Codex Demo 和中文说明。已完成文件写入，并通过 cat README.md 验证内容正确。用户还要求只修改 README.md。当前没有未完成的文件修改；后续如需继续，应先读取现有 README.md，再遵守文件范围限制。"
        }
      ]
    },
    {
      "type": "message",
      "role": "user",
      "content": [
        {
          "type": "input_text",
          "text": "请再确认 README.md 只包含要求的标题和说明，如果正确就告诉我结果。"
        }
      ]
    }
  ],
  "tools": [
    {
      "type": "function",
      "name": "exec_command",
      "description": "Runs a command in a PTY, returning output or a session ID for ongoing interaction.",
      "strict": false,
      "parameters": {
        "type": "object",
        "properties": {
          "cmd": {
            "type": "string",
            "description": "Shell command to execute."
          },
          "workdir": {
            "type": "string",
            "description": "Working directory for the command. Defaults to the turn cwd."
          },
          "tty": {
            "type": "boolean",
            "description": "True allocates a PTY; false or omitted uses plain pipes."
          },
          "yield_time_ms": {
            "type": "number",
            "description": "Wait before yielding output."
          },
          "max_output_tokens": {
            "type": "number",
            "description": "Output token budget."
          }
        },
        "required": ["cmd"],
        "additionalProperties": false
      }
    }
  ],
  "tool_choice": "auto",
  "parallel_tool_calls": true,
  "reasoning": {
    "effort": "medium"
  },
  "store": false,
  "include": ["reasoning.encrypted_content"],
  "prompt_cache_key": "session_001",
  "client_metadata": {
    "x-codex-installation-id": "install_001",
    "session_id": "session_001",
    "thread_id": "thread_001",
    "x-codex-window-id": "window_001",
    "turn_id": "turn_002"
  }
}
```

这里的 summary 文本是 `SUMMARY_PREFIX + summary_suffix` 的教学化结果。真实文本由当前模型输出决定；不能把上面的句子当成固定源码常量。

这次请求重新走普通采样链：

```text
clone_history().for_prompt(...)
  -> build_prompt(...)
  -> build_responses_request(...)
  -> client_session.stream(...)
```

如果模型认为需要再次读取文件，它会返回 FunctionCall，工具完成后再进入下一次 sampling；如果直接返回 assistant message，并且 end_turn/pending input 条件满足，turn 才结束。

---

## 九、压缩前后到底保留了什么

### 压缩前

```text
initial context
+ user goal
+ 多个 function call
+ 多个 function_call_output
+ steer user message
+ assistant progress
+ 工具验证结果
+ 其他 context/world-state item
```

### 压缩请求

```text
压缩前的临时 history
+ 一条由 prompt.md 生成的 user compaction instruction
```

### 压缩后

```text
重新生成的 initial context
+ token 预算内保留的 user messages
+ CompactionSummary user message
+ 压缩之后的新 user input
```

工具调用历史通常不再逐条保留在压缩后的普通 prompt 中，而是由模型生成的摘要表达已完成动作、结果、约束和下一步。保留哪些 user message 由 `collect_annotated_user_messages` 和 token budget 逻辑决定；这不是“永远只保留最后一条消息”。

---

## 十、失败、重试和 context window exceeded

### 1. 压缩请求失败

`run_compact_task_inner_impl` 对不同失败分类：

- `Interrupted` 或 `TurnAborted`：直接返回；
- `SessionBudgetExceeded`：返回错误；
- `ContextWindowExceeded`：如果临时 history 不止一个 item，从开头移除最旧 item，重新构造压缩请求；
- 其他错误：若 provider 允许，按 `stream_max_retries` 和 backoff 重试。

这意味着 compaction 请求本身也可能多次采样，但每次尝试都从当前临时 history 重新构造 `Prompt`。

### 2. 为什么要从开头移除

源码注释说明，context window exceeded 时从开头移除可以保留较新的消息，并尽量保留 prefix-based cache。移除 history item 时还会调用对应的 tool call/output 配对清理逻辑，避免留下孤立的调用或结果。

### 3. 只有成功才替换 live history

压缩模型还没有完成、摘要无法提取、或压缩请求失败时，正常 session history 不应被半成品摘要覆盖。只有成功完成并得到 compaction response 后，才执行 `replace_compacted_history`。

---

## 十一、记忆实现的准确边界

从固定源码可以确认，Codex 的会话记忆主要由以下结构组成：

- `Session` 内的 history；
- `ResponseItemEnvelope` 中的协议 item 和 harness metadata；
- world-state/context fragments；
- compaction summary 和 replacement history；
- auto-compact window IDs；
- token usage/context window bookkeeping；
- compaction response ID 和 model hash；
- rollout/history persistence metadata。

不能从这些代码推出“模型拥有独立永久记忆”。压缩后的摘要只是新的 user-role history item；下一次模型请求仍然由当前 session history 重新构造。

同样，`SUMMARY_PREFIX` 不是一个隐藏的模型记忆数据库入口，而是一段会被拼进 compaction summary 的文本。模型能否正确恢复任务，取决于摘要质量、保留的用户消息、重新注入的 initial context，以及后续请求中的新输入。

---

## 十二、源码调用链总结

```text
run_turn
  -> run_pre_sampling_compact
  -> context_window_token_status
  -> run_auto_compact
  -> run_inline_auto_compact_task
  -> run_compact_task_inner_impl
      -> clone_history
      -> record synthetic compaction UserInput
      -> history.for_prompt
      -> attach executed tool metadata
      -> choose base instructions
      -> build Prompt
      -> drain_to_completed
          -> client_session.stream
          -> build_responses_request
          -> Responses API
          -> OutputItemDone / Completed
      -> extract assistant summary
      -> SUMMARY_PREFIX + summary_suffix
      -> collect_annotated_user_messages
      -> build_compacted_history
      -> build_compaction_replacement_history
      -> replace_compacted_history
      -> recompute_token_usage
  -> clone_history after replacement
  -> build_prompt
  -> build_responses_request
  -> next ordinary sampling request
```

最重要的源码结论是：

1. 会话记忆是结构化历史，不是单一字符串；
2. 压缩是一次独立的模型 sampling request；
3. 压缩请求使用“旧历史 + 合成 compaction prompt”；
4. 摘要由模型返回 assistant message；
5. 摘要经过 `SUMMARY_PREFIX` 包装为 `CompactionSummary`；
6. `build_compacted_history` 依据用户消息和 token 预算生成新历史；
7. `replace_compacted_history` 原子性地替换 session history；
8. 压缩后的普通请求重新从新历史构造，而不是继续发送旧的完整 transcript。

---

## 十三、面试版回答

### 一分钟回答

Codex 的会话记忆不是一个独立向量库，而是 Session 持有的结构化 ResponseItem history。每次普通 turn 通过 `clone_history().for_prompt(...)` 生成 Prompt，加入 base instructions、工具和当前输入后发出 Responses 请求。历史变长后，`run_pre_sampling_compact` 或 turn 中的 token/context 检查触发 `run_auto_compact`。普通 Responses 路径调用 `run_inline_auto_compact_task`，把 `prompts/templates/compact/prompt.md` 作为合成 user input 追加到历史副本，再由 `drain_to_completed` 发一个独立的 compaction sampling request。模型返回 assistant 摘要后，Codex 用 `SUMMARY_PREFIX + summary` 构造 `CompactionSummary`，保留 token 预算内的 user messages，重新注入 initial context，最后调用 `replace_compacted_history` 替换历史并重算 token。下一次普通请求只发送压缩后的结构化历史和新用户输入。

### 三分钟回答

需要区分普通 sampling 和 compaction sampling。普通 sampling 的结果可能是 FunctionCall，工具执行后用 FunctionCallOutput 回填下一轮；compaction sampling 不暴露普通工具，Prompt 默认 tools 为空，它的目标是生成 handoff summary。压缩流程先复制 history，再记录一条合成的压缩指令，调用 `for_prompt` 规范化，避免重复 base instruction，构造 Prompt 并调用 `client_session.stream`。`drain_to_completed` 收集 output items 并等待 Completed。成功后源码从 assistant message 提取 summary，拼接 `summary_prefix.md`，调用 `build_compacted_history`，从 user messages 尾部按 token 预算选择要保留的内容，再追加 `CompactionSummary` user fragment。随后 `build_compaction_replacement_history` 把新的 initial context/world-state 放回正确位置，`replace_compacted_history` 保存 compaction metadata 并替换 live history。压缩后的下一次请求继续走 `clone_history -> build_prompt -> build_responses_request -> stream`，所以模型看到的是摘要和保留的关键输入，而不是旧的完整工具 transcript。

---

## 十四、固定 commit 源码索引

- [`session/turn.rs`](https://github.com/openai/codex/blob/2351d9e1b608e6f9d9a3699b71d7eb39ee41cfa4/codex-rs/core/src/session/turn.rs)
- [`compact.rs`](https://github.com/openai/codex/blob/2351d9e1b608e6f9d9a3699b71d7eb39ee41cfa4/codex-rs/core/src/compact.rs)
- [`context_manager/history.rs`](https://github.com/openai/codex/blob/2351d9e1b608e6f9d9a3699b71d7eb39ee41cfa4/codex-rs/core/src/context_manager/history.rs)
- [`context_manager/normalize.rs`](https://github.com/openai/codex/blob/2351d9e1b608e6f9d9a3699b71d7eb39ee41cfa4/codex-rs/core/src/context_manager/normalize.rs)
- [`prompts/src/compact.rs`](https://github.com/openai/codex/blob/2351d9e1b608e6f9d9a3699b71d7eb39ee41cfa4/codex-rs/prompts/src/compact.rs)
- [`prompts/templates/compact/prompt.md`](https://github.com/openai/codex/blob/2351d9e1b608e6f9d9a3699b71d7eb39ee41cfa4/codex-rs/prompts/templates/compact/prompt.md)
- [`prompts/templates/compact/summary_prefix.md`](https://github.com/openai/codex/blob/2351d9e1b608e6f9d9a3699b71d7eb39ee41cfa4/codex-rs/prompts/templates/compact/summary_prefix.md)
- [`client_common.rs`](https://github.com/openai/codex/blob/2351d9e1b608e6f9d9a3699b71d7eb39ee41cfa4/codex-rs/core/src/client_common.rs)
- [`client.rs::build_responses_request`](https://github.com/openai/codex/blob/2351d9e1b608e6f9d9a3699b71d7eb39ee41cfa4/codex-rs/core/src/client.rs#L948-L1070)
- [`codex-api/common.rs`](https://github.com/openai/codex/blob/2351d9e1b608e6f9d9a3699b71d7eb39ee41cfa4/codex-rs/codex-api/src/common.rs#L279-L370)
- [`protocol/models.rs`](https://github.com/openai/codex/blob/2351d9e1b608e6f9d9a3699b71d7eb39ee41cfa4/codex-rs/protocol/src/models.rs)
