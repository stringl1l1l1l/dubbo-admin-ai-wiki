# 开发指南

本页面向日常维护和功能开发，目标不是覆盖所有工程规范，而是帮助你尽快进入正确的开发节奏。

## 1. 常用命令

### 运行

```bash
go run main.go --config ./config.yaml
```

### 构建

```bash
mkdir -p build
go build -o build/dubbo-admin-ai ./main.go
```

### 测试

```bash
go test ./... -v
```

### 覆盖率

```bash
go test ./... -cover -coverprofile=coverage.out
```

### Benchmark

```bash
go test -bench=. -run ^$ ./test
```

## 2. 建议的开发顺序

1. 先确认变更属于哪个层级：Server、Agent、Tools、RAG、Models 还是 Runtime。
2. 再确认这是“行为变更”还是“能力扩展”。
3. 修改代码前先读对应文档和配置。
4. 改完先跑相关测试，再跑全量测试。
5. 文档、配置、测试和代码一起提交。

## 3. 三类常见改动路径

### 改对外协议

先看：

- `component/server/engine`
- 用户指南 API 文档

### 改编排逻辑

先看：

- `component/agent`
- `prompts/`
- Agent 工作流文档

### 改知识与工具能力

先看：

- `component/tools`
- `component/rag`
- 相关配置 YAML

## 4. 工程约定

- 提交前执行 `gofmt` 或 `go fmt ./...`
- 建议执行 `go vet ./...`
- 优先早返回
- 错误要带上下文
- 配置和文档变更不要遗漏

## 5. 开发建议

不要只盯着单个目录改。这个项目的真实行为通常由“配置 + prompt + 组件代码 + 运行时”共同决定。只改一处，很容易造成“代码看起来改了，但实际行为没变”。
