# Troubleshooting

This page is organized around practical fault paths instead of component ownership. When the system fails, the first question is usually not "which package is wrong", but "where did the request path break".

## 1. The service does not start

Check in this order:

1. Whether `.env` exists and required keys are present
2. Whether paths in `config.yaml` are correct
3. Whether the component YAML files are syntactically valid
4. Whether unknown fields were rejected by strict decode
5. Whether the target port is already in use

Common causes:

- Typo in `config.yaml`
- Invalid `level` in logger config
- Wrong prompt path in Agent config
- Missing or invalid model config
- Port conflict on `8880`

## 2. `/health` works but chat does not

This usually means the HTTP service is alive, but one of the deeper dependencies is not.

Check in this order:

1. Whether session creation works
2. Whether the session used by chat is valid
3. Whether the model provider is reachable
4. Whether Agent initialization completed successfully
5. Whether the request includes `Accept: text/event-stream`

Common causes:

- Invalid or expired `sessionID`
- No working API key for the default model
- Model provider networking problems
- Agent failed during flow construction
- Client is treating SSE like normal JSON

## 3. The streaming endpoint returns no output

Check:

- Whether you used `curl -N` or an SSE-capable client
- Whether the request includes `Accept: text/event-stream`
- Whether the server emitted an `error` event
- Whether the model provider is reachable and returning data
- Whether a proxy buffered or cut the connection

A common mistake is assuming "HTTP 200" means the whole path is healthy. On a streaming path, the real issue may happen after headers are sent.

## 4. The frontend page opens but cannot chat

Check these first:

- Whether the backend is running on `http://localhost:8880`
- Whether the frontend is opened from `http://localhost:8881/admin`
- Whether Vite proxying for `/api/v1/ai` is working
- Whether session creation fails before chat starts

This kind of issue often looks like a UI problem but is really an API connectivity issue.

## 5. Sessions can be created, but the model seems unavailable

Check:

- Whether `default_model` in `component/models/models.yaml` is correct
- Whether the matching provider has a real `api_key`
- Whether `base_url` is reachable
- Whether the provider was skipped during initialization

Typical symptoms:

- Service boots successfully
- Session APIs work
- Chat returns errors or no useful output

That pattern usually means the HTTP layer is fine, but Models is not actually usable.

## 6. The Agent fails during initialization

Check:

- Whether `component/agent/agent.yaml` points to a valid model
- Whether `prompt_base_path` exists
- Whether every `prompt_file` exists
- Whether stage config matches expected flow types and output structure

Typical symptoms:

- Startup completes partially
- The Agent component logs initialization errors
- Chat requests fail before any real stream is produced

## 7. Conversation context feels broken

Check:

- Whether you are reusing the same `sessionID`
- Whether Memory is being cleared unexpectedly
- Whether the service restarted and lost in-process state
- Whether `NextTurn(sessionID)` is being reached after each interaction

Important fact:

- Memory is in-process only. A restart resets context.

## 8. RAG retrieval quality is poor

Check in this order:

1. Whether the index was actually built
2. Whether the embedding model used for indexing matches retrieval expectations
3. Whether chunk size and overlap are reasonable
4. Whether top-k is too small or too large
5. Whether tool invocation actually triggered retrieval
6. Whether the prompt integrated the retrieved content correctly

Poor answer quality is not always a model problem. In RAG-heavy paths, the problem may be data quality, chunking, indexing, retrieval, or orchestration.

## 9. Tool calls do not happen when expected

Check:

- Whether the relevant tools were registered successfully
- Whether the stage has `enable_tools` enabled
- Whether the prompt clearly tells the model when to call tools
- Whether tool names and schemas are understandable to the model

This is often a Tools plus Prompt problem, not just an Agent problem.

## 10. A useful troubleshooting rule

When debugging Dubbo Admin AI, split the system into five checkpoints:

1. Config loaded
2. Component initialized
3. Session accepted
4. Agent entered the loop
5. Model or tool produced usable output

If you can identify the first checkpoint that fails, the investigation scope becomes much smaller.
