# Logger 组件

Logger 是系统里最小、但影响范围最大的组件。它本身不复杂，却决定了整个运行时后续看到的日志格式和日志级别。

## 1. 它负责什么

- 读取日志级别配置
- 创建 `slog` handler
- 调用 `slog.SetDefault(...)` 设置全局默认 logger

这意味着 Logger 组件不是“提供一个 logger 给别人用”，而是直接改写全局日志行为。

## 2. 为什么它必须尽早初始化

因为 runtime 和其他组件都通过 `slog.Default()` 间接取日志实例。如果 Logger 没有先完成初始化，系统会退回到 Go 默认 logger 行为，导致日志格式不统一。

## 3. 配置示例

```yaml
type: logger
spec:
  level: "info"
```

当前支持的级别为：

- `debug`
- `info`
- `warn`
- `error`

## 4. 实现要点

- 工厂：`component/logger/factory.go`
- 配置：`component/logger/config.go`
- 组件：`component/logger/component.go`

读取路径很简单：

1. factory 把 YAML `spec` 解码为 `LoggerSpec`
2. `Validate()` 检查 level 是否合法
3. `Init()` 根据 level 初始化 handler 并设置为默认 logger

## 5. 它的边界在哪里

Logger 当前只负责“日志输出入口”，不负责：

- 日志落盘
- 日志轮转
- trace 关联
- metrics 打点
- 按请求注入字段

如果后续要做结构化 JSON、文件输出或远程日志采集，应该扩展 Logger 组件，而不是在各业务组件里各写一套。

## 6. 当前限制

- 配置维度还比较少，主要只有 `level`。
- 更复杂的 handler 选择逻辑还没有抽象出来。
- 请求级字段仍需要业务代码自己附加，比如 `request_id`、`session_id`。
