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
│       │   ├── http/           break-glass only; prunes certificates
│       │   ├── tls-baseline/   rollback target; HTTPS kept, HSTS released
│       │   │   ├── cert-manager-application.yaml
│       │   │   ├── certificates-application.yaml
│       │   │   └── platform-project.yaml
│       │   └── tls-enforced/   production; redirect and HSTS
│       └── kustomization.yaml
├── docs/
│   └── OPERATIONS.md
├── .gitignore
├── LICENSE
└── README.md
```

Licensed [MIT](LICENSE), matching
[`NovaShop`](https://github.com/nguyenlpn2015/NovaShop/blob/main/LICENSE). The
platform half is the part of this project most worth reusing, so it carries the
same permissive terms as the application half rather than defaulting to reserved
rights.

## Deployment Contract

- Container images use immutable source commit tags, pinned **per environment**.
- The application Helm chart is pinned to an immutable Git revision, **shared by
  all three environments** — see [Promotion model](#promotion-model).
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
The `ubuntu-k3s` overlay selects `phases/tls-enforced`. Certificate resources
use Let's Encrypt production, HTTP permanently redirects to HTTPS, and HTTPS
responses enforce HSTS and the approved security-header policy. Automatic sync,
self-healing, pruning, bounded retry, and revision history remain consistent.

Rollback targets `phases/tls-baseline`, which keeps cert-manager, the
certificates, and HTTPS while releasing enforcement. Every pull request here is
validated by the same engine that gates the application repository; see
[Operations](docs/OPERATIONS.md).

## Promotion model

Two things are pinned, and they are pinned at different scopes. Knowing which is
which is the difference between a staged rollout and an accidental
three-environment deployment.

| Pinned thing | Where | Scope |
|---|---|---|
| Image tag (`backend`/`frontend`) | `apps/novashop/values/<env>.yaml` | **Per environment** |
| Chart revision (`targetRevision`) | `clusters/base/novashop-applicationset.yaml` | **Shared by all three** |

`targetRevision` lives in `spec.template` of the `ApplicationSet`, so every
generated `Application` resolves the same value. That is a deliberate
consequence of generating three Applications from one template, and it has a
consequence of its own:

- **An image change can be promoted one environment at a time.** Bump
  `development.yaml`, verify, then `staging.yaml`, verify, then
  `production.yaml`. One commit each, so a revert costs one environment.
- **A chart change cannot.** Bumping `targetRevision` moves development, staging
  and production on the same reconcile. There is no staged rollout for the chart
  as this repository is currently shaped.

### The rule that follows

When the application commit contains **no change under `helm/`**, do not touch
`targetRevision`. Bump only the image tags, one environment per pull request.
The chart pin still describes what is running, because the chart did not change:

```sh
git -C ../NovaShop diff --name-only <pinned-revision> <new-revision> -- helm/
```

An empty result means the chart is byte-identical and `targetRevision` must be
left alone. This is the normal case for an application release.

When the chart **has** changed, bumping `targetRevision` is unavoidable and all
three environments move together. Land it deliberately, in its own pull
request, with nothing else in it — and expect to verify three environments
rather than one.

### If staged chart rollout becomes necessary

Move `targetRevision` out of `spec.template` and into the list generator
elements, so each environment carries its own chart pin alongside its own values
file. That is the smallest change that buys staged chart promotion. It is not
done today because no chart change has yet needed it, and an unused mechanism is
a mechanism that will be wrong when it is first used.

Operational deployment, synchronization, and rollback procedures are defined
in [Operations](docs/OPERATIONS.md).
