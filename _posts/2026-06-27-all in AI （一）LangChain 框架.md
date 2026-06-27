---
layout:     post
title:      all in AI （一）LangChain 框架
subtitle:  all in AI （一）LangChain 框架
date:       2026-06-27
author:     ica10888
catalog: true
tags:
    - AI 
    - LangChain
---

# all in AI （一）LangChain 框架

现代AI模型的进度速度飞快，可以说从最开始的简单模型使用到AI嵌入到人们的日常之中，可以说是日新月异，沧海桑田地快速发展。

基本发展可以有以下的发展路径

SFT -> LoRA微调 ->  Prompts engineering -> LangChain -> RAG -> AI Agent -> harness engineering -> loop engineering

今天主要讲述的是 LangChain 的基本概念，从中我们可以管中窥豹，学习到一些AI模型的思路。

### LangChain 框架基本概念

一般来说，LangChain 有六个部分组成

- 模型 （Model）包含各大语言模型的 LangChain 接口和调用细节，以及输出解析机制
- 提示模板 （Prompts）使提示工程流线化，进一步激发大语言模型的潜力。
- 链 （Chains）是 LangChain 中的核心机制，以特定方式封装各种功能，并通过一系列的组合，自动而灵活地完成常见用例。

- 记忆 （Memory）通过短时记忆和长时记忆，在对话过程中存储和检索数据，让 ChatBot 记住你是谁。
- 代理（Agents）是另一个 LangChain 中的核心机制，通过“代理”让大模型自主调用外部工具和内部工具，使强大的“智能化”自主 Agent 成为可能！
- 数据检索 （Indexes）构建并操作文档的方法，接受用户的查询并返回最相关的文档，轻松搭建本地知识库。



##### 模型（Model）

支持三种模型，大语言模型（LLM），聊天模型（Chat Model），文本嵌入模型（Embedding Model）

##### 提示模板 （Prompts）

LangChain 提示词框架会自动添加 schema

LangChain 中提供 String（StringPromptTemplate）和 Chat（BaseChatPromptTemplate）两种基本类型的模板

Few-Shot（少样本）、One-Shot（单样本）和与之对应的 Zero-Shot（零样本）的概念都起源于机器学习。

``` python
from langchain.prompts.prompt 
import PromptTemplatefrom langchain.prompts 
import FewShotPromptTemplatefrom langchain.prompts.pipeline 
import PipelinePromptTemplatefrom langchain.prompts 
import ChatPromptTemplatefrom langchain.prompts 
import (    
    ChatMessagePromptTemplate,    
    SystemMessagePromptTemplate,    
    AIMessagePromptTemplate,    
    HumanMessagePromptTemplate,
)
```

可以使用思考链模式写提示词（Chain of Thought, CoT），进一步思维树（Tree of Thoughts，ToT），再进一步思考图（Graph of Thoughts , GoT）

解释器

``` python
from langchain.output_parsers import PydanticOutputParser
from langchain.output_parsers import RetryWithErrorOutputParser
```

##### 链 （Chains）

使各部分链接起来

``` python
# 导入所需的库
from langchain import PromptTemplate, OpenAI, LLMChain
# 原始字符串模板
template = "{flower}的花语是?"
# 创建模型实例
llm = OpenAI(temperature=0)
# 创建LLMChain
llm_chain = LLMChain(
    llm=llm,
    prompt=PromptTemplate.from_template(template))
# 调用LLMChain，返回结果
result = llm_chain("玫瑰")
print(result)
```

链的分类

- LLMChain ：最基础的链。围绕着语言模型推理功能又添加了一些功能，整合了 PromptTemplate、语言模型（LLM 或聊天模型）和 Output Parser，相当于把 Model I/O 放在一个链中整体操作。
- Sequential Chain：顺序多次调用模型，io，多个输入、输出。前一个的输出是后一个输入。
- TransformChian：通过设置转换函数，用于分割语句，满足LLM 的 Token限制，输出总结。
- RouterChian：包含判断和一系列目标链。通过LLM，动态判断条件，判断走哪个分支。一般配合使用MultiPromptChain。
- 其他：LLMMathChian 使用数学模型，RetrievalQA：嵌入向量索引进行查询。

##### 记忆 （Memory）

使用 ConversationChain ，记录摘要。

在 LangChain 中，通过 ConversationBufferMemory（缓冲记忆）可以实现最简单的记忆机制。

``` python 
from langchain import OpenAI
from langchain.chains import ConversationChain
from langchain.chains.conversation.memory import ConversationBufferMemory

# 初始化大语言模型
llm = OpenAI(
    temperature=0.5,
    model_name="gpt-3.5-turbo-instruct")

# 初始化对话链
conversation = ConversationChain(
    llm=llm,
    memory=ConversationBufferMemory()
)

# 第一天的对话
# 回合1
conversation("我姐姐明天要过生日，我需要一束生日花束。")
print("第一次对话后的记忆:", conversation.memory.buffer)
```

记忆的分类

-  ConversationBufferMemory：最简单的类型。
- ConversationBufferWindowMemory : 缓冲窗口记忆，它的思路就是只保存最新最近的几次人类和 AI 的互动。
- ConversationSummaryMemory: 将对话历史进行汇总，然后再传递给 {history} 参数。
- ConversationSummaryBufferMemory：混合记忆模型，结合了上述各种记忆机制。

##### 代理 （Agent）

 先思考，作为推理过程（Reasoning），然后指导行动（Acting），就被称作智能代理，或者说是 ReAct框架。

关键组件

- 代理（Agent）：这个类决定下一步执行什么操作。
- 工具（Tools）：工具是代理调用的函数。
- 工具包（Toolkits）：工具包是一组用于完成特定目标的彼此相关的工具，每个工具包中包含多个工具。
- 代理执行器（AgentExecutor）：代理执行器是代理的运行环境，它调用代理并执行代理选择的操作。

工具

``` python
# 设置OpenAI API的密钥
import os 
os.environ["OPENAI_API_KEY"] = 'Your Key' 

# 导入库
from langchain.chat_models import ChatOpenAI
from langchain.agents import load_tools, initialize_agent, AgentType

# 初始化模型和工具
llm = ChatOpenAI(temperature=0.0)
tools = load_tools(
    ["arxiv"],
)

# 初始化链
agent_chain = initialize_agent(
    tools,
    llm,
    agent=AgentType.ZERO_SHOT_REACT_DESCRIPTION,
    verbose=True,
)

# 运行链
agent_chain.run("介绍一下2005.14165这篇论文的创新点?")
```

##### 数据检索 （Indexes）

已经被整合到 Retrieval（数据检索）这个单元中了，还有callback，这里就不展开了。

CAMEL - 多 AI 通过角色扮演进行交互的框架。Simulation Agents（模拟代理）和 Autonomous Agents（自治代理或自主代理）
