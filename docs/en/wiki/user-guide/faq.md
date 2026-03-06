# FAQ

## Does `/health` prove the whole system is healthy?

No. It only proves the HTTP service is up. It does not prove model providers, MCP tools, or vector backends are healthy.

## Why does the service start even when chat does not work?

Because some providers without usable API keys are skipped instead of crashing the whole process. The service can be alive while capabilities are incomplete.

## Why does conversation history disappear after a restart?

Because Session and Memory are currently stored in process memory, not in persistent storage.

## Why is the field named `sessionID` instead of `session_id`?

Because that is what the current streaming request schema expects. Follow the API contract exactly.
