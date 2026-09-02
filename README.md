# harbor-operator

A Kubernetes operator that gives **any namespace its own private Harbor registry, on request**.
A user creates one small resource; the operator works out whether a Harbor already exists in that
namespace, deploys one if not, sizes it, and carves out a project with credentials.

```yaml
apiVersion: registry.opencloud.wso2.com/v1alpha1
kind: Registry
metadata:
  name: web
spec:
  plan: starter
```

That is the entire user-facing API. There is no field naming a Harbor deployment — the one serving a
`Registry` is always the one in its own namespace, so a `Registry` cannot reach another namespace's
images because there is no field in which to name one.

---

## Why this repository exists

This code was originally written for WSO2's Open Cloud Datacenter platform, where the design was
**one Harbor deployment per namespace**. That platform has since moved to a different model: a
single centralised Harbor shared by every cluster, with a project created per tenant.

The per-namespace model is not obsolete — it is a genuinely different set of trade-offs, and the
better choice for some situations:

| | Harbor per namespace (this operator) | One shared Harbor |
|:---|:---|:---|
| Blast radius of a Harbor bug | one namespace | every tenant |
| Data separation | separate database, blob store, admin credentials | logical, inside one Harbor |
| Noisy-neighbour isolation | complete | shared |
| Cost | a full Harbor stack per namespace (~29 GiB provisioned before a single image) | one stack |
| Operational surface | N Harbors to upgrade, back up, monitor | one |

**Choose this operator when tenant isolation matters more than cost** — regulated workloads,
untrusted tenants, or anywhere a shared control plane is unacceptable. Choose a shared Harbor when
cost and operational simplicity dominate.

This repository preserves the per-namespace implementation as a standalone, working operator so it
stays usable independently of the platform it was written for.

---

## What it does

- **Two CRDs.** `Registry` is what a user creates. `RegistryBackend` is the Harbor deployment the
  operator manages on their behalf — a namespaced singleton, always named `harbor`, that users never
  author.
- **Level-triggered reconciliation.** Every pass re-asserts desired state and no-ops when already
  correct. There is no change-event tracking anywhere.
- **Helm-driven deployment.** Harbor is installed from the upstream chart, with values diffed against
  the deployed release so an unchanged pass costs nothing.
- **Pinned credentials.** Every secret the chart would otherwise regenerate is created once into an
  operator-owned Secret and never overwritten, which is what makes `helm upgrade` safe.
- **Storage autoscaling.** The deployment grows across three plan tiers as registries commit quota.
  Growth is recorded in status, never written back into the spec.
- **Deletion guards.** A backend refuses to delete while `Registry` objects still depend on it;
  overriding that takes a deliberate annotation.

Full design rationale, the deletion/reclaim matrix, and the Harbor component breakdown are in the
architecture documents that accompanied the original implementation.

---

## Requirements

| | |
|:---|:---|
| Kubernetes | 1.28+ |
| A `StorageClass` | with `allowVolumeExpansion: true`, or plan upgrades cannot grow storage |
| An `IngressClass` | Harbor is exposed through it |
| cert-manager | a `ClusterIssuer` for per-namespace ingress TLS. **The operator verifies Harbor's certificate**, so the issuer's CA must be trusted by the manager pod |
| Egress | the Harbor chart is pulled from `helm.goharbor.io`, images from their upstream registries |

## Configuration

| Variable | Default | Meaning |
|:---|:---|:---|
| `BASE_DOMAIN` | *(required)* | Registry URLs are `registry.<namespace>.<BASE_DOMAIN>` |
| `STORAGE_CLASS` | `longhorn` | For Harbor's PVCs |
| `INGRESS_CLASS` | `nginx` | For Harbor's ingress |
| `CERT_ISSUER` | `letsencrypt-prod` | cert-manager ClusterIssuer |
| `HARBOR_HELM_REPO` | `https://helm.goharbor.io` | Chart repository |
| `HARBOR_CHART_VERSION` | `1.19.2` | Pinned chart version |
| `METRICS_CERT_DIR` | *(empty)* | Serving certificate for the metrics endpoint |

> With `nip.io`, `BASE_DOMAIN` **must** use the dash-separated form (`192-168-10-6.nip.io`). A
> namespace ending in a digit merges into the dotted form and resolves to a different address.

## Deploy

```bash
make docker-build docker-push IMG=<your-registry>/harbor-operator:latest
make install                      # CRDs
make deploy IMG=<your-registry>/harbor-operator:latest
kubectl -n registry-system rollout status deploy/regi-controller-manager
```

## Use

```bash
kubectl create namespace my-team
kubectl -n my-team apply -f - <<'YAML'
apiVersion: registry.opencloud.wso2.com/v1alpha1
kind: Registry
metadata: {name: web}
spec: {plan: starter}
YAML

kubectl -n my-team get registry,registrybackend -w      # Ready in ~3 minutes
kubectl -n my-team get secret registry-credentials-web \
  -o go-template='{{range $k,$v := .data}}{{$k}}={{$v|base64decode}}{{"\n"}}{{end}}'
```

## Plans

`RegistryBackend.spec.plan` sizes the whole Harbor deployment. `Registry.spec.plan` sizes one
project's storage quota inside it. **They are independent concepts** and are easy to conflate.

| Plan | Deployment: registry / database | Project quota |
|:---|:---:|:---:|
| starter | 20 Gi / 2 Gi | 5 Gi |
| professional | 50 Gi / 5 Gi | 20 Gi |
| enterprise | 200 Gi / 10 Gi | 100 Gi |

Backend plans are upgrade-only, enforced by a CEL rule — PersistentVolumeClaims cannot shrink.

## Development

```bash
make test           # unit tests, no cluster required
make manifests generate
```

Harbor's API is exercised against an `httptest` server and the reconcilers against
controller-runtime's fake client, so the suite runs in about a second with no cluster.

---

## Known gaps

Stated plainly rather than discovered later:

- **No NetworkPolicy is created.** Namespaces are an RBAC boundary, not a network one. Without one,
  a pod in another namespace can reach this Harbor's database and cache directly, bypassing
  authentication. Harbor's chart exposes no password field for its internal Redis at all.
- **No backup or restore path.** Deleting a `RegistryBackend` destroys its volumes immediately.
- **Robot credentials do not expire** and carry push, pull and delete across the whole project.
- **One Harbor per namespace is expensive.** Roughly 29 GiB of provisioned storage per namespace
  before a single image is pushed.

---

## Status and licence

Preserved as a working implementation, not under active feature development. Issues and pull
requests are welcome.

Originally developed as part of the [WSO2 Open Cloud Datacenter](https://github.com/wso2/open-cloud-datacenter)
initiative.
