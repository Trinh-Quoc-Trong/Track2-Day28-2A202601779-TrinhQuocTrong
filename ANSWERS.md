# Lab 28 - Submission Answers

## Architecture and ownership

The architecture diagram is
[`docs/images/lab28-architecture-overview.png`](docs/images/lab28-architecture-overview.png).
This is an individual submission. I completed all four operating roles:

| Role | Integration points | Responsibility |
|---|---|---|
| Ingestion | IP01-IP02 | API, Kafka contract, Airflow orchestration |
| Data | IP03-IP04, IP06 | Delta, Feast, MLflow release lifecycle |
| Serving | IP05, IP07 | Qdrant retrieval and vLLM boundary |
| Platform | IP08-IP10 | Gateway, metrics, tracing and deployment contracts |

## Technical decisions and trade-offs

- Kafka retains at-least-once delivery; deterministic Delta `MERGE` provides
  idempotency without discarding broker history.
- Events carry byte-valued idempotency headers and optional W3C `traceparent`.
  A bounded group-assignment wait avoids false empty Airflow runs.
- Feast uses registry-owned feature references and exposes missing or stale
  values. Optional failures become `degraded`; mandatory failures become
  `not_ready`.
- Qdrant uses stable point IDs and a pinned embedding revision. The initial
  download is slower, but re-indexing is repeatable and duplicate-safe.
- MLflow stores prompt, model, embedding, retrieval and Delta provenance in an
  immutable release. The mutable `champion` alias enables promotion and rollback
  without code changes.
- LangSmith export is opt-in through the OpenTelemetry Collector. The default
  stack remains offline-capable and stores no external credential.

## Verification

| Gate | Result |
|---|---|
| Starter suite | `4 passed` |
| Fast suite | `83 passed` |
| Integration matrix | `245 checks passed` |
| Required live suite | `56 passed, 16 deselected` |
| Golden path | `12 passed, 3 deselected` |
| LangSmith gate | `1 passed`; project `lab28-platform` received traces |
| GPU/vLLM | `UNVERIFIED`: no reachable real GPU endpoint |
| Kubernetes reconciliation | `UNVERIFIED`: no cluster or Argo CD runtime |

Latest submission evidence:

- Kafka entity `it-j1-be3e0a45`, trace `f899698f69b84d7bbaf8879831a4b331`.
- Airflow run `it-d6a70e86`, state `success`.
- Delta feedback version `20`, 30 rows; Feast materialized version `20`.
- Qdrant collection `lab28_documents`, 21 points.
- MLflow `lab28-rag-release` version `7` is `champion`; release Delta version
  is `19`, run `4f5a2967576049208ffd0935ceb84791`.
- Jaeger continuity trace `f835ce1c7d3f4d778310ee26f616fa80` contains 11
  spans across gateway, API, Kafka, Airflow and Spark.

## Failure and recovery

The recovery journey stopped Feast and observed `degraded`, then stopped Qdrant
and observed fail-closed `not_ready`. Both services were restored in `finally`
blocks. An invalid Kafka record was parked in the DLQ while a valid record from
the same batch reached Delta. Replay restored the valid event and Delta retained
one logical row. Details are in `failure-recovery.md`.

## Load profile

The baseline returned 50/50 HTTP 200 responses with P50/P95/P99 of
271.27/297.27/538.90 ms. An eight-worker burst returned 22 HTTP 200 and 178
intentional HTTP 429 responses with P50/P95/P99 of 4.14/374.72/435.60 ms. The
10 RPS gateway protects downstream dependencies; deep readiness checks dominate
accepted-request latency and should be separated from lightweight pod probes.

## Reflection

- **Hardest part:** preserving one trace and idempotency contract across HTTP,
  Kafka, Airflow and Delta while keeping replay safe.
- **Chosen trade-off:** retain every Kafka delivery and deduplicate at the Delta
  boundary. This uses more broker storage but preserves auditability.
- **Next improvement:** deploy replicated stateful services, workload identity,
  mTLS, managed secrets and a real GPU endpoint; then validate Argo CD self-heal
  and rollback on a live cluster.

## Production gaps

- Local Kafka, MLflow, Feast, Qdrant and Airflow are single-node services.
- Internal traffic uses lab credentials and plaintext networking.
- IP07 remains unverified because no real vLLM GPU endpoint was available.
- Kubernetes manifests pass static validation, but live drift repair and rollback
  were not exercised.
- The embedding runtime warning requires a pinned library/model regression test
  before rebuilding a production index.

## Contribution

I implemented the four student-owned integration functions, corrected the Kafka
assignment race and load-test HTTP 429 accounting, added credential-safe
LangSmith export, validated the deployment contracts, and produced the reports
and evidence bundle. No mock vLLM, fabricated trace or synthetic evidence is
claimed.
