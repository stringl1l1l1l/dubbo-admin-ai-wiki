# Logger Component

Logger is the smallest component in the system, but it has one of the widest effects. It is simple, but it decides the log format and log level that the rest of the runtime sees.

## 1. What it does

- reads the configured log level
- creates the `slog` handler
- calls `slog.SetDefault(...)` to replace the global default logger

That means Logger does not "provide a logger to others". It directly rewrites global logging behavior.

## 2. Why it must initialize early

Runtime and the other components indirectly obtain a logger through `slog.Default()`. If Logger has not initialized first, the system falls back to Go's default logger behavior and log formatting becomes inconsistent.

## 3. Example config

```yaml
type: logger
spec:
  level: "info"
```

Currently supported levels:

- `debug`
- `info`
- `warn`
- `error`

## 4. Implementation highlights

- factory: `component/logger/factory.go`
- config: `component/logger/config.go`
- component: `component/logger/component.go`

The path is straightforward:

1. the factory decodes YAML `spec` into `LoggerSpec`
2. `Validate()` checks whether `level` is valid
3. `Init()` creates the handler for the level and sets it as the default logger

## 5. Component boundary

Logger currently only owns the log output entry. It does not handle:

- log persistence
- log rotation
- trace correlation
- metrics emission
- per-request field injection

If the project later needs structured JSON logs, file output, or remote log collection, extend the Logger component instead of building separate logging logic inside each business component.

## 6. Current limitations

- The config surface is still small, mostly just `level`.
- More complex handler selection has not been abstracted yet.
- Request-level fields such as `request_id` and `session_id` still need to be attached by business code.
