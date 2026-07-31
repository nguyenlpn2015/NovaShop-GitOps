# GitOps Operations

## Deployment

1. Confirm the NovaShop release workflow published both GHCR images with the
   same source commit SHA.
2. Update the backend and frontend image tags in the target environment values
   file.
3. Validate the Helm render and Kubernetes schemas.
4. Open a pull request with the source revision, target environment, and
   validation evidence.
5. Merge after required review and successful checks.
6. Confirm the Argo CD application reaches `Synced` and `Healthy`.

Promotion copies the exact image SHA from development to staging and then to
production. Images are never rebuilt between environments.

## Synchronization

Argo CD polls the GitOps default branch and reconciles changes automatically.
Self-heal restores resources changed outside Git. Prune removes resources that
are removed from the desired state. Failed synchronization uses bounded
exponential retry.

| Application | Desired state |
|-------------|---------------|
| `novashop-development` | `development.yaml` |
| `novashop-staging` | `staging.yaml` |
| `novashop-production` | `production.yaml` |
| `novashop-cert-manager` | Pinned cert-manager chart and ClusterIssuer |
| `novashop-certificates` | Environment Certificate resources |

Kubernetes workload probes and Argo CD resource health must both pass before an
application is considered healthy.

## Rollback

1. Identify the last known healthy GitOps commit or image SHA.
2. Revert the failed deployment commit.
3. Open and approve the rollback pull request.
4. Merge and observe automatic reconciliation.
5. Confirm `Synced` and `Healthy`.
6. Record the failed revision and rollback revision in the incident timeline.

An emergency rollback from the Argo CD revision history is permitted only
during active incident mitigation. Commit the restored state to Git
immediately after service recovery.

Database recovery is independent of application rollback. Schema changes must
remain compatible with the previous application revision.

## Drift

Manual changes to managed resources are prohibited. Argo CD self-heal will
restore the Git state. Emergency cluster changes must be followed by a pull
request that either codifies or explicitly reverses the change.

## Secrets

The database and Redis Secret referenced by each environment must exist before
the first workload becomes Ready. This runtime Secret is delivered by the
platform and is not committed to this repository.

TLS Secrets are generated and renewed by cert-manager from declarative
`Certificate` resources. Do not create or edit their key material manually.

## Future Release Automation

A future NovaShop release workflow may authenticate with a short-lived GitHub
App token, update the target values file, run validation, and open a pull
request in this repository.

The workflow must:

- use the immutable `${GITHUB_SHA}` image tag;
- update backend and frontend references explicitly;
- include release provenance in the pull request;
- require normal branch protection and review;
- never push directly to `main`;
- never deploy with `kubectl`, Helm install, or the Argo CD API.
