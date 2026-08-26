# helm-charts

Helm charts deployed to the saoad.dk clusters, synced by ArgoCD.

## Layout

| Chart | Kind | Notes |
| --- | --- | --- |
| `charts/argocd` | wrapper | Pins upstream `argo/argo-cd`; see its README to install |
| `charts/grafana` | wrapper | Pins upstream `grafana-community/grafana`; overrides only |
| `charts/prometheus` | wrapper | Pins upstream `prometheus-community/prometheus`; overrides only |
| `charts/matomo` | first-party | Matomo + in-cluster MariaDB + archiving CronJob |
| `charts/rabbitmq` | first-party | `RabbitmqCluster` CR; needs the RabbitMQ Cluster Operator |

Each chart carries a single `values.yaml` targeting the stage cluster. The
previous `values-stage.yaml` / `values-prod.yaml` split (and rabbitmq's
`stage/` and `prod/` directories) was collapsed; recover the production values
from git history if needed.

## Wrapper charts

`grafana` and `prometheus` contain no templates. They declare the upstream chart
as a dependency and carry only the values that differ from upstream defaults, so
upgrades are a one-line version bump instead of a merge against a vendored fork.

`charts/grafana/dashboards/` is deliberate and is **not** consumed by Helm — it
is the location Grafana's Git Sync provisioning reads dashboard JSON from
(`feature_toggles.provisioning` / `kubernetesDashboards` in the values file).
Keep `dashboards: {}` there; the chart is not meant to render these.

To upgrade one, edit the pinned `version` in its `Chart.yaml`, then:

```sh
helm dependency update charts/<name>
git add charts/<name>/Chart.yaml charts/<name>/Chart.lock
```

`Chart.lock` is committed; the `charts/*.tgz` tarballs are not — Helm and ArgoCD
both rebuild them from the lock file.

To see every value available for overriding:

```sh
helm show values argo/argo-cd --version 10.4.0
helm show values grafana-community/grafana --version 9.4.5
helm show values prometheus-community/prometheus --version 28.9.0
```

Argo CD is installed with Helm rather than managed by itself — it cannot sync
itself into existence on a fresh cluster. See `charts/argocd/README.md`.

## Rendering locally

```sh
helm template grafana  charts/grafana
helm template matomo   charts/matomo
helm template rabbitmq charts/rabbitmq
```

`charts/rabbitmq` renders a `RabbitmqCluster` custom resource and needs the
RabbitMQ Cluster Operator installed in the target cluster.

## Credentials

All credentials in this repo are static placeholders intended for a local test
cluster — not secrets. No external secret manager is wired up.

For anything real, override at deploy time (`--set`) or point the chart at an
externally provisioned Secret; the upstream Grafana chart supports this via
`admin.existingSecret`.
