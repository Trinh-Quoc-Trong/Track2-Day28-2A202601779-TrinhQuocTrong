# Kubernetes and GitOps validation

## Static validation: PASS

`scripts/validate_manifests.py` passed. The base includes Deployment, Service,
ServiceAccount, ConfigMap, HPA, PDB, NetworkPolicy, Gateway and HTTPRoute. The
validator also confirmed non-root execution, resource declarations, probes,
container security contexts, stable Gateway API v1, and a pinned Argo CD
revision.

The Argo CD application targets `refs/tags/v3.0.0`, enables automated pruning
and self-healing, and retains five revisions for rollback.

## Live drift and rollback: UNVERIFIED

`kubectl` is installed, but this machine has no current Kubernetes context;
`kind` and the Argo CD CLI are unavailable. Therefore no live reconciliation,
drift repair or cluster rollback was claimed. The manifests are statically
verified only.
