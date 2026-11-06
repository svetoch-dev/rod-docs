# Repository routing

The working repository records its own rod-docs **tag** in root `AGENTS.md`. Code versions are separate: resolve Bazel aliases and commits in `MODULE.bazel`, Terraform module tags in `source` addresses, and Helm code from the submodule gitlink. A branch named in `.gitmodules` is not the pinned version. Record the consumer's resolved refs, aliases, and local paths beside its code; do not put product metadata here.

| Repository | Owns | Version and local navigation in consumer |
|---|---|---|
| [rod-docs](https://github.com/svetoch-dev/rod-docs) | Shared rules, repository map, repeatable skills | Root `AGENTS.md` points to an existing rod-docs tag and `docs/ai/`; topic pages are opened on demand. |
| [bazel-lib](https://github.com/svetoch-dev/bazel-lib) | Bazel macros, extensions, generated Terraform files, image helpers | `MODULE.bazel` `bazel_dep`/`git_override` gives module name and commit; inspect `tools/` or `deps/images/` at that ref. |
| [rules_tf](https://github.com/ggramal/rules_tf) | Terraform Bazel rules and toolchain extension | `MODULE.bazel` gives module name, override and toolchain version; inspect `tf/` at that ref. |
| [rules_bin_tools](https://github.com/ggramal/rules_bin_tools) | Downloaded helper executables | `MODULE.bazel` gives module name, override, extension and `use_repo` executable aliases; inspect `tools/` at that ref. |
| [tf-modules](https://github.com/svetoch-dev/tf-modules) | Reusable Terraform module inputs, implementation and outputs | Each `source` address supplies its own module path and tag; read only the selected `modules/<path>/` at that tag. Different module families may use different tags. |
| [helm-charts](https://github.com/svetoch-dev/helm-charts) | Shared Helm charts and Argo CD App-of-Apps templates | `.gitmodules` gives URL/path; gitlink gives commit. Read its `AGENTS.md`, then selected `Chart.yaml`, `values.yaml`, templates and `Chart.lock`. `Chart.yaml` aliases map dependency names to values keys. |

Aliases are local names, not versions: Bazel `use_repo` names an external repository or executable in labels, Terraform module block names label local instances while `source` pins code, and Helm `Chart.yaml` aliases select dependency values keys. Trace each alias from its declaration to its consumers before changing it.

| Task | Next page | Consumer-owned facts to locate |
|---|---|---|
| CI trigger, tests, plan, deploy | [CI](ci.md) | Workflow, target selection, checks and deployment authorization. |
| Bazel target, helper or image | [Bazel](bazel.md) | `MODULE.bazel`, affected `BUILD.bazel`, aliases and lockfiles. |
| Terraform root or module | [Terraform](terraform.md) | Root templates, local modules, `source` tags, input/output declarations. |
| Kubernetes application or chart | [Argo CD and Helm](argocd.md) | Environment values, local charts, submodule gitlink and applicable local instructions. |
