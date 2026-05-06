# 02 — llama-server Load Test Results

Runtime: `python -m llama_cpp.server` via `make serve`.

Note: OpenAI-compatible `/v1/chat/completions` worked. Prometheus `/metrics` was not available in this runtime and returned `{"detail":"Not Found"}`.

| Concurrency | Total RPS | E2E P50 (ms) | E2E P95 (ms) | E2E P99 (ms) | Failures | Source |
|--:|--:|--:|--:|--:|--:|---|
| 10 | 0.25 | 31000 | 38000 | 38000 | 0 | `submission/screenshots/04-locust-10.png` |
| 50 | 0.25 | 20000 | 43000 | 43000 | 0 | `submission/screenshots/05-locust-50.png` |

Locust response time percentiles are full response latency, not separate TTFB.
