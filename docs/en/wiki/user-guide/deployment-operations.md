# Deployment & Operations

This page focuses on one thing: keeping the service running reliably. For Dubbo Admin AI, deployment is not just about starting a binary. It also includes model credential preparation, long-lived connection timeouts, logging signals, and fallback strategies during incidents.

## 1. Facts to know before deployment

- The service is fundamentally a Go HTTP service and listens on `localhost:8880` by default.
- Its core external capability depends on model providers. Without any usable API key, the service may start but chat will not work.
- The streaming API depends on SSE, so timeouts in gateways, reverse proxies, and load balancers matter a lot.
- Session and Memory are currently in-process state, not persistent storage. Context is lost after a restart.

## 2. Build and start

### Run locally

```bash
go run main.go --config ./config.yaml
```

### Build a binary

```bash
mkdir -p build
go build -o build/dubbo-admin-ai ./main.go
```

### Start with the binary

```bash
./build/dubbo-admin-ai --config ./config.yaml
```

## 3. Configuration preparation

The main config entry is `config.yaml`, which references all component YAML files:

- `component/logger/logger.yaml`
- `component/memory/memory.yaml`
- `component/models/models.yaml`
- `component/rag/rag.yaml`
- `component/tools/tools.yaml`
- `component/agent/agent.yaml`
- `component/server/server.yaml`

Config loading goes through:

1. Reading `.env`
2. Expanding environment variables in YAML
3. Applying JSON Schema defaults and validation
4. Strictly decoding unknown fields

That makes "the field name is wrong but the service still started fine" much less likely, which is a good thing.

## 4. Configuration reference entry

The YAML configuration breakdown is documented on its own page for easier maintenance and reuse:

- [YAML Configuration](yaml-configuration.md)

If you plan to tune models, tools, RAG, or server parameters, read that page directly instead of jumping around inside the deployment section.

## 5. Production recommendations

### 5.1 Environment variables

Prepare at least some of these variables:

- `DASHSCOPE_API_KEY`
- `GEMINI_API_KEY`
- `SILICONFLOW_API_KEY`
- `COHERE_API_KEY`
- `PINECONE_API_KEY`

Recommendations:

- Use different keys for different environments.
- Do not bake `.env` into container images.
- Inject secrets through a Secret Manager, Kubernetes Secret, or CI/CD.

### 5.2 Reverse proxy / gateway

SSE is easy for middle layers to accidentally break. Verify these points first:

- Request timeouts are long enough.
- Response buffering does not swallow streaming flushes.
- Idle connection timeouts are reasonable.
- The proxy supports `text/event-stream`.

### 5.3 Resource planning

Resource usage mainly comes from three places:

- Model inference latency and concurrency
- Time spent calling external systems through tools
- RAG retrieval and index backend access

If you switch from mock tools to real tools, the resource profile changes significantly, especially in outbound connections and upstream rate limiting.

## 6. Health checks and runtime signals

### Basic health check

```bash
curl http://localhost:8880/health
```

### Signals that actually matter in production

- Total HTTP request volume, error rate, and P95/P99 latency
- SSE connection count, connection duration, and interruption rate
- Total Agent latency per turn
- Model call success rate, rate-limit rate, and timeout rate
- Tool call success rate, failure distribution, and average latency
- RAG recall latency, rerank latency, and empty-hit ratio

## 7. Minimal runbook

### Scenario 1: service unavailable

Check in this order:

1. Whether the process exists and the port is listening
2. Whether `config.yaml` and component configs loaded successfully
3. Whether critical environment variables are present
4. Whether `/health` is reachable
5. Whether the logs show startup failure or request-time failure

### Scenario 2: endpoint is up but no answer is returned

Check in this order:

1. Whether the session is valid
2. Whether the model provider is reachable
3. Whether the Agent enters the `think/act/observe` stages
4. Whether tools or RAG are stuck
5. Whether SSE is being truncated by the proxy layer

### Scenario 3: answer quality drops noticeably

Check in this order:

1. Whether the default model changed
2. Whether prompt files changed
3. Whether tools were registered successfully
4. Whether the RAG index is stale or uses a mismatched embedding model
5. Whether new upstream rate limits or provider behavior changes appeared

## 8. Degradation strategy recommendations

The project itself does not yet have a fully mature multi-level degradation workflow, but operationally you can control impact in this order:

1. Disable MCP tools first and keep only internal or mock tools.
2. Disable RAG next and keep pure model-based answers.
3. If model calls are failing, switch to another configured provider.
4. If necessary, keep only session creation and basic health checks, and publicly announce degraded chat capability.

## 9. Known deployment constraints today

- Session and Memory are not persistent, so they are not suitable for shared multi-instance context out of the box.
- The runtime registry stores components by `Component.Name()`, and most components use fixed names, which naturally biases the system toward a single-instance model.
- `/health` is only a process-level liveness check, not a dependency-health check.
- Some config fields exist in schema but may not fully participate in the real execution path, so read them together with the developer docs.
