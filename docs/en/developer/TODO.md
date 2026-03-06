# TODO List

This page records items that are still worth advancing at the architecture and engineering level. It is not for daily small tasks.

## Documentation and information architecture

- [ ] Build a unified glossary for Chinese and English documentation
- [ ] Add implementation-status labels to key pages so shipped capabilities and planned capabilities are clearly separated
- [ ] Add more complete examples for API and SSE events

## Runtime and architecture

- [ ] Make Runtime register components by instance name rather than fixed component name
- [ ] Make `Start()` and `Stop()` ordering explicit to reduce dependency uncertainty
- [ ] Provide a shareable persistent solution for Session and Memory

## Agent and tool system

- [ ] Add finer-grained stage observability and latency metrics
- [ ] Complete degradation paths after tool invocation failures
- [ ] Design clearer security policy for high-risk tools

## RAG subsystem

- [ ] Clarify the boundary between implemented config fields and reserved fields
- [ ] Build a stable incremental indexing workflow
- [ ] Establish a regression-friendly evaluation baseline for retrieval quality

## Testing and quality

- [ ] Add tests for SSE event order and error paths
- [ ] Add regression tests for multi-provider configuration
- [ ] Establish key-path performance baselines and alert thresholds
