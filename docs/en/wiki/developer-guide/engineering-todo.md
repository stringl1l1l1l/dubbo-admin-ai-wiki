# Engineering TODO

This page tracks architecture and engineering work that is still worth pushing forward. It is not meant for small day-to-day task notes.

## Documentation and Information Architecture

- [ ] Build a shared terminology glossary across Chinese and English documentation
- [ ] Add implementation-status markers on key pages to separate shipped features from planned work
- [ ] Expand API and SSE event examples

## Runtime and Architecture

- [ ] Register runtime components by instance name instead of a fixed component name
- [ ] Define a clear `Start()` / `Stop()` order to reduce dependency ambiguity
- [ ] Provide a reusable persistence model for Session and Memory

## Agent and Tooling

- [ ] Add finer-grained stage observability and latency statistics
- [ ] Complete fallback paths for tool invocation failures
- [ ] Define clearer safety policies for high-risk tools

## RAG Subsystem

- [ ] Clarify which config fields are implemented and which are only reserved
- [ ] Build a stable incremental indexing workflow
- [ ] Establish a regression-friendly retrieval-quality baseline

## Testing and Quality

- [ ] Add tests for SSE event ordering and failure paths
- [ ] Add regression tests for multi-provider configuration
- [ ] Establish performance baselines and alert thresholds for critical paths
