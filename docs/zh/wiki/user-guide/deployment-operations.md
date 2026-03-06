# 部署与运维

本页聚焦“把服务稳定跑起来”这件事。对 Dubbo Admin AI 来说，部署并不只是把二进制启动成功，还包括模型密钥准备、长连接超时配置、日志观察点和故障时的降级策略。

## 1. 部署前要知道的事实

- 服务本质上是一个 Go HTTP 服务，默认监听 `localhost:8880`。
- 对外核心能力依赖模型 Provider；没有任何可用 API Key 时，服务可能能启动，但对话能力不可用。
- 流式接口依赖 SSE，所以网关、反向代理和 LB 的超时配置非常关键。
- 当前 Session 和 Memory 都是进程内状态，不是持久化存储。实例重启后上下文会丢失。

## 2. 构建与启动

### 本地运行

```bash
go run main.go --config ./config.yaml
```

### 构建二进制

```bash
mkdir -p build
go build -o build/dubbo-admin-ai ./main.go
```

### 使用二进制启动

```bash
./build/dubbo-admin-ai --config ./config.yaml
```

## 3. 配置准备

主配置入口是 `config.yaml`，其中引用了各组件 YAML：

- `component/logger/logger.yaml`
- `component/memory/memory.yaml`
- `component/models/models.yaml`
- `component/rag/rag.yaml`
- `component/tools/tools.yaml`
- `component/agent/agent.yaml`
- `component/server/server.yaml`

配置加载时会经历：

1. 读取 `.env`。
2. 对 YAML 执行环境变量展开。
3. 按 JSON Schema 填充默认值并校验。
4. 严格解码未知字段。

这意味着“拼错字段名但服务还能正常启动”的概率很低，这是好事。

## 4. 配置说明入口

YAML 配置解析已经拆分为独立页面，便于单独维护和引用：

- [YAML 配置详解](yaml-configuration.md)

如果你准备调整模型、工具、RAG 或服务参数，建议直接阅读这一页，而不是在部署章节中跳转查找。

## 5. 生产环境建议

### 5.1 环境变量

至少应准备以下变量中的一部分：

- `DASHSCOPE_API_KEY`
- `GEMINI_API_KEY`
- `SILICONFLOW_API_KEY`
- `COHERE_API_KEY`
- `PINECONE_API_KEY`

建议：

- 不同环境使用不同密钥。
- 不要把 `.env` 带入镜像。
- 通过 Secret Manager、Kubernetes Secret 或 CI/CD 注入。

### 5.2 反向代理 / 网关

SSE 很怕被中间层“优化掉”。建议重点确认：

- 请求超时足够长。
- 响应缓冲未吞掉流式刷新。
- 空闲连接时间合理。
- 代理支持 `text/event-stream`。

### 5.3 资源规划

资源消耗主要来自三块：

- 模型推理延迟和并发。
- 工具调用外部系统的耗时。
- RAG 检索与索引后端访问。

如果你把 mock tool 切换成真实工具，资源模型会明显变化，尤其是网络连接数和上游限流。

## 6. 健康检查与运行信号

### 基础健康检查

```bash
curl http://localhost:8880/health
```

### 线上真正该关注的信号

- HTTP 请求总量、错误率、P95/P99 延迟
- SSE 连接数、连接持续时间、中断率
- Agent 单轮总耗时
- 模型调用成功率、限流率、超时率
- 工具调用成功率、失败分布、平均耗时
- RAG 召回耗时、重排耗时、空召回比例

## 7. 最小运维 Runbook

### 场景一：服务不可用

按顺序检查：

1. 进程是否存在，端口是否监听。
2. `config.yaml` 和组件配置是否加载成功。
3. 关键环境变量是否存在。
4. `/health` 是否可访问。
5. 应用日志里是启动失败还是请求期失败。

### 场景二：接口可用但没有回答

按顺序检查：

1. Session 是否有效。
2. 模型 Provider 是否可访问。
3. Agent 是否进入 `think/act/observe` 阶段。
4. 工具或 RAG 是否卡住。
5. SSE 是否被代理层截断。

### 场景三：回答质量明显下降

按顺序检查：

1. 默认模型是否被改过。
2. Prompt 文件是否变更。
3. 工具是否注册成功。
4. RAG 索引是否过期或嵌入模型不匹配。
5. 是否有新的上游限流或 Provider 行为变化。

## 8. 降级策略建议

当前项目本身还没有非常完整的多级降级编排，但运维层面可以按下列顺序控制影响面：

1. 先禁用 MCP 工具，只保留 internal/mock。
2. 再禁用 RAG，只保留纯模型回答。
3. 如模型调用异常，切换到另一个已配置 Provider。
4. 必要时仅保留创建会话与基础健康检查，对外声明对话能力降级。

## 9. 当前部署层面的已知约束

- Session 与 Memory 不持久化，不适合直接做多实例共享上下文。
- Runtime 注册表按 `Component.Name()` 存储，当前组件基本都是固定名，天然偏向单实例。
- `/health` 只做进程级存活检查，不包含依赖探活。
- 部分配置字段在 Schema 存在，但不一定完全进入执行链路，需要结合开发文档确认。
