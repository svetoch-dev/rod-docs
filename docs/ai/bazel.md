# Bazel context path

Bazel connects the consumer's build targets to shared macros, Terraform rules, helper executables, Python dependencies, and container images. It is an execution and packaging layer; the consumer's CI decides when its targets are selected.

1. Read `MODULE.bazel` for the module name, version or override commit, extension, `use_repo` alias, and toolchain. A declared dependency alone does not show which target uses it.
2. Read the affected `BUILD.bazel` for loaded symbols and labels. Follow only those symbols to the pinned `bazel-lib`, `rules_tf`, `rules_bin_tools`, Python, or OCI source and any relevant lockfile.
3. For Terraform generation, follow [Terraform](terraform.md). For CI target selection, follow [CI](ci.md). Check generated-file boundaries before editing source or running a target.
