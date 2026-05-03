# Agent Tools

Static tool descriptors should not require loading full channel or plugin
runtimes.

## Guardrails

- Message-tool discovery should flow through shared discovery helpers and
  lightweight channel artifacts before falling back to a full channel plugin
  load.
- Channel-specific tool schemas, action lists, and static capabilities belong
  in plugin-owned helpers that are reused by both the full plugin and the
  lightweight artifact.
- If a code path starts paying multi-second import/setup cost, split the static
  descriptor path from runtime execution instead of routing more callers
  through the broad import.

## Verification

- Run `pnpm build` when adding or changing bundled plugin artifacts.
