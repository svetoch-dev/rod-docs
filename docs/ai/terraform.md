# Terraform context path

The consumer owns root configuration and environment selection. `bazel-lib` may generate root files; `rules_tf` supplies Terraform rules and a toolchain; `tf-modules` owns reusable module interfaces. Resolve each version from the consumer before using its documentation.

| Source edge | Interface to inspect |
|---|---|
| Consumer `MODULE.bazel` → `bazel-lib`/`rules_tf` | `git_override` commit, `load_file.json` source, `override_repo`, Terraform toolchain. If used, `bazel-lib/tools/extensions.bzl` loads JSON into `json.bzl`; `tools/utils/format.bzl` and `tools/macros/tf.bzl` select root data and render root files/targets. Inspect declarations without reading values. |
| Consumer root `BUILD.bazel` → generated root | `tf()` arguments, template `main.tf.tpl`, variable schema, generated file names and target names at the pinned macro version. |
| Root `main.tf.tpl`/`.tf` → local or shared module | `source` path and tag, input arguments, module `input.tf`/`variables.tf`, output declarations, then only implementation relevant to the task. |
| Root → another root | `terraform_remote_state` declaration and referenced output names establish a source dependency. Read declarations only; outputs may contain secrets. |

Do not read raw state, plan, command output, generated `tfvars` values or sensitive outputs for context. Source declarations do not establish applied resources or current values. Terraform execution needs operation-specific authorization under [shared rules](AGENTS.md).
