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
| `novashop-cert-manager` | TLS phase only: pinned chart and ClusterIssuers |
| `novashop-certificates` | TLS phase only: environment Certificate resources |

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

TLS Secrets are absent in the default HTTP phase. After a reviewed TLS
activation, they are generated and renewed by cert-manager from declarative
`Certificate` resources. Do not create or edit their key material manually.

## Edge Phases

The root `clusters/ubuntu-k3s/kustomization.yaml` selects exactly one phase.
Bootstrap observes that selection and never changes it.

| Phase | Certificates | HTTP | HSTS | Role |
|-------|--------------|------|------|------|
| `phases/tls-enforced` | issued | redirects to HTTPS | `max-age=31536000` | Production |
| `phases/tls-baseline` | issued | answers directly | `max-age=0` | **Rollback target** |
| `phases/http` | **pruned** | answers directly | absent | Break-glass only |

Each phase differs from the one above it by a single concern, so a change of
phase is a one-line reviewed revert of the root kustomization.

## Rollback Ladder

Roll back to `phases/tls-baseline`. cert-manager, the `Certificate` resources,
and HTTPS all remain in place; only enforcement is released, and
`Strict-Transport-Security: max-age=0` actively clears the pin that browsers
were given. No certificate is pruned and no issuance quota is consumed.

`phases/http` is not a rollback target. It removes the certificate resources
from the desired state, and with `prune: true` Argo CD deletes them. Because
production advertises HSTS with a one-year `max-age`, returning browsers would
refuse plaintext outright rather than degrade. Let's Encrypt permits five
duplicate certificates per identical hostname set per 168 hours, so recovering
from that choice can take a week. Use it only after browsers have already
observed `max-age=0` from the baseline phase, and only with a certificate backup
taken beforehand.

No imperative Helm or `kubectl apply` command is part of any phase change.

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
