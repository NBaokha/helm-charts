# Argo CD

Thin wrapper around the upstream [argo-helm](https://github.com/argoproj/argo-helm)
`argo-cd` chart. No templates are vendored; `values.yaml` holds only the
overrides that differ from upstream defaults.

Tuned for a single-node local cluster: SSO (dex) and the notifications
controller are disabled, and no Ingress is created. The API server keeps TLS
enabled, so it serves HTTPS on 8080 and redirects plain HTTP there — both
schemes work, at the cost of a self-signed certificate warning.

## Install

```sh
helm dependency build charts/argocd
helm install argocd charts/argocd -n argocd --create-namespace
```

Wait for it to come up:

```sh
kubectl -n argocd rollout status deploy/argocd-server
kubectl -n argocd get pods
```

## Log in

Get the auto-generated admin password:

```sh
kubectl -n argocd get secret argocd-initial-admin-secret \
  -o jsonpath='{.data.password}' | base64 -d; echo
```

Port-forward the UI and open <https://localhost:8080> as user `admin`:

```sh
kubectl -n argocd port-forward svc/argocd-server 8080:443
```

The certificate is self-signed, so the browser will warn once — accept it. Plain
`http://localhost:8080` redirects to HTTPS rather than failing.

The initial admin secret is generated once at install. Rotate the password from
the UI, then delete the Secret — Argo CD does not need it afterwards.

## Registering the other charts

Point an Application at a chart in this repo, for example:

```sh
kubectl -n argocd apply -f - <<'EOF'
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: prometheus
  namespace: argocd
  # Without this finalizer, deleting the Application leaves its workloads behind.
  finalizers:
    - resources-finalizer.argocd.argoproj.io
spec:
  project: default
  source:
    repoURL: https://github.com/NBaokha/helm-charts.git
    targetRevision: HEAD
    path: charts/prometheus
  destination:
    server: https://kubernetes.default.svc
    namespace: default
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
EOF
```

Installing into the `argocd` namespace also makes the Argo CD scrape targets in
`charts/prometheus/values.yaml` resolve.

## Upgrading

Bump the pinned `version` in `Chart.yaml`, then:

```sh
helm dependency update charts/argocd
helm upgrade argocd charts/argocd -n argocd
```

## Uninstall

```sh
helm uninstall argocd -n argocd
kubectl delete namespace argocd
# CRDs are not removed by helm uninstall:
kubectl delete crd applications.argoproj.io applicationsets.argoproj.io appprojects.argoproj.io
```

## Do not let it manage itself yet

Argo CD cannot sync itself into existence on a fresh cluster, and a bad
self-sync can take out the controller that would fix it. Install it with Helm as
above, let it manage the other charts, and only revisit self-management once
things are stable.
