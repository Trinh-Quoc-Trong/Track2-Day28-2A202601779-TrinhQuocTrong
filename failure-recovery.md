# Failure and Recovery Record

## Test result

The required live suite completed with `56 passed, 16 deselected`. The dedicated
recovery journey completed with `9 passed, 4 deselected`. Deselected tests require
a real vLLM GPU endpoint; no mock endpoint was used.

## Promotion and rollback

The MLflow journey registered a release with provenance, promoted it through the
`champion` alias, resolved the serving release, and rolled the alias back without
changing code. The final submission release is `lab28-rag-release` version `7`,
run `4f5a2967576049208ffd0935ceb84791`, evaluated on Delta version `19`.

## Dependency recovery

Stopping Feast produced the predicted `degraded` verdict because it is optional
for the tested decision path. Stopping Qdrant produced fail-closed `not_ready`
because retrieval is mandatory. Both dependencies were restored in `finally`
blocks, and the final platform state matched the baseline.

## No data loss

One batch contained an invalid record and a valid record. The invalid payload was
parked on `data.raw.dlq`; the valid event still reached Delta. Replay re-injected
the valid dead-letter envelope, rejected an unparseable envelope, and Delta
retained exactly one row for the idempotency key. A separate replay journey also
proved that Feast counted the fact once and Qdrant retained one point.
