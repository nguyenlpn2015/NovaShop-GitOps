# NovaShop GitOps

This repository is the authoritative deployment state for NovaShop.

Application source code, Dockerfiles, CI workflows, and the Helm chart remain
in [`NovaShop`](https://github.com/nguyenlpn2015/NovaShop). Argo CD watches this
repository for desired-state changes and deploys no application resources
directly from the application repository.

## Repository Structure

```text
NovaShop-GitOps/
├── .github/
│   └── CODEOWNERS
├── apps/
│   └── novashop/
│       ├── values/
│       │   ├── development.yaml
│       │   ├── staging.yaml
│       │   └── production.yaml
│       └── targets/
│           └── ubuntu-k3s/
│               ├── development.yaml
│               ├── staging.yaml
│               └── production.yaml
├── clusters/
│   ├── base/
│   │   └── novashop-applicationset.yaml
│   ├── in-cluster/
│   │   └── kustomization.yaml
│   └── ubuntu-k3s/
│       ├── phases/
│       │   ├── http/
│       │   └── tls/
│       │       ├── cert-manager-application.yaml
│       │       ├── certificates-application.yaml
│       │       └── platform-project.yaml
│       └── kustomization.yaml
├── docs/
│   └── OPERATIONS.md
├── .gitignore
└── README.md
```

## Deployment Contract

- Container images use immutable source commit tags.
- The application Helm chart is pinned to an immutable Git revision.
- Shared defaults remain in the application Helm chart.
- This repository stores only environment-specific overrides.
- Development, staging, and production use isolated namespaces and Secrets.
- Runtime database and Redis Secret values are provisioned externally.
- Ubuntu bootstrap reconciles HTTP only and does not install cert-manager.
- cert-manager and TLS are activated only by a separate reviewed GitOps change;
  no private key is stored in Git.
- Every deployment and rollback is performed through a reviewed pull request.
- CI may open deployment pull requests but may not write to the default branch.

## Environments

| Environment | Namespace | Values |
|-------------|-----------|--------|
| Development | `novashop-development` | `development.yaml` |
| Staging | `novashop-staging` | `staging.yaml` |
| Production | `novashop-production` | `production.yaml` |

## Reconciliation

The shared `ApplicationSet` generates one Argo CD `Application` per
environment. The `in-cluster` overlay preserves Docker Desktop local access.
The `ubuntu-k3s` overlay defaults to `phases/http`, which adds only public HTTP
edge resources. `phases/tls` is repository-integrated but inactive; selecting
it is an explicit platform change. Its first rollout uses Let's Encrypt staging
before a separate production certificate promotion. Automatic sync,
self-healing, pruning, bounded retry, and revision history remain consistent.

Operational deployment, synchronization, and rollback procedures are defined
in [Operations](docs/OPERATIONS.md).
