# Repository routing

| Task concerns | Read in shared repositories | Read in the consuming repository |
|---|---|---|
| Bazel and Terraform targets | `bazel-lib/tools/macros/tf.bzl`, `tools/extensions.bzl`; `ggramal/rules_tf/tf/defs.bzl` and `tf/extensions.bzl` | `MODULE.bazel` for pinned refs and toolchain; the affected root's `BUILD.bazel` |
| Terraform resources and module interfaces | Relevant `*.tf` files under `tf-modules/modules/<module>/` at the ref in the source address: inputs, implementation, and outputs if present | The affected `main.tf.tpl` or `.tf` source, local variables, local modules, and outputs |
| Argo CD application hierarchy and Helm templates | `helm-charts/charts/environments/`, `charts/environment/`, and the selected chart under `charts/` at the gitlink commit | `.gitmodules`, the gitlink, environment definition, values, and any local chart |

`rod-docs` owns this navigation and background documentation. `bazel-lib`, `tf-modules`, `helm-charts`, and `rules_tf` own their code interfaces. Product-specific configuration and dependency selections stay in the consuming repository. Follow the [Terraform](terraform.md) or [GitOps](gitops.md) path only when relevant.
