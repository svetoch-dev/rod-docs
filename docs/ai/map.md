# Rod template map

```text
rod-docs  [you are here] — shared rules, map, registry, documentation
rod                     — public reference template
└─ consuming repository — product code, environment configuration, dependency refs

rod / consuming repository
├─ GitHub Actions       — CI entry points in `.github/workflows/`
├─ bazel-lib            — shared build targets and macros
├─ rules_tf             — external Bazel rules for Terraform
├─ rules_bin_tools      — external Bazel rules for executable tools
├─ tf-modules           — reusable Terraform infrastructure modules
└─ helm-charts          — shared Kubernetes charts and Argo CD App-of-Apps templates
```

Use the [repository registry](repositories.md) to open the owner repository and its instructions at the relevant ref. For product configuration, CI workflows, or dependency selection, start in the consuming repository. For shared code, use the module repository's own instructions and documentation.

Each module repository's local map should use the same node names, move `[you are here]` to that repository, and point to its task entry paths. The local `AGENTS.md` links to the shared context at a specific `rod-docs` tag.
