# 05 · AI 集成与 RAG

> 解决：**Spring Boot 4.x 下如何引入 AI、RAG 怎么搭、模型怎么抽象、Token 与成本怎么治理、AI 输出怎么落入业务流程**

---

## 1. 整体架构

```
┌──────────────────────────────────────────────────────────────┐
│                       业务 BizApp                             │
│   ┌─────────────────────────────────────────────────────┐    │
│   │   AI Use Case 层（编排，业务可见）                    │    │
│   │   · 智能问答  · 文档摘要  · 工单分类  · Agent 执行    │    │
│   └────────────────┬────────────────────────────────────┘    │
└────────────────────┼───────────────────────────────────────────┘
                     │
   ┌─────────────────▼─────────────────────────────────────┐
   │           starter-ai（脚手架，业务无需感知模型差异）    │
   │  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐ │
   │  │ ChatClient   │  │ RAG Pipeline │  │ Tool / Func  │ │
   │  │ (chat/stream)│  │ (retrieve+gen)│  │ Calling      │ │
   │  └──────┬───────┘  └──────┬───────┘  └──────┬───────┘ │
   │         │                 │                 │         │
   │  ┌──────▼─────────────────▼─────────────────▼───────┐ │
   │  │          ModelProvider 抽象（SPI）                │ │
   │  └──────┬──────┬──────┬──────┬──────┬──────────────┘ │
   └─────────┼──────┼──────┼──────┼──────┼─────────────────┘
             ▼      ▼      ▼      ▼      ▼
         OpenAI  Azure  通义  DeepSeek  Ollama(本地)
                 ▲              ▲
                 │              │
              Embedding      Reranker
                 ▼
         ┌─────────────────┐
         │  Vector DB      │  ← 向量库（详见 04）
         │  pgvector/Milvus│
         └─────────────────┘
```

---

## 2. 核心抽象（starter-ai 暴露给业务的 SPI）

| 接口 | 职责 | 业务调用示例（伪代码） |
|------|------|------------------------|
| `ChatClient` | 单轮 / 多轮对话，支持流式 | `chatClient.prompt(...).user(q).call()` |
| `EmbeddingClient` | 文本 → 向量 | `embedding.embed("text")` |
| `RerankClient` | 候选重排 | `reranker.rerank(query, candidates)` |
| `RagPipeline` | 检索增强生成（封装 retrieve+rerank+generate） | `rag.ask(query, scope)` |
| `ToolRegistry` | 工具 / Function Calling 注册 | `tools.register(MyTool.class)` |
| `AgentExecutor` | 多步推理 Agent（可选，谨慎用） | `agent.run(goal)` |
| `PromptStore` | Prompt 模板版本化（不写死代码） | `promptStore.get("qa.v3")` |

**关键约束**：业务代码**禁止**直接 import 任何模型 SDK（OpenAI、通义等），必须经 starter-ai。

---

## 3. 模型 Provider 抽象（多模型策略）

### 3.1 配置驱动（Nacos）

```yaml
ai:
  providers:
    - name: openai-gpt4o
      type: openai
      base-url: ${OPENAI_PROXY}
      api-key-ref: secret://ai/openai
      models: [gpt-4o, gpt-4o-mini]
    - name: tongyi
      type: dashscope
      api-key-ref: secret://ai/tongyi
      models: [qwen-max, qwen-plus]
    - name: local-ollama
      type: ollama
      base-url: http://ollama:11434
      models: [llama3, qwen2]
  routing:
    default-chat: openai-gpt4o
    default-embedding: tongyi:text-embedding-v3
    rules:
      - match: { tenant: "tenant-a", task: "chat" }
        provider: tongyi:qwen-max
      - match: { task: "summary", token-budget: low }
        provider: openai-gpt4o-mini
      - match: { env: local }
        provider: local-ollama
```

### 3.2 路由策略（按场景）

| 维度 | 示例 |
|------|------|
| 任务类型 | chat / summary / classify / embedding |
| 租户 / 数据敏感度 | 国资客户 → 仅国产模型 |
| 成本预算 | 低预算 → mini 模型 |
| 环境 | local → Ollama，生产 → 云模型 |
| 降级链 | 主模型超时 → 备模型 → 降级返回 |

---

## 4. RAG Pipeline（标准化 4 阶段）

```
┌────────────┐   ┌────────────┐   ┌────────────┐   ┌────────────┐
│ Ingestion  │ → │ Chunking   │ → │ Embedding  │ → │ Indexing   │
│ (离线)      │   │ +元数据    │   │            │   │ (向量库)    │
└────────────┘   └────────────┘   └────────────┘   └────────────┘

Query 时：
┌────────┐  ┌──────────┐  ┌──────────┐  ┌────────┐  ┌────────┐
│ Query  │→│ Rewrite  │→│ Retrieve │→│ Rerank │→│ Generate│
│        │  │ (可选)   │  │ (向量+BM25)│ │        │  │         │
└────────┘  └──────────┘  └──────────┘  └────────┘  └────────┘
```

### 4.1 Ingestion 来源
- 业务库 CDC（实时同步业务数据进 RAG）
- 文档上传（PDF/Word/Markdown，走对象存储 + 解析队列）
- 网页爬取（受控源）

### 4.2 Chunking 策略
- 默认：递归切分 + 语义边界（以段落 / 标题为优先）
- chunk_size 默认 512 token，overlap 50
- 必带元数据：`tenant_id, source, doc_id, chunk_idx, created_at, acl`

### 4.3 检索
- **混合检索**：向量召回 (top 50) + BM25 (top 50) → RRF 融合 → top 20
- ACL 过滤：tenant_id / 用户权限标签必须在 retrieval 层过滤，不能在生成后过滤
- Reranker（可选）：bge-reranker / cohere

### 4.4 生成
- Prompt 模板版本化（`PromptStore` 管理）
- 强制带 source 引用（让 LLM 输出包含 [^1] 标注）
- 长上下文截断策略：相似度低的 chunk 优先丢弃

---

## 5. Token 与成本治理（企业级必须）

| 治理点 | 手段 |
|--------|------|
| **配额** | 按租户 / 用户 / 应用三级配额（CommonDB 配置） |
| **限流** | 模型级 RPS / TPM 限制（避免触发 provider 限流） |
| **缓存** | 查询缓存（同样 prompt 命中缓存）+ embedding 缓存 |
| **审计** | 每次调用记录：prompt hash、token 数、模型、耗时、成本 |
| **预算告警** | 日 / 月预算超阈值告警，可自动降级到便宜模型 |
| **成本视图** | Grafana 面板：按租户 / 应用 / 模型 / 任务维度 |

---

## 6. 安全与合规

| 风险 | 应对 |
|------|------|
| **Prompt 注入** | 输入分段标记 + 输出后过滤 + 系统 prompt 强约束 |
| **数据泄漏** | 脱敏后送外部模型；敏感数据强制走本地 / 国产模型 |
| **越权访问** | RAG 检索阶段强制 ACL 过滤，不依赖 LLM 自觉 |
| **幻觉** | 强制带引用、未召回到结果时拒答、关键场景人工复核 |
| **合规审计** | 完整日志（脱敏后）保留 ≥ 6 月 |
| **Function Calling** | 工具白名单 + 参数校验 + 执行权限分级 |

---

## 7. 异步与流式

- **流式**：starter-ai 提供 SSE 适配，业务 Controller 直接 `return Flux<String>`
- **长任务**：抽取 / 索引 / Agent 执行走任务队列（详见 07），结果回调或轮询
- **超时分级**：
  - 同步对话：30s（流式可放宽到 120s）
  - Embedding：10s
  - RAG：60s
  - Agent：5min（必须显式声明）

---

## 8. 与 Spring AI 4.x 的关系

- starter-ai 是 Spring AI 的**薄封装层**，目的：
  - 屏蔽 Spring AI API 变更（防止业务代码大改）
  - 加入企业治理（配额、审计、路由、Prompt 仓库）
  - 提供 RAG 全套封装（Spring AI 只给原子能力）
- 同时预留 LangChain4j 适配（备选方案，用 SPI 切换）

---

## 9. AI 输出如何融入业务（关键工程问题）

| 场景 | 设计 |
|------|------|
| AI 结果直接返回用户 | 必须流式 + 带引用 + 一键举报反馈 |
| AI 结果落库 | 必须打 `ai_generated=true` 标 + 模型版本 + prompt 版本 |
| AI 结果触发动作（如自动派单） | 必须人工兜底 + 置信度阈值 + 灰度比例 |
| AI 用于决策辅助 | 不入业务流，仅作 UI 提示 |

---

## 10. 关键决策点（待你确认）

| # | 决策点 | 选项 | 我的倾向 |
|---|--------|------|----------|
| D5.1 | AI 框架基座 | A. Spring AI 1.x / B. LangChain4j / C. 双适配 | **A**（SB4 原生） |
| D5.2 | 默认外部模型 | A. OpenAI / B. 通义 / C. DeepSeek / D. 多 provider | **D** |
| D5.3 | 本地模型方案 | A. Ollama / B. vLLM / C. 不支持本地 | **A**（local 环境用） |
| D5.4 | RAG 检索方式 | A. 纯向量 / B. 向量+BM25 混合 / C. 三段式（含 KG） | **B** |
| D5.5 | Reranker | A. 默认开（bge）/ B. 可选 / C. 不开 | **B** |
| D5.6 | Prompt 管理 | A. 代码内 / B. 配置中心 / C. 独立 PromptStore（DB） | **C** |
| D5.7 | Agent 是否一等公民 | A. 是 / B. 实验性可选 / C. 不支持 | **B**（谨慎） |
| D5.8 | AI 调用是否走网关 | A. 走，统一鉴权审计 / B. 直连 provider | **A** |

---

## 11. 待澄清问题

1. AI 主要场景是什么？（智能问答 / 文档处理 / 数据分析 / Agent 自动化）影响优先实现哪些 Use Case 模板。
2. 是否有"必须用国产模型"的客户 / 行业要求？影响默认 provider 选择。
3. RAG 文档量级？（万 / 百万 / 亿）影响向量库选型。
4. 是否需要"AI 训练 / 微调"链路？还是只用现成模型？影响是否引入数据标注、训练框架。
5. 预算敏感度？是否需要严格的成本上限和强制降级？
