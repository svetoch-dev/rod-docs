# Initiazlizing argocd

This script is responsible for initiazlizing argocd (NOT YET IMPLEMENTED). After initial setup all k8s resources are managed by commiting their state to `argocd` folder in `master` branch


```
bazel run @svetoch_bazel_lib//scripts/init/argocd 

```

What it does is

1. Installation of argocd CRDs

```
kubectl apply --server-side --validate=false -f argocd/charts/infra/crds/argocd/
```

2. Installation of argocd helm release

Replace `<registry-url>` with the value of `global.env.registry.url`; direct chart installation requires the resolved image repository. Bootstrap uses built-in Redis, so `argocd.externalRedis.host` must be empty.

```
helm repo add argo https://argoproj.github.io/argo-helm
cd argocd/charts/infra/charts/argocd
helm dependency update
cd -
helm upgrade --install  argocd-gcp-int argocd/charts/infra/charts/argocd/ --set  "redis.enabled=false" --values=argocd/environments/gcp-int/argocd/values.yaml --values argocd/charts/infra/charts/globals.yaml --namespace argocd --set "argocd.enabled=true" --set "argocd.redis.enabled=true" --set-string "argocd.externalRedis.host=" --set-string "argocd.global.image.repository=<registry-url>/argocd" --set-string "argocd.global.image.tag=v3.5.3" --set "probes.enabled=false" --set "prometheus-rules.enabled=false" --set global.env.name=internal --set global.env.short_name=int --set global.env.cloud_short_name=gcp-int --set-string "global.env.dns.domain=<int.your-domain>"
```

3. Creating a root argocd application

```
#./root.yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: root
  namespace: argocd
spec:
  destination:
    server: https://kubernetes.default.svc
    namespace: argocd
  project: default
  source:
    repoURL: <repo_url>
    path: argocd/charts/infra/charts/environments
    targetRevision: master
    helm:
      valueFiles:
        - ../globals.yaml
        - ../../../../envs.yaml
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
```

```
kubectl apply -f root.yaml
```
