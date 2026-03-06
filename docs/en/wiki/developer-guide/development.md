# Development Guide

This page is for day-to-day maintenance and feature work. The goal is not to cover every engineering rule, but to help you enter the right development rhythm quickly.

## 1. Common commands

### Run

```bash
go run main.go --config ./config.yaml
```

### Build

```bash
mkdir -p build
go build -o build/dubbo-admin-ai ./main.go
```

### Test

```bash
go test ./... -v
```

### Coverage

```bash
go test ./... -cover -coverprofile=coverage.out
```

### Benchmark

```bash
go test -bench=. -run ^$ ./test
```

## 2. Recommended development order

1. Identify which layer the change belongs to: Server, Agent, Tools, RAG, Models, or Runtime.
2. Decide whether it is a behavior change or a capability extension.
3. Read the corresponding docs and config before editing code.
4. Run targeted tests first, then the full test suite.
5. Commit docs, config, tests, and code together.

## 3. Three common change paths

### External protocol changes

Start with:

- `component/server/engine`
- the User Guide API docs

### Orchestration logic changes

Start with:

- `component/agent`
- `prompts/`
- the Agent Workflow docs

### Knowledge and tool capability changes

Start with:

- `component/tools`
- `component/rag`
- the related YAML config

## 4. Engineering conventions

- Run `gofmt` or `go fmt ./...` before committing
- Prefer running `go vet ./...`
- Prefer early returns
- Add context to errors
- Do not forget config and documentation changes

## 5. Development advice

Do not stare at only one directory. Real behavior in this project is usually determined by config, prompts, component code, and runtime together. If you change only one of them, it is easy to end up with "the code changed, but behavior did not".
