# AI 与 RAG 优化设计

## 1. 定位

AI 能力是平台可选 starter，不是所有业务服务的核心依赖。业务服务通过平台暴露的 `AiClient`、`EmbeddingClient`、`RagService` 等接口使用 AI，不直接依赖具体模型 SDK，也不直接依赖 Spring AI 的全部 API。

这样可以在模型、供应商、限额、审计、提示词治理变化时减少业务改动。

## 2. 分层架构

```text
Biz UseCase
   │
   ▼
starter-ai API
   │  AiClient / RagService / PromptTemplateService
   ▼
AI Orchestrator
   │  routing / fallback / quota / audit
   ▼
Provider Adapter
   │  OpenAI-compatible / Azure / local model / custom
   ▼
Model Provider
```

RAG 链路：

```text
document source
  └── ingestion
        └── parsing
              └── chunking
                    └── embedding
                          └── vector store
                                └── retrieval
                                      └── rerank
                                            └── generation
```

## 3. 技术基线

| 项 | 策略 |
| --- | --- |
| AI 框架 | Spring AI 2.0.x 兼容线；GA 前只作为可选 starter |
| 适配方式 | 平台薄封装，不把 Spring AI 对象扩散到业务层 |
| 模型协议 | 优先 OpenAI-compatible API，保留 provider SPI |
| 向量库 | pgvector 起步 |
| 流式输出 | WebFlux/SSE 可选，默认同步接口先稳定 |
| 工具调用 | 只允许注册经过审计的 Tool |

## 4. Provider 路由

配置示例：

```yaml
platform:
  ai:
    default-scene: general-chat
    scenes:
      general-chat:
        provider: openai-compatible
        model: gpt-compatible-large
        timeout: 30s
        max-output-tokens: 2048
      embedding:
        provider: openai-compatible
        model: text-embedding-compatible
        timeout: 10s
    fallback:
      enabled: true
      order: [primary, backup]
```

路由维度：

1. 场景：chat、embedding、rerank、classification、tool-call。
2. 租户：不同租户可绑定不同 provider。
3. 成本：超过预算自动降级或拒绝。
4. 可用性：provider 异常时 fallback。
5. 合规：敏感数据禁止发往外部模型。

## 5. RAG Pipeline

### 5.1 Ingestion

支持来源：

1. Markdown/HTML/PDF/Office 文档。
2. 数据库记录。
3. 对象存储文件。
4. API 拉取内容。

每次导入生成 `knowledge_job`，记录来源、版本、hash、处理状态和错误信息。

### 5.2 Parsing

要求：

1. 保留标题层级。
2. 保留表格结构的可读文本。
3. 记录原始文档位置。
4. 失败文档进入重试队列，不阻塞整个批次。

### 5.3 Chunking

默认策略：

| 参数 | 默认 |
| --- | --- |
| chunk size | 800-1200 tokens |
| overlap | 10%-15% |
| metadata | tenantId、kbId、docId、section、version、hash |
| 去重 | 按 doc hash + chunk hash |

### 5.4 Retrieval

检索流程：

1. query rewrite。
2. embedding。
3. vector search topK。
4. metadata filter。
5. rerank。
6. context pack。

必须支持租户隔离和知识库隔离，禁止跨租户召回。

### 5.5 Generation

输出要求：

1. 必须携带引用来源。
2. 低置信度时返回“不足以回答”的业务错误，而不是编造。
3. 敏感操作必须进入人工确认。
4. 生成内容写入审计日志，但注意脱敏。

## 6. Token 与成本治理

| 能力 | 要求 |
| --- | --- |
| 配额 | 按租户、应用、用户、场景设置日/月额度 |
| 限流 | 按 provider 和模型分别限流 |
| 预算 | 超预算后降级、排队或拒绝 |
| 统计 | 记录 input/output tokens、费用估算、耗时 |
| 看板 | 展示租户、应用、模型维度成本 |

## 7. 安全

1. 默认开启提示词注入检测。
2. Tool calling 采用白名单注册。
3. 外部模型调用前做敏感信息脱敏。
4. RAG 检索必须带 ACL filter。
5. 原始文件和向量数据按租户隔离。
6. prompt、completion、工具调用结果写审计。

## 8. 异步与流式

| 场景 | 方式 |
| --- | --- |
| 短 chat | 同步 HTTP |
| 长文档问答 | 异步 job + 轮询 |
| 流式生成 | SSE |
| 批量 embedding | 消息队列 + worker |
| 大批知识库导入 | job 表 + 分片任务 |

## 9. 首版交付范围

v1：

1. `starter-ai` 基础 chat 和 embedding。
2. Provider SPI。
3. pgvector 向量存储。
4. Markdown 文档 ingestion。
5. 基础 RAG 问答。
6. Token 统计和租户限额。

v1 暂不做：

1. 多 AI 框架双适配。
2. 复杂 Agent。
3. 自动工具调用开放平台。
4. 多模态 RAG。
5. 专用向量数据库集群。

## 10. 兼容性注意

Spring AI 1.x 稳定线面向 Spring Boot 3.4/3.5，不应直接作为 Spring Boot 4 脚手架默认依赖。Boot 4 首版若必须交付 AI 能力，应使用 Spring AI 2.0.x 兼容线做 spike，并把 AI starter 与核心 Web/Data/Security starter 解耦。
