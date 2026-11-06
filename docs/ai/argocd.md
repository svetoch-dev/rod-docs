# Argo CD and Helm context path

Argo CD App-of-Apps describes Kubernetes Applications; Helm charts supply their templates and values. CI triggers and target selection are documented separately in [CI](ci.md).

1. Resolve a `helm-charts` submodule by its gitlink commit; `.gitmodules` supplies URL and path, not the code version. Read the pinned submodule's `AGENTS.md` before editing it.
2. In the selected version, inspect `charts/environments/` for root and environment Applications and `charts/environment/` for generated `envApps`, explicit `chart_apps`, and conditional CRD/manifests Applications. Follow `repository.paths` and each generated `targetRevision`; parent and child revisions need not match.
3. In the consumer, inspect the root environment definition, affected `env.yaml`, application `values.yaml`, and any local chart. For a shared chart change, check its `Chart.yaml` dependencies and aliases, `values.yaml`, templates, and `Chart.lock` at the pinned version.

Templates and values declare desired configuration; they do not prove cluster sync or deployed state.
