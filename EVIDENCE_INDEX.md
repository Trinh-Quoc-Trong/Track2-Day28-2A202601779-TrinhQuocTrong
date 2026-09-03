# Evidence Index

## Automated results

| Evidence | Artifact |
|---|---|
| Fast and live test output | `fast-suite-output.txt` |
| Integration score | `integration-report.json` |
| Load profile and analysis | `load-profile.json` |
| Failure and no-data-loss record | `failure-recovery.md` |
| Kubernetes/GitOps validation | `gitops-validation.md` |

The machine-readable IP01-IP10 artifacts are packaged in
`evidence-bundle.zip`. Screenshots are under `submission-screenshots/`.

## Screenshot map

| Files | Proof |
|---|---|
| `01`-`03` | Tests, evidence inventory and architecture |
| `04`-`06` | Happy-path IDs, Airflow success and Delta versions |
| `07`-`08` | Replay safety, Feast and Qdrant |
| `09`-`10` | MLflow promotion, rollback and champion provenance |
| `11`-`12` | Failure recovery, no data loss and gateway rate limit |
| `13`-`15` | Grafana, Prometheus and Jaeger |
| `16`-`17` | Load profile and LangSmith trace |

## Environment-gated evidence

- **IP07 vLLM:** `UNVERIFIED`. No reachable real GPU endpoint was available.
  `evidence/ip07-vllm-identity.json` records the failed real-vLLM gate.
- **Live GitOps:** `UNVERIFIED`. Manifests passed static validation, but this
  machine has no Kubernetes context or Argo CD runtime.

No token, temporary privileged URL, database, cache or model weight is included.
