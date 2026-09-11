# k8s-infra-setup-testing

GitOps source of truth for per-tenant Helm values. This repo holds no
application code and no chart — it's watched directly by an Argo CD
`ApplicationSet`, which renders [`helm-chart-bridge`](../helm-chart-bridge)
against each file here to produce one tenant's live resources.

## Layout

```
tenants/
└── <tenant-slug>.yaml    one file per tenant, e.g. hbss-11.yaml
```

## How a file gets here

Every file under `tenants/` is written by
[`tenant-operator`](../tenant-operator) (`app/services/git_service.py`) —
never by hand, and never directly by Argo CD. The flow:

```
POST /api/v1/tenant  →  tenant-operator writes Vault secrets
                     →  commits tenants/<slug>.yaml to this repo, pushes
                     →  Argo CD's ApplicationSet (git generator) notices
                        the new/changed file and creates/updates/syncs the
                        matching Application automatically
```

Deleting a tenant works the same way in reverse: tenant-operator removes
that tenant's file and pushes; the `ApplicationSet` notices the file is gone
and prunes the `Application` (and, separately, the operator also force-deletes
the tenant's namespace directly, so nothing is left dangling even if the git
side of the deletion is delayed).

**Filename convention**: `<tenant-slug>.yaml`, where slug is
`Tenant.slug` (`<tenant_name>-<sequence-number>`, e.g. `hbss-11`) — **not**
the bare tenant name. `tenant_name` has no database uniqueness constraint,
so two tenants sharing a display name would silently overwrite each other's
file if the bare name were used instead. If you ever see a stale file here
that doesn't match this pattern, it likely predates this convention and is
safe to remove once you've confirmed no live tenant maps to it (check
`GET /api/v1/tenant` for the `slug` field of every current tenant first).

## What's safe to hand-edit here (and what isn't)

These files are documented as **generated — do not edit by hand** for a
reason: tenant-operator overwrites the entire file on the next update to
that tenant, so a manual edit only survives until that happens. That said,
if you do need to patch one out-of-band (e.g. for a one-off test), the chart
this repo feeds distinguishes between fields that are safe to override and
one that is actively dangerous:

- **Safe**: `tenant:` (id/name/namespace/domain), `vault:` (per-cluster Vault
  address), `ingress:` (per-cluster ingress class name), `disabledServices:`
  (which of the chart's ~32 services this tenant runs).
- **Never**: `services:`. It's a Helm **list** in the chart's own
  `values.yaml`, and Helm replaces lists wholesale on override rather than
  merging elements — adding a `services:` block here to change even one
  field on one service silently discards the chart's other ~31 service
  definitions, which breaks any template that looks up a specific service by
  name (e.g. the chart's `erep-pod-configmap.yaml`) and blocks manifest
  generation for the *entire* tenant, not just the field you meant to
  change. Use `disabledServices:` to turn services off, and change a
  service's own fields (replicas, resources, image) in the chart's own
  `values.yaml` instead, as a chart-wide default.

## Example (annotated)

```yaml
tenant:
  id: hbss-11
  name: hbss
  namespace: tenant-hbss-11
  domain: hbss-bridgestg.example.internal   # single Ingress host for this tenant

imagePullSecrets:
  - ocir-regcred

# Per-cluster override -- this cluster's Vault runs on a different cluster
# than the tenant's own workloads, so the chart's in-cluster DNS default
# doesn't resolve here.
vault:
  server: http://<vault-host>:<port>

# Per-cluster override -- this cluster's ingress-nginx controller was
# installed with a non-default --ingress-class.
ingress:
  className: stg-spoke-nginx

# The only sanctioned lever for which of the chart's ~32 services this
# tenant runs -- see "What's safe to hand-edit" above.
disabledServices:
  - voxflow
  - prism-ui
  - acl-server
```

## Verifying a change before it reaches a tenant

Since Argo CD's repo-server does nothing more than `helm template` against
whatever's committed here, you can reproduce exactly what it will do,
locally, before pushing:

```bash
helm template <tenant-slug> ../helm-chart-bridge \
  -f tenants/<tenant-slug>.yaml \
  --namespace tenant-<tenant-slug> --include-crds
```

A clean exit (0) means Argo CD will render it cleanly too; any Helm error
here is the same error you'd otherwise only discover via a stuck/`OutOfSync`
`Application` on the cluster.
