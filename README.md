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
│       └── values/
│           ├── development.yaml
│           ├── staging.yaml
│           └── production.yaml
├── clusters/
│   └── in-cluster/
│       └── novashop-applicationset.yaml
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
- Secret values are provisioned externally and are never stored in Git.
- Every deployment and rollback is performed through a reviewed pull request.
- CI may open deployment pull requests but may not write to the default branch.

## Environments

| Environment | Namespace | Values |
|-------------|-----------|--------|
| Development | `novashop-development` | `apps/novashop/values/development.yaml` |
| Staging | `novashop-staging` | `apps/novashop/values/staging.yaml` |
| Production | `novashop-production` | `apps/novashop/values/production.yaml` |

## Reconciliation

The in-cluster `ApplicationSet` generates one Argo CD `Application` per
environment. Automatic sync, self-healing, pruning, bounded retry, and revision
history are configured consistently for every environment.

Operational deployment, synchronization, and rollback procedures are defined
in [Operations](docs/OPERATIONS.md).
