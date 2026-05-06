# Spring Boot 4 企业级脚手架优化设计

> 版本：v0.2 优化稿  
> 日期：2026-05-06  
> 来源：基于 `springboot4-enterprise-scaffold-design` 目录下 v0.1 草案分析、收敛与重写  
> 状态：可进入详细设计与原型验证

## 1. 本次优化做了什么

原设计覆盖了总体架构、环境部署、配置中心、多数据源、AI、认证、消息、依赖治理、编码规范和可观测性，主题完整，但整体仍停留在“方案候选集合”。本优化版做了四类调整：

1. 收敛默认技术路线，减少每篇文档反复出现的待确认项。
2. 修正版本基线，避免把尚未稳定或兼容性未验证的组合直接写成默认方案。
3. 把能力模块从“全量内置”调整为“核心强制、能力可插拔、风险能力渐进启用”。
4. 增加落地路线图、风险清单和验收标准，让设计能直接进入脚手架骨架开发。

## 2. 文档目录

| 文件 | 内容 |
| --- | --- |
| [01-设计评审与优化意见.md](01-设计评审与优化意见.md) | 对原设计的优点、问题和优化建议 |
| [02-目标架构与技术基线.md](02-目标架构与技术基线.md) | 优化后的总体架构、技术栈和版本基线 |
| [03-模块边界与仓库结构.md](03-模块边界与仓库结构.md) | mono-repo、BOM、starter、示例应用和业务接入方式 |
| [04-环境部署与配置治理.md](04-环境部署与配置治理.md) | local/test/beta/pro、配置分层、灰度、回滚 |
| [05-数据持久化与多租户.md](05-数据持久化与多租户.md) | CommonDB、BizDB、多数据源、事务、缓存、向量库 |
| [06-AI与RAG优化设计.md](06-AI与RAG优化设计.md) | Spring AI 适配、模型路由、RAG pipeline、成本与安全 |
| [07-认证鉴权与三方接入.md](07-认证鉴权与三方接入.md) | 身份模型、OIDC/OAuth2/SAML、RBAC+ABAC、数据权限 |
| [08-消息异步与任务调度.md](08-消息异步与任务调度.md) | 消息抽象、Outbox、幂等、重试、定时任务 |
| [09-依赖治理与编码规范.md](09-依赖治理与编码规范.md) | BOM、依赖准入、local-libs、代码质量门禁 |
| [10-可观测性与运维基线.md](10-可观测性与运维基线.md) | 日志、指标、链路、SLO、告警、备份 |
| [11-实施路线图与验收清单.md](11-实施路线图与验收清单.md) | 分阶段交付计划、验收项、风险清单 |
| [12-参考资料与版本依据.md](12-参考资料与版本依据.md) | Spring Boot、Spring Cloud、Spring AI、Nacos 等版本依据 |

## 3. 优化后的默认组合

| 领域 | 默认方案 | 说明 |
| --- | --- | --- |
| Java | JDK 21 LTS | 企业落地优先稳定；JDK 25 作为验证矩阵，不作为默认运行基线 |
| Spring Boot | 4.x | 原生对齐 Jakarta EE 11、Spring Framework 7 与 Jackson 3 生态 |
| 构建 | Maven + Maven Wrapper | 企业接受度高，BOM、Enforcer、Archetype 更易推广 |
| 仓库形态 | 脚手架 mono-repo + 业务独立仓 | 脚手架集中治理，业务项目独立演进 |
| 服务网关 | Spring Cloud Gateway | 默认走网关统一认证、限流、动态路由 |
| 配置中心 | Nacos 优先，但需兼容性验证 | 若 Spring Cloud Alibaba 对 Boot 4 支持滞后，短期保留 Spring Cloud Config 适配层 |
| 灰度发布 | Helm + Argo Rollouts | beta/pro 使用；local/test 不引入复杂发布控制面 |
| 数据库 | PostgreSQL 起步，MySQL 可选 | pgvector 可复用 PostgreSQL 运维体系 |
| ORM | MyBatis-Flex 或 MyBatis-Plus 二选一验证 | 不默认引入重型分片套件；读写分离先做轻量路由 |
| 消息 | Kafka 默认，RocketMQ 可选 | 默认 Outbox 保证业务与消息最终一致 |
| AI | Spring AI 2.0.x 兼容线薄封装 + Provider SPI | AI 能力可选启用；Spring AI 1.x 稳定线不作为 Boot 4 默认基线 |
| 观测 | OpenTelemetry + Prometheus/Grafana + Loki/Tempo | 三件套统一 traceId、tenantId、userId、requestId |

## 4. 关键原则

1. 核心能力强制：异常、日志、上下文、配置模型、依赖治理、代码规范必须统一。
2. 基础设施可替换：消息、缓存、AI、IdP、向量库、任务调度都走 SPI 或 Adapter。
3. 高风险能力渐进：分库分表、Service Mesh、双 AI 框架、CDC 不进入 v1 默认组合。
4. 业务代码低侵入：业务优先使用注解、接口和配置，不直接依赖平台实现类。
5. 运维先可见再自动化：灰度、自动回滚、配置热更新必须依赖明确指标和审计链路。

## 5. 需要立即拍板的事项

| 编号 | 决策 | 推荐 |
| --- | --- | --- |
| P0-1 | 是否接受 JDK 21 + Maven 作为唯一首版基线 | 接受 |
| P0-2 | 是否把 ShardingSphere 从默认依赖降为可选 starter | 接受 |
| P0-3 | 是否把 Spring Cloud Alibaba/Nacos 兼容性验证列为 P0 spike | 接受 |
| P0-4 | 是否首版只支持一种默认消息中间件 | 接受，默认 Kafka |
| P0-5 | 是否允许 AI 能力作为可选 starter 而不是核心启动依赖 | 接受 |

## 6. 版本校准说明

截至 2026-05-06，Spring Boot 4 当前文档显示最低 Java 要求不是 JDK 21，而是 Java 17；本设计选择 JDK 21 是企业 LTS 策略，不是框架最低要求。Spring AI 1.x 稳定文档面向 Spring Boot 3.4/3.5，Boot 4 适配应跟随 Spring AI 2.0.x 兼容线，因此 AI starter 必须先做独立兼容性验证。
