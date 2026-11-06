# GitOps context path

1. If `helm-charts` is a Git submodule, use the consumer's gitlink commit to read its code; `.gitmodules` identifies the source and branch but does not pin the checkout.
2. At the inspected ref, check `charts/environments/` for root and environment Argo CD Application templates. Check `charts/environment/` for applications generated from `envApps` and explicit `chart_apps`; CRD and manifests Applications depend on their `enabled` values.
3. Read the consumer's root environment definition, the affected environment's `env.yaml`, its application `values.yaml`, and any local chart. When changing a shared chart's interface or dependencies, inspect its `Chart.yaml`, `values.yaml`, templates, and `Chart.lock` if present.

Application templates and values describe desired configuration. They do not prove what is currently synchronized in a cluster.
