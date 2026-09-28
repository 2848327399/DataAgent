# Spring AI × DataAgent 零基础学习手册

这套手册面向第一次接触 Spring AI、但具备基础 Java / Spring Boot 知识的读者。目标不是记住所有 API，而是在最短时间内建立一条可用的主线：

```text
模型调用 → 提示词 → 结构化输出 → 流式响应
       → Embedding / VectorStore / RAG
       → Memory / Tool / MCP
       → Spring AI Alibaba Graph → 本项目完整工作流
```

## 这套手册与已有项目指南的关系

- 本目录回答“Spring AI 的概念是什么，这个项目为什么这样写”。
- [DataAgent 项目实战学习指南](../dataagent-study-guide/README.md)回答“项目怎样启动、怎样配置、怎样按页面日志排障”。
- 两套资料互补。建议先读本手册 01～07，再进入项目实战指南。

## 推荐阅读顺序

| 阶段 | 章节 | 读完能做到 |
| --- | --- | --- |
| 建立地图 | [01-先建立正确认知](01-先建立正确认知.md) | 分清模型、Spring AI、Spring AI Alibaba 和 DataAgent。 |
| 掌握模型调用 | [02-ChatClient与提示词](02-ChatClient与提示词.md) | 看懂项目怎样创建模型、发送 system/user 消息。 |
| 掌握可靠输出 | [03-结构化输出与流式响应](03-结构化输出与流式响应.md) | 看懂 JSON 输出、校验重试、Flux 和 SSE。 |
| 掌握知识检索 | [04-Embedding向量库与RAG](04-Embedding向量库与RAG.md) | 看懂数据源初始化、Schema 召回和知识召回。 |
| 掌握扩展能力 | [05-记忆工具调用与MCP](05-记忆工具调用与MCP.md) | 分清多轮记忆、Tool、MCP 和普通 Java 调用。 |
| 掌握 Agent 编排 | [06-Spring-AI-Alibaba-Graph](06-Spring-AI-Alibaba-Graph.md) | 看懂节点、边、状态、条件路由、检查点。 |
| 对照项目 | [07-本项目完整调用链](07-本项目完整调用链.md) | 从 `/api/stream/search` 一路追到最终报告。 |
| 动手巩固 | [08-源码阅读与动手实验](08-源码阅读与动手实验.md) | 按风险从低到高修改并验证代码。 |
| 日常速查 | [09-排障与速查表](09-排障与速查表.md) | 快速定位模型、JSON、向量检索、Graph 和 SSE 问题。 |

## 版本边界

本手册按仓库当前版本编写：

- Java 17
- Spring Boot 3.4.8
- Spring AI 1.1.2
- Spring AI Alibaba 1.1.2.2

网上示例可能来自 Spring AI 2.x，包名、Starter 名称或 Tool/Advisor 行为可能不同。学习本项目时，以根目录 `pom.xml` 和实际源码为准，不要直接复制其他版本代码。

## 30 分钟快速路线

如果今天只想尽快读懂项目，请只做下面几件事：

1. 阅读 01，记住 `ChatModel`、`ChatClient`、`EmbeddingModel`、`VectorStore`、`StateGraph`。
2. 阅读 02 和 03，对照 `StreamLlmService`、`IntentRecognitionNode`。
3. 阅读 04，对照 `EvidenceRecallNode`、`SchemaRecallNode`。
4. 阅读 06 和 07，对照 `DataAgentConfiguration`、`GraphServiceImpl`。
5. 最后打开 `resources/prompts/`，任选一个提示词，从“输入 → 模型 → DTO → 状态 → 下一节点”完整追一次。

## 学习原则

- 把大模型当作“不稳定的外部服务”，不要当作普通确定性函数。
- Prompt 是程序的一部分，需要版本管理、测试和安全边界。
- JSON 解析成功不等于业务语义正确，仍需校验、路由和重试。
- RAG 不是“接一个向量库”就结束，而是索引、检索、过滤、拼接上下文的完整链路。
- Agent 不是一个神秘类，而是模型、工具、状态和控制流的组合。

## 官方延伸阅读

- [Spring AI 1.1 ChatClient](https://docs.spring.io/spring-ai/reference/1.1/api/chatclient.html)
- [Spring AI 1.1 Tool Calling](https://docs.spring.io/spring-ai/reference/1.1/api/tools.html)
- [Spring AI RAG](https://docs.spring.io/spring-ai/reference/api/retrieval-augmented-generation.html)
- [Spring AI Alibaba 项目](https://github.com/alibaba/spring-ai-alibaba)
- [Spring AI Alibaba Graph 核心说明](https://github.com/alibaba/spring-ai-alibaba/blob/main/spring-ai-alibaba-graph-core/README.md)

> 安全提醒：API Key、数据库密码、真实业务数据和个人信息都不要写进教材、日志、测试数据或 Git 提交。
