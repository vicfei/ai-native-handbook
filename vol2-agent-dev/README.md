# Volume 2 · Agent Application Developer Handbook

[中文](README.zh.md)

> Status: skeleton. Chapters below are the migration plan for first-hand practice records, ordered by dependency.

## rag/ — Enterprise RAG

Centered on productionizing an open-source RAG platform (WeKnora family):

| Planned chapter | Contents |
|---|---|
| Platform fork governance | official-first merge strategy, migration tooling (Dify→WeKnora mapping), tag-tree semantics |
| The retrieval path | embedding/reranker deployment shapes; task semantics (embedding = bi-encoder recall / score = cross-encoder rerank); score-semantics acceptance tests |
| Java-side integration | auth combinations (login-Bearer vs scoped-key fallback; key+HMAC minted external tokens), SSE citation contracts |
| Production deployment | multi-environment matrix (staging/prod/DMZ), managed-DB migration, NAS file layer, breaking-change governance for parser services |
| Data operations | bulk-ingestion throughput calibration, summary-flood governance, cleaning data-sheet noise from retrieval |

## harness/ — Enterprise Harness

Centered on DeepSeek Harness (DSH) in the enterprise:

| Planned chapter | Contents |
|---|---|
| Deployment | private deployment shapes; Linux toolchain traps (Windows pnpm interop panic, dual-side port checks); the profile system |
| Plugin development | the cordis patch mechanism, MCP client wiring, complementary design of the official read-only plugin and a first-party write-path plugin (three-layer safety model) |
| Model wiring | public API vs local vLLM dual forms; secret management (DSH scrubs `*KEY*` env vars in agent shells — the config-file channel) |
| Load governance | small/large-model traffic split (gateway vs plugin vs default flip); measured limits of routing gains (finding: no congestion, no benefit) |

## apps/ — Application scenarios

Citation-grounded Q&A, dual-submission pipelines, retrieval-audited investment research (the "factor authentication" pattern: separating the generator from the judge), and other engineering records.
