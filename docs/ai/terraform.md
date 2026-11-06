# Terraform context path

1. Read the consumer's `MODULE.bazel` to identify its `bazel-lib` and `rules_tf` refs, Terraform toolchain, and any `load_file.json` / `override_repo` configuration.
2. If that JSON setup is present, trace the source file through `bazel-lib/tools/extensions.bzl`, `tools/repo_rules/load_json_file.bzl`, and `tools/utils/format.bzl` to `tools/macros/tf.bzl`. The macro renders Terraform root files and declares Bazel targets; use its version at the consumer's pinned ref for exact target names.
3. Read the affected root's `BUILD.bazel`, `main.tf.tpl` or `.tf`, local variables, and outputs. Resolve each Terraform module's `source` ref from those files. Read only the affected module's inputs, implementation, and outputs at that ref.
4. For data crossing Terraform roots, inspect the root outputs and any `terraform_remote_state` configuration. Source files do not reveal current state or the results of an apply.

The `tf-modules` repository contains reusable modules; the consumer owns environment configuration and any local modules. The consumer's CI and runbooks define execution paths.
