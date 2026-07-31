# devops-task-gitops

Everything ArgoCD reconciles for the DevOps take-home assessment - a generic Helm chart, per-cluster
ArgoCD `AppProject`/`Application`/`ApplicationSet` definitions, and the platform manifests
(namespace/quota/network-policy) that live here instead of in Terraform.

Part of a 3-repo submission:

- [devops-task-infra](https://github.com/yoavmanoim-prog/devops-task-infra) - Terraform/Terragrunt IaC (provisions the clusters, installs ArgoCD, points it at this repo)
- **gitops** (this repo)
- [devops-task-app](https://github.com/yoavmanoim-prog/devops-task-app) - FastAPI sample app + CI/CD (the only thing that actually writes to this repo's `apps/*/values-*.yaml` files - see "Promotion flow" below)

## Ownership handoff

Terraform (in `infra`) provisions each cluster, its controllers, ArgoCD itself, and exactly one
bootstrap `Application` per cluster pointing at this repo. Everything below that line - namespaces,
quotas, network policies, AppProjects, the actual workload - is reconciled by ArgoCD from here on,
never re-applied by Terraform.

## Structure

```
gitops/
├── charts/app/          # generic, reusable Helm chart (Deployment/Service/Ingress/ConfigMap/ExternalSecret/HPA/ServiceAccount)
├── apps/dev/             # dev cluster: AppProject, platform Application, ApplicationSet (fans out to dev+staging), values files
├── apps/prod/            # prod cluster: AppProject, platform Application, single manual-sync Application, values file
└── platform/{dev,prod}/  # Namespace + ResourceQuota + NetworkPolicy per namespace
```

## Promotion flow

- **`dev` namespace** (dev cluster): `app`'s CI pushes an image on every commit to its `dev`
  branch, then bumps `apps/dev/values-dev.yaml` directly. The `ApplicationSet` here has
  `automated: {prune: true, selfHeal: true}` - it picks the new tag up on its own.
- **`staging` namespace** (same dev cluster, different namespace): a PR within `app` merging
  `dev` → `staging` triggers CI to re-tag the *same* image (no rebuild) into
  `apps/dev/values-staging.yaml`. Auto-syncs the same way.
- **`production` namespace** (prod cluster): a PR within `app` merging `staging` → `prod` writes
  the same tag into `apps/prod/values-production.yaml` - but that `Application` has **no**
  `automated:` sync policy. The tag lands in git immediately; a human still has to click Sync in
  ArgoCD before it actually deploys.

Nobody edits `apps/*/values-*.yaml` by hand in the normal flow - they're written by `app`'s CI
(`patch_gitops_values.py`, see that repo). The values files currently point at a pinned public
placeholder image (`public.ecr.aws/nginx/nginx`, never `:latest`) specifically so these
Applications can sync before `app`'s CI has ever run for real.

## Chart values contract

Every values file under `apps/*/` only overrides what's actually different for that namespace -
everything else falls back to `charts/app/values.yaml`'s defaults:

| Overridden per namespace | Left at chart default |
|---|---|
| `image.repository`/`image.tag` (owned by CI, see above) | `podSecurityContext`/`securityContext` (non-root, read-only rootfs) |
| `replicaCount`, `resources` | `service.port`/`targetPort` |
| `ingress.host`, `ingress.annotations.scheme` (opt-in to `internet-facing` - see below) | `ingress.annotations.target-type`/`listen-ports` |
| `env`, `configMap.data` | - |
| `externalSecret.remoteRef.key` (per-env secret path) | `externalSecret.secretStoreRef` |

`image.tag` has no chart default at all - `helm template`/`install` fails closed with an explicit
error rather than silently falling back to any implicit tag.

## Known limitations

- **~~Ingress defaulted to public exposure~~ - fixed.** A security-review pass caught that the
  chart's `ingress.annotations` scheme defaulted to `internet-facing` with no per-env override -
  every environment silently inherited public exposure, backwards from secure-by-default for a
  "generic, reusable chart." Fixed: the chart now defaults to `internal`; each of the 3 env values
  files explicitly opts back into `internet-facing` (this demo genuinely wants external ALB
  routing exercised, per spec 3.3) - public exposure is now a visible, per-env, auditable choice
  instead of an inherited default.
- **NetworkPolicy enforcement is not verified.** `platform/dev/networkpolicy-{dev,staging}.yaml`
  are what's meant to keep the `dev` and `staging` namespaces apart on their one shared cluster,
  but enforcement depends on the VPC CNI's network policy feature, which nothing in the `infra`
  repo explicitly turns on - as written, these policies are accepted by the API but their
  real-world enforcement is unconfirmed.
- **AppProjects are wide open** (`namespaceResourceWhitelist: "*"/"*"`) - reasonable for a demo,
  would narrow this for anything real.
- **Resource quota sizing** (`platform/*/resourcequota-*.yaml`) is round numbers picked for a
  demo, not derived from any real workload estimate.
- **`ExternalSecret` uses `dataFrom.extract`** (pulls the whole AWS secret as one JSON blob) rather
  than per-key mapping - a genericity trade-off for a reusable chart, not appropriate for every
  secret shape.
