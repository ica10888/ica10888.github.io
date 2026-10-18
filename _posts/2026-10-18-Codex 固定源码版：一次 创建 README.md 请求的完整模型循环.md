---
layout:     post
title:      Codex 固定源码版：一次创建 README.md 请求的完整模型循环
subtitle:  Codex 固定源码版：一次 创建 README.md 请求的完整模型循环
date:       2026-10-18
author:     ica10888
catalog: true
tags:
    - AI 
    - Prompts engineering
---

# Codex 固定源码版：一次“创建 README.md”请求的完整模型循环

## 研究范围

本文重新从本次会话已经核对过的本地 Git 源码推导，不采用网上文章、搜索摘要，也不读取上一版文章。研究对象是 openai/codex 的 Rust 实现，固定 commit 为：

2351d9e1b608e6f9d9a3699b71d7eb39ee41cfa4

调研日期：2026-10-17。源码链接全部固定到这个 commit。本文讨论的是该 commit 中 CLI/core 的实现，不能把它无条件推广成所有 Codex 产品的内部实现。

本文贯穿一个教学案例：

> 用户：请创建 README.md，内容如下：  
> # Codex Demo  
> 这是一个使用 Rust 编写的 Agent 项目。

为了把请求逐步写完整，使用以下可复现 fixture：

- 普通 Responses HTTP 传输；
- 不启用 Responses Lite、WebSocket 增量、Guardian、自动压缩、重试和 warmup；
- Linux/bash，当前工作目录为 /workspace/demo；
- base instruction 使用短的自定义文本，仅用于可复现展示；它不是声称默认安装一定使用的完整 bundled prompt；
- 只暴露 exec_command，并将本例的 exec_permission_approvals_enabled fixture 设为 true，因此 schema 中展示 sandbox_permissions、justification、prefix_rule；真实配置关闭该开关时这些字段不会加入；
- 工具执行结果中的 chunk id、时间和 UUID 是教学伪值；
- 用户在第一次工具完成后追加一条 steer，要求不得覆盖其他文件。

“完整请求”必须先分层。Codex 中至少有四个层次：

1. Session 历史和 Prompt：结构化的 ResponseItem 序列，不是 HTTP JSON。
2. ResponsesApiRequest：core/client.rs 将 Prompt 转换出的请求对象。
3. wire JSON：HTTP body，或 WebSocket 的 response.create 包装体。
4. 模型响应的 ResponseItem：协议层解析完成后，才由 ToolRouter 构造 runtime ToolCall。

主要源码入口：

- run_turn、run_sampling_request、try_run_sampling_request：
  https://github.com/openai/codex/blob/2351d9e1b608e6f9d9a3699b71d7eb39ee41cfa4/codex-rs/core/src/session/turn.rs
- Prompt：
  https://github.com/openai/codex/blob/2351d9e1b608e6f9d9a3699b71d7eb39ee41cfa4/codex-rs/core/src/client_common.rs#L23-L116
- build_responses_request：
  https://github.com/openai/codex/blob/2351d9e1b608e6f9d9a3699b71d7eb39ee41cfa4/codex-rs/core/src/client.rs#L948-L1070
- ResponsesApiRequest：
  https://github.com/openai/codex/blob/2351d9e1b608e6f9d9a3699b71d7eb39ee41cfa4/codex-rs/codex-api/src/common.rs#L279-L304

## 一、一次 turn 并不等于一次请求

turn.rs 在 run_turn 旁边的注释用简化语言描述了循环：一次 sampling 可以返回 function call 或 assistant message；若是 function call，执行工具并在下一次 sampling request 发送 output；若只返回 assistant message，通常进入结束判定。真实结束还要看 Completed/end_turn、pending input 和 in-flight 工具。

所以：

- session/thread 是更长生命周期；
- turn 是一次用户目标的编排边界；
- 一次 sampling request 是一次模型采样；
- 一次 tool call 是模型在一次采样中提出的动作；
- 一次本地 handler 执行不是模型请求。

抽象控制流是：

~~~text
run_turn
  -> 捕获 step context、工具、环境和策略
  -> 记录初始 world state、用户输入、注入项
  -> clone_history().for_prompt(...)
  -> build_prompt(...)
  -> ModelClientSession::stream(...)
  -> 处理 response events
  -> 若有 FunctionCall，build_tool_call
  -> ToolInvocation -> Registry -> Handler
  -> FunctionCallOutput 写入历史
  -> 下一轮 sampling
  -> assistant message 后进入 end_turn/pending-input 判定
~~~

run_turn 开始阶段的前置 compact、MCP 启动、hooks、world-state 捕获都可能失败；这些失败并不产生一次模型请求。本文 fixture 明确假设环境/权限/AGENTS/skills/plugins/extension 等 world-state contributor 均未产生可见片段，并仅保留与主链相关的历史项用于展开；真实配置若启用这些 contributor，它们会追加到 input，下面的 JSON 就应相应增加这些 ResponseItem。

真实配置并不一定只有 base 和用户消息。world state 可能产生：

- developer 指令；
- AGENTS.md 指令，角色由源码实现为 user；
- 权限说明；
- environment XML；
- apps、plugins、deferred tool namespace；
- skill catalog 和选中的 skill；
- 协作、多 agent、token budget、模型切换说明；
- extension contributor fragments。

这些内容先由 build_initial_context_with_world_state 产生结构化 item，再持久化到 history。代码见：

https://github.com/openai/codex/blob/2351d9e1b608e6f9d9a3699b71d7eb39ee41cfa4/codex-rs/core/src/session/world_state.rs#L58-L410

https://github.com/openai/codex/blob/2351d9e1b608e6f9d9a3699b71d7eb39ee41cfa4/codex-rs/core/src/session/mod.rs#L4317-L4592

## 二、用户输入、历史和 Prompt

### 1. 用户输入先变成 ResponseItem

protocol/src/models.rs 的 ResponseInputItem::from_user_input 对每个 UserInput 分支处理：

- Text 生成 ContentItem::InputText；
- Image 生成 InputImage；
- Audio 生成 InputAudio；
- Skill 和 Mention 产生空 content，正文由 core 后续注入。

外层消息角色是 user。源码：

https://github.com/openai/codex/blob/2351d9e1b608e6f9d9a3699b71d7eb39ee41cfa4/codex-rs/protocol/src/models.rs#L2022-L2110

本例当前用户消息的结构化 item 是：

~~~json
{
  "type": "message",
  "role": "user",
  "content": [
    {
      "type": "input_text",
      "text": "请创建 README.md，内容如下：\n\n# Codex Demo\n\n这是一个使用 Rust 编写的 Agent 项目。"
    }
  ]
}
~~~

record_user_prompt_and_emit_turn_item 会把用户消息保存进会话历史。随后 clone_history().for_prompt(input_modalities) 复制并规范化历史：

- 删除当前模型不支持的图片/音频；
- 保留 assistant、reasoning、tool call、tool output 等结构化 item；
- 保持工具调用与输出的配对关系；
- 不把所有 UI 生命周期事件、审批对象和底层 sandbox 对象原样塞入模型输入。

### 2. Prompt 的字段

client_common.rs 中的 Prompt 可概括为：

~~~rust
Prompt {
    input: Vec<ResponseItem>,
    tools: Arc<[ToolSpec]>,
    parallel_tool_calls: bool,
    base_instructions: BaseInstructions,
    output_schema: Option<Value>,
    output_schema_strict: bool,
    cyber_access_program: Option<CyberAccessProgram>,
}
~~~

在上述“无额外 world-state fragment”的教学 fixture 下，内部 Prompt 的完整伪内容如下。provenance 是内部持久化信息，不会作为 wire 字段发送。

~~~json
{
  "input": [
    {
      "type": "message",
      "role": "user",
      "content": [
        {
          "type": "input_text",
          "text": "请创建 README.md，内容如下：\n\n# Codex Demo\n\n这是一个使用 Rust 编写的 Agent 项目。"
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
            "description": "True allocates a PTY for the command; false or omitted uses plain pipes."
          },
          "yield_time_ms": {
            "type": "number",
            "description": "Wait before yielding output. Defaults to 10000 ms; effective range is 250-30000 ms."
          },
          "max_output_tokens": {
            "type": "number",
            "description": "Output token budget. Defaults to 10000 tokens; larger requests may be capped by policy."
          },
          "shell": {
            "type": "string",
            "description": "Shell binary to launch. Defaults to the user's default shell."
          },
          "sandbox_permissions": {
            "type": "string",
            "enum": ["use_default", "require_escalated"],
            "description": "Per-command sandbox override. Defaults to `use_default`; use `require_escalated` for unsandboxed execution."
          },
          "justification": {
            "type": "string",
            "description": "User-facing approval question for require_escalated; omit otherwise."
          },
          "prefix_rule": {
            "type": "array",
            "items": {
              "type": "string"
            },
            "description": "Reusable approval prefix for cmd, only with sandbox_permissions require_escalated."
          }
        },
        "required": ["cmd"],
        "additionalProperties": false
      }
    }
  ],
  "parallel_tool_calls": true,
  "base_instructions": {
    "text": "You are a coding agent in the Codex CLI.\nUse the available tools when they are needed.\nRespect the active sandbox and approval policy.",
    "provenance": "custom"
  },
  "output_schema": null,
  "output_schema_strict": true,
  "cyber_access_program": null
}
~~~

build_prompt 在 incremental_tools 为 true 时会把 tools 清空、base text 清空，转而由 world state 注入；本例 incremental_tools 为 false，所以工具和 base instruction 保留在 Prompt。

### 3. base instructions 的真实映射

一个常见误解是把 Prompt.base_instructions 直接写成顶层 instructions 字段。固定 commit 的实际实现不是这样：

1. build_responses_request 复制并格式化 Prompt.input；
2. 如果 base text 非空，构造 BaseInstructionsFragment；
3. BaseInstructionsFragment 的 role() 返回 developer；
4. ContextualUserFragment::into 生成 ResponseItem::Message；
5. 这个 developer item 被 splice 到 input 开头。

源码：

https://github.com/openai/codex/blob/2351d9e1b608e6f9d9a3699b71d7eb39ee41cfa4/codex-rs/core/src/context/base_instructions.rs#L26-L46

https://github.com/openai/codex/blob/2351d9e1b608e6f9d9a3699b71d7eb39ee41cfa4/codex-rs/context-fragments/src/fragment.rs#L35-L51

因此内部 provenance 不出现在 wire；模型看到的是 developer message。

## 三、工具 schema 如何约束 arguments

exec_command 的 schema 在 core/src/tools/handlers/shell_spec.rs 中构造。固定字段包括 cmd、workdir、tty、yield_time_ms、max_output_tokens、shell。审批能力还会加入 sandbox_permissions、justification、prefix_rule；启用额外权限时可能加入 additional_permissions。

这段实现刻意把 strict 设为 false。它表示请求仍然给模型明确的参数形状，但不能把 strict false 理解为 runtime 验证。工具定义的核心逻辑是：

~~~rust
ResponsesApiTool {
    name: "exec_command",
    description: "Runs a command in a PTY, returning output or a session ID for an ongoing command.",
    strict: false,
    parameters: JsonSchema::object(
        properties,
        Some(vec!["cmd"]),
        Some(false.into()),
    ),
    output_schema: Some(unified_exec_output_schema()),
}
~~~

ResponsesApiTool.output_schema 被 serde skip，因此不出现在顶层 tools 的 function definition 中。源码：

https://github.com/openai/codex/blob/2351d9e1b608e6f9d9a3699b71d7eb39ee41cfa4/codex-rs/tools/src/responses_api.rs#L33-L46

### arguments 的两层格式

模型返回的 FunctionCall.arguments 是字符串，字符串内部才是 JSON。以读取命令为例：

外层 ResponseItem：

~~~json
{
  "type": "function_call",
  "name": "exec_command",
  "arguments": "{\"cmd\":\"printf 'README absent\\\\n'\",\"yield_time_ms\":1000}",
  "call_id": "call_001",
  "id": "fc_001"
}
~~~

内层参数对象：

~~~json
{
  "cmd": "printf 'README absent\\n'",
  "yield_time_ms": 1000
}
~~~

在 registry/handler 处，parse_arguments<T> 使用 serde_json::from_str<T> 解码成 ExecCommandArgs。JSON Schema 负责让模型知道字段、类型、必填项和禁止额外属性；serde 负责本地 Rust 解码；sandbox、审批和语义检查仍由 runtime 完成。若解码失败，错误以 FunctionCallError::RespondToModel 形式成为下一轮可见的工具错误。

## 四、第一次真实 sampling：先检查 README 是否存在

### 1. 准备阶段不是请求

run_turn 先完成以下本地步骤：

- capture step context；
- 记录初始 world state；
- 记录用户输入；
- 记录 skills/plugins injection（本 fixture 为空）；
- clone_history；
- build_prompt。

直到 try_run_sampling_request 中的 client_session.stream 才发送网络请求。不能给每一个 Rust 函数虚构一份 HTTP body。

### 2. R1 完整 ResponsesApiRequest

在上述教学 fixture 下，本例的第一份实际 wire 请求伪 JSON 如下。所有工具属性完整写出；没有使用“同上”或省略号。

~~~json
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
          "text": "You are a coding agent in the Codex CLI.\nUse the available tools when they are needed.\nRespect the active sandbox and approval policy."
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
            "description": "True allocates a PTY for the command; false or omitted uses plain pipes."
          },
          "yield_time_ms": {
            "type": "number",
            "description": "Wait before yielding output. Defaults to 10000 ms; effective range is 250-30000 ms."
          },
          "max_output_tokens": {
            "type": "number",
            "description": "Output token budget. Defaults to 10000 tokens; larger requests may be capped by policy."
          },
          "shell": {
            "type": "string",
            "description": "Shell binary to launch. Defaults to the user's default shell."
          },
          "sandbox_permissions": {
            "type": "string",
            "enum": ["use_default", "require_escalated"],
            "description": "Per-command sandbox override. Defaults to `use_default`; use `require_escalated` for unsandboxed execution."
          },
          "justification": {
            "type": "string",
            "description": "User-facing approval question for require_escalated; omit otherwise."
          },
          "prefix_rule": {
            "type": "array",
            "items": {
              "type": "string"
            },
            "description": "Reusable approval prefix for cmd, only with sandbox_permissions require_escalated."
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
~~~

字段依据：

- model 和 service_tier：模型路由；本文的 gpt-5.2-codex、priority 是教学 fixture，实际值由当前 ModelInfo 和配置解析；
- input：developer base item 加历史 user item；
- tools：create_tools_raw_json_for_responses_api；
- tool_choice 固定为 auto；
- parallel_tool_calls：Prompt 的值且非 Lite；
- reasoning：build_reasoning；普通模式省略 context。本文的 medium 是 fixture，实际 effort 由用户设置或模型默认值解析；summary 也是可选项，本 fixture 将其设为 None。
- stream：固定为 true；
- include：固定加入 reasoning.encrypted_content；
- store：固定为 false；
- prompt_cache_key：Codex 的缓存亲和键；
- client_metadata：Responses metadata 的兼容投影，不能误写为自然语言 prompt。本文只展示最小 fixture 的 installation/session/thread/window/turn 字段；真实 metadata 是否包含 turn metadata、父线程、root turn 等取决于当前请求和 include_internal。

本 fixture 使用 `CodexResponsesMetadata::new` 的最小字段并把 `request_kind` 留空，所以没有人为编造 `x-codex-turn-metadata` 大 JSON。真实 turn 若设置了 request_kind，会按 `responses_metadata.rs` 再加入该键；它仍是服务端元数据，不是 `input` 中的模型对话。

ResponsesApiRequest 的 Option 字段在 None 时 skip_serializing；因此本例没有伪造 null 的 text、stream_options、access_programs。

### 3. R1 返回的完整结构化 item

模型的 response stream 可能先返回 reasoning delta。真正交给 core 的完整工具 item 是：

~~~json
{
  "type": "function_call",
  "id": "fc_001",
  "name": "exec_command",
  "arguments": "{\"cmd\":\"printf 'README absent\\\\n'\",\"yield_time_ms\":1000}",
  "call_id": "call_001"
}
~~~

SSE 解析路径是 codex-api/src/sse/responses.rs：

- response.output_item.done.item 直接 serde_json::from_value::<ResponseItem>；
- response.custom_tool_call_input.delta 才产生 ToolCallInputDelta；
- response.function_call_arguments.delta 和 .done 在本 commit 的 unhandled event 分支中只 trace，不由 core 累积普通 function arguments。

因此不要把本版本写成“每个普通 function arguments delta 都由 turn.rs 拼接”。普通 FunctionCall 的完整字符串来自 output_item.done。

### 4. ResponseItem 到 runtime ToolCall 的唯一核心位置

ToolRouter::build_tool_call 位于：

https://github.com/openai/codex/blob/2351d9e1b608e6f9d9a3699b71d7eb39ee41cfa4/codex-rs/core/src/tools/router.rs#L256-L309

它做三种转换：

1. FunctionCall：
   - ToolName::new(namespace, name).with_default_namespace();
   - payload = ToolPayload::Function { arguments };
   - 保留 encrypted_function_args 和 call_id。
2. ToolSearchCall：
   - 只接受 call_id 存在并且 execution == client；
   - serde_json::from_value 解析 SearchToolCallParams；
   - payload = ToolPayload::ToolSearch。
3. CustomToolCall：
   - namespace 和 name 规范化；
   - input 放进 ToolPayload::Custom；
   - 不把 input 当 function arguments JSON。

其他 ResponseItem（assistant message、reasoning、tool output）返回 None。

本例 R1 的内部对象是：

~~~rust
ToolCall {
    tool_name: ToolName::plain("exec_command"),
    call_id: "call_001",
    payload: ToolPayload::Function {
        arguments: "{\"cmd\":\"printf 'README absent\\\\n'\",\"yield_time_ms\":1000}"
    },
    encrypted_function_args: None
}
~~~

这里的 ToolCall 是 core/src/tools/router.rs 中的四字段 runtime 类型。codex-rs/tools/src/tool_call.rs 还有一个同名扩展回调结构，带 turn_id、model、history、environment 等字段；不要把两个类型混为一谈。

## 五、ToolCall 的本地执行阶段：没有新增模型请求

dispatch_tool_call_with_state 解构 ToolCall，再创建 ToolInvocation：

~~~rust
ToolInvocation {
    session,
    turn,
    step_context,
    cancellation_token,
    tracker,
    call_id,
    tool_name,
    source,
    payload,
}
~~~

然后 registry.dispatch_any_with_state：

1. 根据 tool_name 查找注册 runtime；
2. 检查 payload kind；
3. 执行 pre-tool-use hooks；
4. 观察 cancellation token；
5. handler 解码 arguments；
6. 执行 unified_exec；
7. 执行 post-tool-use hooks；
8. 生成 ToolOutput。

源码：

https://github.com/openai/codex/blob/2351d9e1b608e6f9d9a3699b71d7eb39ee41cfa4/codex-rs/core/src/tools/router.rs#L313-L393

https://github.com/openai/codex/blob/2351d9e1b608e6f9d9a3699b71d7eb39ee41cfa4/codex-rs/core/src/tools/registry.rs

本例第一次命令是 printf 'README absent\n'，输出伪内容：

~~~text
Chunk ID: chunk_001
Wall time: 0.0030 seconds
Process exited with code 0
Output:
README absent
~~~

ExecCommandToolOutput::to_response_item 通过 function_tool_response 创建 FunctionCallOutput。单一 InputText 会成为 output 字符串；多 content 时才是 content 数组。response_header/response_text 的条件字段和截断预算见 https://github.com/openai/codex/blob/2351d9e1b608e6f9d9a3699b71d7eb39ee41cfa4/codex-rs/core/src/tools/context.rs#L510-L570。Code Mode 的 JSON 结果字段是另一条路径，不能拿来代表 direct exec 的模型输出。

## 六、第二次真实 sampling：工具结果和 steer

### 1. steer 的消费时机

用户在 R1 工具完成后追加：

补充限制：README 必须保留上面的标题和中文说明，不要覆盖其他文件。

run_turn 下一轮从 sess.input_queue.get_pending_input(&sess.active_turn) 取 pending input，然后用 PersistContext::SteeredUserInput 记录它。这个动作发生在下一次 sampling request 之前。

steer：

- 不修改已经发给模型的 R1 arguments；
- 不修改已经启动的进程；
- 不直接调用 handler；
- 会成为下一次 clone_history().for_prompt(...) 的 user ResponseItem；
- 若 steer 引用了新插件/MCP，step context 可能重新捕获工具和世界状态。

本例假设工具 output 已经 drain 并记录后，用户才提交 steer，因此历史此时按顺序是：原始 user、R1 FunctionCall、R1 FunctionCallOutput、steer user。若 steer 在工具或流阶段更早排队，实际顺序以各次 record 时序为准。

### 2. R2 完整逻辑请求

~~~json
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
          "text": "You are a coding agent in the Codex CLI.\nUse the available tools when they are needed.\nRespect the active sandbox and approval policy."
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
          "arguments": "{\"cmd\":\"printf 'README absent\\\\n'\",\"yield_time_ms\":1000}",
      "call_id": "call_001"
    },
    {
      "type": "function_call_output",
      "call_id": "call_001",
      "output": "Chunk ID: chunk_001\nWall time: 0.0030 seconds\nProcess exited with code 0\nOutput:\nREADME absent"
    },
    {
      "type": "message",
      "role": "user",
      "content": [
        {
          "type": "input_text",
          "text": "补充限制：README 必须保留上面的标题和中文说明，不要覆盖其他文件。"
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
            "description": "True allocates a PTY for the command; false or omitted uses plain pipes."
          },
          "yield_time_ms": {
            "type": "number",
            "description": "Wait before yielding output. Defaults to 10000 ms; effective range is 250-30000 ms."
          },
          "max_output_tokens": {
            "type": "number",
            "description": "Output token budget. Defaults to 10000 tokens; larger requests may be capped by policy."
          },
          "shell": {
            "type": "string",
            "description": "Shell binary to launch. Defaults to the user's default shell."
          },
          "sandbox_permissions": {
            "type": "string",
            "enum": ["use_default", "require_escalated"],
            "description": "Per-command sandbox override. Defaults to `use_default`; use `require_escalated` for unsandboxed execution."
          },
          "justification": {
            "type": "string",
            "description": "User-facing approval question for require_escalated; omit otherwise."
          },
          "prefix_rule": {
            "type": "array",
            "items": {
              "type": "string"
            },
            "description": "Reusable approval prefix for cmd, only with sandbox_permissions require_escalated."
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
~~~

R2 的历史中，FunctionCall 的 arguments 仍然是字符串；FunctionCallOutput 的 output 是 wire string。历史中的 call_001 与新调用的 call_002 分别配对，不能把不同调用共用一个 call_id。

R2 模型返回写入命令：

~~~json
{
  "type": "function_call",
  "id": "fc_002",
  "name": "exec_command",
  "arguments": "{\"cmd\":\"cat > README.md <<'EOF'\\n# Codex Demo\\n\\n这是一个使用 Rust 编写的 Agent 项目。\\nEOF\",\"yield_time_ms\":1000}",
  "call_id": "call_002"
}
~~~

Codex 再次本地执行，产生：

~~~text
Chunk ID: chunk_002
Wall time: 0.0210 seconds
Process exited with code 0
Process exited with code 0
Output:
~~~

这里仍然没有新的模型请求。只有在 FunctionCallOutput 被 drain_in_flight、记录进历史后，下一次 sampling 才会把它发送给模型。

## 七、第三次真实 sampling：读取并验证

R2 写入成功后，模型仍可以先调用读取工具。R3 的完整请求必须包含到目前为止的全部历史和完整工具定义：

~~~json
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
          "text": "You are a coding agent in the Codex CLI.\nUse the available tools when they are needed.\nRespect the active sandbox and approval policy."
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
          "arguments": "{\"cmd\":\"printf 'README absent\\\\n'\",\"yield_time_ms\":1000}",
      "call_id": "call_001"
    },
    {
      "type": "function_call_output",
      "call_id": "call_001",
      "output": "Chunk ID: chunk_001\nWall time: 0.0030 seconds\nProcess exited with code 0\nOutput:\nREADME absent"
    },
    {
      "type": "message",
      "role": "user",
      "content": [
        {
          "type": "input_text",
          "text": "补充限制：README 必须保留上面的标题和中文说明，不要覆盖其他文件。"
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
      "output": "Chunk ID: chunk_002\nWall time: 0.0210 seconds\nProcess exited with code 0\nOutput:"
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
            "description": "True allocates a PTY for the command; false or omitted uses plain pipes."
          },
          "yield_time_ms": {
            "type": "number",
            "description": "Wait before yielding output. Defaults to 10000 ms; effective range is 250-30000 ms."
          },
          "max_output_tokens": {
            "type": "number",
            "description": "Output token budget. Defaults to 10000 tokens; larger requests may be capped by policy."
          },
          "sandbox_permissions": {
            "type": "string",
            "enum": ["use_default", "require_escalated"],
            "description": "Per-command sandbox override. Defaults to `use_default`; use `require_escalated` for unsandboxed execution."
          },
          "justification": {
            "type": "string",
            "description": "User-facing approval question for require_escalated; omit otherwise."
          },
          "prefix_rule": {
            "type": "array",
            "items": {
              "type": "string"
            },
            "description": "Reusable approval prefix for cmd, only with sandbox_permissions require_escalated."
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
~~~

R3 返回：

~~~json
{
  "type": "function_call",
  "id": "fc_003",
  "name": "exec_command",
  "arguments": "{\"cmd\":\"cat README.md\",\"yield_time_ms\":1000}",
  "call_id": "call_003"
}
~~~

本地结果：

~~~text
Chunk ID: chunk_003
Wall time: 0.0040 seconds
Process exited with code 0
Process exited with code 0
Process exited with code 0
Output:
# Codex Demo

这是一个使用 Rust 编写的 Agent 项目。
~~~

注意输出头的实际字段来自 response_header；不同 handler 和版本可能有 Wall time 或 Process exited with code。这里的时间和头部是教学伪值，关键是 output 通过 call_id 进入下一轮。

## 八、第四次真实 sampling：assistant message 结束 turn

R3 的 FunctionCall 和 FunctionCallOutput 记录完成后，R4 发送完整 input。为了避免把同一个工具 schema 复制四次而失去可读性，下面仍完整列出 R4 input、tools、控制字段；没有使用“与上次相同”或省略号。

~~~json
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
          "text": "You are a coding agent in the Codex CLI.\nUse the available tools when they are needed.\nRespect the active sandbox and approval policy."
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
          "arguments": "{\"cmd\":\"printf 'README absent\\\\n'\",\"yield_time_ms\":1000}",
      "call_id": "call_001"
    },
    {
      "type": "function_call_output",
      "call_id": "call_001",
      "output": "Chunk ID: chunk_001\nWall time: 0.0030 seconds\nProcess exited with code 0\nOutput:\nREADME absent"
    },
    {
      "type": "message",
      "role": "user",
      "content": [
        {
          "type": "input_text",
          "text": "补充限制：README 必须保留上面的标题和中文说明，不要覆盖其他文件。"
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
      "output": "Chunk ID: chunk_002\nWall time: 0.0210 seconds\nProcess exited with code 0\nOutput:"
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
      "output": "Chunk ID: chunk_003\nWall time: 0.0040 seconds\nProcess exited with code 0\nOutput:\n# Codex Demo\n\n这是一个使用 Rust 编写的 Agent 项目。"
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
            "description": "True allocates a PTY for the command; false or omitted uses plain pipes."
          },
          "yield_time_ms": {
            "type": "number",
            "description": "Wait before yielding output. Defaults to 10000 ms; effective range is 250-30000 ms."
          },
          "max_output_tokens": {
            "type": "number",
            "description": "Output token budget. Defaults to 10000 tokens; larger requests may be capped by policy."
          },
          "shell": {
            "type": "string",
            "description": "Shell binary to launch. Defaults to the user's default shell."
          },
          "sandbox_permissions": {
            "type": "string",
            "enum": ["use_default", "require_escalated"],
            "description": "Per-command sandbox override. Defaults to `use_default`; use `require_escalated` for unsandboxed execution."
          },
          "justification": {
            "type": "string",
            "description": "User-facing approval question for require_escalated."
          },
          "prefix_rule": {
            "type": "array",
            "items": {
              "type": "string"
            },
            "description": "Reusable approval prefix for cmd, only with sandbox_permissions require_escalated."
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
~~~

R4 的 assistant response：

~~~json
{
  "type": "message",
  "role": "assistant",
  "content": [
    {
      "type": "output_text",
      "text": "README.md 已创建并完成验证，内容符合你给出的标题和中文说明。"
    }
  ]
}
~~~

在本 fixture 中，本次只有 assistant message，且没有 pending input、未完成工具，并收到表示结束的 Completed 状态，因此 turn 结束；不会再发送“确认”请求。源码仍会根据 end_turn、pending input 和 in-flight futures 决定真实结束条件。

## 九、ToolOutput 的真实 wire 形状与 call_id 生命周期

工具输出不是普通 assistant 文本。FunctionCallOutputPayload 的注释和序列化实现表明，wire 上 output 可以是：

- plain string；
- structured content item array。

本例使用单一 input_text，所以是字符串：

~~~json
{
  "type": "function_call_output",
  "call_id": "call_002",
  "output": "Chunk ID: chunk_002\nWall time: 0.0210 seconds\nProcess exited with code\nOutput:"
}
~~~

生命周期：

~~~text
模型 ResponseItem::FunctionCall(call_id=call_002)
  -> build_tool_call 保留 call_id
  -> ToolInvocation.call_id=call_002
  -> handler output.to_response_item(call_id=call_002)
  -> history 写入 FunctionCallOutput
  -> 下一次 ResponsesApiRequest.input 使用同一个 call_id
~~~

多工具并行时，每一个 call_id 都要独立配对。工具失败、取消和参数解析错误也应转换为模型可见 output；不能只写 UI 日志而丢失协议配对。

## 十、Markdown 模板和 active Rust 描述的边界

仓库中存在两个 Markdown 文件：

1. core/templates/search_tool/tool_description.md
2. core/templates/search_tool/request_plugin_install_description.md

它们分别包含 app_descriptions 和 discoverable_tools 占位符。固定 commit 的源码检索没有发现这两个**具体文件名**的 active include/use；这不能证明所有 Markdown 资源都不参与其他路径，也不能把它们写成“每次 README 请求一定进入 Prompt”。

当前活跃 tool-search 描述由 Rust 运行时生成，位置：

https://github.com/openai/codex/blob/2351d9e1b608e6f9d9a3699b71d7eb39ee41cfa4/codex-rs/core/src/tools/handlers/tool_search_spec.rs#L16-L105

其逻辑包含：

~~~text
# Tool discovery

Searches over deferred tool metadata with BM25 and exposes matching tools for the next model call.

You have access to tools from the following sources:
- source-name: source-description

Some of the tools may not have been provided to you upfront, and you should use this tool (tool_search) to search for the required tools.
~~~

request_plugin_install 的 active description 由：

https://github.com/openai/codex/blob/2351d9e1b608e6f9d9a3699b71d7eb39ee41cfa4/codex-rs/core/src/tools/handlers/request_plugin_install_spec.rs#L10-L80

按 presentation 选择独立的 Rust 静态字符串和 schema；它不是对上面 Markdown 占位符做插值。两类描述都只影响模型何时选择工具，不负责把自然语言解析为 ToolCall。

正确链路是：

~~~text
基础指令、world-state fragment、active tool description、JSON Schema
  -> 模型可见 input/tools
  -> 模型协议输出 ResponseItem
  -> build_tool_call
  -> runtime ToolCall
~~~

而不是：

~~~text
Markdown -> 正则 -> ToolCall
~~~

## 十一、普通 Responses 与 Responses Lite

主例是普通 Responses。Lite 模式有三个关键区别：

1. build_responses_request 将工具转成 create_tools_json_for_responses_lite；
2. tools 不在顶层，而是 input 前缀中的 AdditionalTools item；
3. parallel_tool_calls=false，reasoning.context=all_turns。

Lite 第一次逻辑请求可完整写成（其中 namespace 形状只是示意，具体由当前 ToolSpec 和配置决定）：

~~~json
{
  "model": "gpt-5.2-codex",
  "stream": true,
  "input": [
    {
      "type": "additional_tools",
      "id": "at_001",
      "role": "developer",
      "tools": [
        {
          "type": "namespace",
          "name": "functions",
          "description": "",
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
                    "description": "True allocates a PTY for the command; false or omitted uses plain pipes."
                  },
                  "yield_time_ms": {
                    "type": "number",
                    "description": "Wait before yielding output. Defaults to 10000 ms; effective range is 250-30000 ms."
                  },
                  "max_output_tokens": {
                    "type": "number",
                    "description": "Output token budget. Defaults to 10000 tokens; larger requests may be capped by policy."
                  },
                  "shell": {
                    "type": "string",
                    "description": "Shell binary to launch. Defaults to the user’s default shell."
                  },
                  "sandbox_permissions": {
                    "type": "string",
                    "enum": ["use_default", "require_escalated"],
                    "description": "Per-command sandbox override. Defaults to `use_default`; use `require_escalated` for unsandboxed execution."
                  },
                  "justification": {
                    "type": "string",
                    "description": "User-facing approval question for require_escalated; omit otherwise."
                  },
                  "prefix_rule": {
                    "type": "array",
                    "items": {
                      "type": "string"
                    },
                    "description": "Reusable approval prefix for cmd, only with sandbox_permissions require_escalated."
                  }
                },
                "required": ["cmd"],
                "additionalProperties": false
              }
            }
          ]
        }
      ]
    },
    {
      "type": "message",
      "id": "msg_base_001",
      "role": "developer",
      "content": [
        {
          "type": "input_text",
          "text": "You are a coding agent in the Codex CLI.\nUse the available tools when they are needed.\nRespect the active sandbox and approval policy."
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
    }
  ],
  "tool_choice": "auto",
  "parallel_tool_calls": false,
  "reasoning": {
    "effort": "medium",
    "context": "all_turns"
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
~~~

这是 Lite 的一条完整分支伪请求。它说明工具注入位置变化，但 ToolCall 构造仍然是 ResponseItem 到 build_tool_call。

## 十二、WebSocket continuation 的完整边界

WebSocket 使用 ResponsesWsRequest 的 response.create tag。第一次请求可以完整携带 input；收到 response_id 后，同一 turn 的下一次请求可能携带 previous_response_id，并只传 input 尾差分。

一个在 prepare_websocket_request 确认前缀匹配、存在 completed response 时才成立的 continuation wire body 教学例子：

~~~json
{
  "type": "response.create",
  "model": "gpt-5.2-codex",
  "stream": true,
  "previous_response_id": "resp_001",
  "input": [
    {
      "type": "function_call_output",
      "call_id": "call_001",
      "output": "Chunk ID: chunk_001\nWall time: 0.0030 seconds\nProcess exited with code\nOutput:\nREADME absent"
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
            "description": "True allocates a PTY for the command; false or omitted uses plain pipes."
          },
          "yield_time_ms": {
            "type": "number",
            "description": "Wait before yielding output. Defaults to 10000 ms; effective range is 250-30000 ms."
          },
          "max_output_tokens": {
            "type": "number",
            "description": "Output token budget. Defaults to 10000 tokens; larger requests may be capped by policy."
          },
          "shell": {
            "type": "string",
            "description": "Shell binary to launch. Defaults to the user's default shell."
          },
          "sandbox_permissions": {
            "type": "string",
            "enum": ["use_default", "require_escalated"],
            "description": "Per-command sandbox override. Defaults to `use_default`; use `require_escalated` for unsandboxed execution."
          },
          "justification": {
            "type": "string",
            "description": "User-facing approval question for require_escalated."
          },
          "prefix_rule": {
            "type": "array",
            "items": {
              "type": "string"
            },
            "description": "Reusable approval prefix for cmd, only with sandbox_permissions require_escalated."
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
~~~

它是网络层的差分，不是 core 历史只有 FunctionCallOutput。逻辑上的当前 input 仍有 base、用户、上一次 FunctionCall、FunctionCallOutput；客户端和服务端通过 previous_response_id 共享前缀。

HTTP 没有 previous_response_id，通常每次发送完整 input。WebSocket 若没有 completed response、字段前缀不匹配或 request 关键字段改变，就会退回完整请求。

## 十三、安全、steer、中断与取消

模型提出动作，runtime 决定是否执行：

~~~text
ResponseItem
  -> ToolRouter::build_tool_call
  -> ToolInvocation
  -> registry / pre-hook
  -> cancellation / approval / sandbox
  -> unified_exec handler
  -> ToolOutput
~~~

安全边界有三层：

1. 模型可见约束：工具 description、JSON Schema、权限说明和 environment 投影。
2. core runtime 判断：registry、pre/post hooks、审批策略、取消 token、diff tracker。
3. 操作系统/环境执行：文件系统 accessor、网络策略、进程沙箱和 shell backend。

sandbox_permissions、justification、prefix_rule 出现在 schema 中只是让模型知道可用字段；真正的权限决定仍在本地。environment_context 可能把 cwd、shell、filesystem permission profile 渲染给模型，但底层 sandbox policy 不会因为文本投影而失去强制性。

steer、interrupt、cancel 和新 turn 的边界：

- steer：排队新 user input，在下一次采样前记录；不修改已提交 arguments。
- interrupt：请求响应流优雅中断；是否继续 follow-up 由 stream 和 provider 状态决定。
- cancel：取消 token 传播到 sampling 与工具 future，可能导致 TurnAborted。
- 新 turn：新的生命周期和 turn id，不是当前 turn 的追加消息。

用户在工具运行期间 steer，通常要等当前采样和工具阶段完成或被中断；下一次请求才消费该文本。工具已经启动时，steer 不会变更正在运行的 shell 命令。

## 十四、历史、重试和失败

历史是 Vec<ResponseItem)，不是字符串拼接。它可能包含：

- developer/user context fragments；
- 原始用户消息；
- assistant message；
- reasoning；
- FunctionCall、CustomToolCall、ToolSearchCall；
- FunctionCallOutput、CustomToolCallOutput、ToolSearchOutput；
- compaction 或 context update item。

工具 output 由 drain_in_flight 持久化后，才会在下一次 clone_history 中出现。流断开、取消和参数错误会影响“哪些 item 已经落历史”，所以不能简单说重试永远复用原始 input。源码的 retry 路径会再次从 session history 构造 Prompt。

输出截断也属于请求内容的一部分：ExecCommandToolOutput 会根据模型输出预算、原始 token 数和 omission marker 截断 output；下一次模型看到的是截断后的 response_text，而不是必然完整 stdout。

## 十五、面试版总结

### 一分钟

在这个固定 commit 中，模型返回的是 Responses 协议的结构化 ResponseItem，不是自然语言命令。SSE 的 output_item.done 反序列化为 FunctionCall；ToolRouter::build_tool_call 把 name、namespace、原始 arguments 字符串和 call_id 变成 runtime ToolCall。router 再包装为带 session、turn、取消 token、tracker 的 ToolInvocation，registry 查找 handler，handler 用 serde_json::from_str<T> 解码 Rust 参数并执行 sandbox/approval。结果用同一个 call_id 生成 FunctionCallOutput，写入历史，下一次 sampling 再发送。base instructions、world-state、历史和用户请求先形成 Prompt，build_responses_request 才生成 ResponsesApiRequest。steer 只在下一请求前写入历史，不改变已经执行的工具。

### 三分钟

run_turn 的前置步骤包含 compact、step context、world-state、hooks、用户输入和工具注入，这些不等于网络请求。循环中 clone_history().for_prompt 取得结构化历史，build_prompt 设置可见工具、base instruction、并行开关和 output schema，client_session.stream 进入 build_responses_request。普通 Responses 把工具放顶层 tools，base instruction 通过 BaseInstructionsFragment 变成 developer input item；Lite 则将工具放进 AdditionalTools。模型响应由 protocol parser 变成 ResponseItem。此版本普通 function arguments 的 delta 不由 core 累积，完整 arguments 来自 output_item.done。build_tool_call 只做协议 item 到 runtime ToolCall 的转换；registry、hooks、取消、审批、sandbox 和具体 handler 决定执行。工具结果必须是带 call_id 的结构化 output。HTTP 每次通常发送完整逻辑 input，WS continuation 可用 previous_response_id 和尾差分。assistant message 只有在 Completed/end_turn、无 pending input 且无未完成工具等条件满足时才结束 turn。

### 面试追问速答

1. ToolCall 在哪里构造？  
   core/src/tools/router.rs 的 ToolRouter::build_tool_call。

2. 是否正则解析自然语言？  
   否。协议层先得到 ResponseItem，runtime 再做结构化转换。

3. arguments 是对象还是字符串？  
   FunctionCall.arguments 是包含 JSON 的字符串；handler 再 serde 解码。

4. Schema 是否保证正确参数？  
   它是模型约束；strict false，runtime 仍要解析、校验并处理错误。

5. ToolCall 和 ToolInvocation 区别？  
   ToolCall 是动作四字段；ToolInvocation 加入 session、turn、cancel、tracker、source 后供执行。

6. output 为什么需要 call_id？  
   让下一轮把 FunctionCallOutput 与原始调用配对。

7. 工具输出是否 assistant 文本？  
   不是；是 function_call_output 或 custom_tool_call_output。

8. 普通 function delta 是否累积？  
   本 commit 的 SSE parser 忽略普通 function arguments delta；完整 item 来自 output_item.done。

9. steer 什么时候生效？  
   pending input 被 run_turn 取出、记录历史后，影响下一次 sampling。

10. steer 会改已运行命令吗？  
    不会。它不能改变已提交 arguments。

11. HTTP 和 WS 的差异？  
    HTTP 通常发完整 input；WS 可用 previous_response_id 加尾差分。

12. Responses Lite 的工具在哪？  
    input 前缀的 AdditionalTools，而不是顶层 tools。

13. base instruction 在哪里？  
    固定实现把 BaseInstructionsFragment 转成 developer ResponseItem，插入 input。

14. Markdown 模板是否解析 ToolCall？  
    不能证明。两个 search_tool Markdown 在此 commit 未找到 active 加载；活跃描述由 Rust format! 生成。

15. output_schema 约束什么？  
    最终 assistant 文本；工具 arguments schema 是 tools.parameters，二者不同。

16. 工具执行错误如何恢复？  
    解析/handler 错误转 RespondToModel，带 call_id 形成下一轮可见输出。

17. sandbox 是 prompt 还是 runtime？  
    两者都有投影边界；文本可提示模型，强制执行在 runtime/OS。

18. 一次 tool call 是否一次模型请求？  
    不是。模型请求产生 call，本地执行后下一次请求才带 output。

19. 为什么不能把每个 Rust 函数写成请求？  
    大多数函数只在本地构造状态；真正发送点是 client_session.stream。

20. turn 什么时候结束？  
    当前采样没有待执行工具、收到 assistant message 并完成记录时结束；若有 function call，继续 sampling。

## 结论

完整链路可以压缩为：

~~~text
run_turn 本地准备
  -> clone_history().for_prompt
  -> build_prompt
  -> build_responses_request
  -> HTTP/WS stream
  -> ResponseItem
  -> ToolRouter::build_tool_call
  -> ToolInvocation / Registry / Handler
  -> ToolOutput(call_id)
  -> history
  -> 下一次完整逻辑请求
  -> assistant message
  -> turn 完成
~~~

最重要的源码事实是：Markdown、base instruction、历史和工具 schema共同影响模型上下文，但它们不会被一个 Markdown 解析器直接转换成 ToolCall。真正的构造边界是协议层 ResponseItem 到 Rust runtime ToolCall 的 build_tool_call；真正的闭环边界是 call_id 关联的工具 output 被写入下一次 sampling input。
