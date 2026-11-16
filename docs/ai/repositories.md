# Rod repository registry

Versions below are the refs selected by the public [Rod template at `ff0c5d2`](https://github.com/svetoch-dev/rod/tree/ff0c5d2636291268a5e5d8ecf6dc8f591f32a351), checked on 2026-11-12. For work in a consuming repository, use its own dependency refs and its selected `rod-docs` tag.

| Repository | Template ref | Agent instructions at that ref |
|---|---|---|
| [rod](https://github.com/svetoch-dev/rod) | [`ff0c5d2`](https://github.com/svetoch-dev/rod/tree/ff0c5d2636291268a5e5d8ecf6dc8f591f32a351), reference snapshot | No root `AGENTS.md` |
| [rod-docs](https://github.com/svetoch-dev/rod-docs) | Shared context tag pending | [Shared `docs/ai/AGENTS.md`](AGENTS.md). No root `AGENTS.md` in this draft. |
| [bazel-lib](https://github.com/svetoch-dev/bazel-lib) | [`v0.30.0`](https://github.com/svetoch-dev/bazel-lib/tree/v0.30.0) | [Root `AGENTS.md`](https://github.com/svetoch-dev/bazel-lib/blob/v0.30.0/AGENTS.md) |
| [tf-modules](https://github.com/svetoch-dev/tf-modules) | [`v0.26.0`](https://github.com/svetoch-dev/tf-modules/tree/v0.26.0) | No root `AGENTS.md` |
| [helm-charts](https://github.com/svetoch-dev/helm-charts) | [`v11.8.2`](https://github.com/svetoch-dev/helm-charts/tree/v11.8.2), matches the submodule gitlink | [Root `AGENTS.md`](https://github.com/svetoch-dev/helm-charts/blob/v11.8.2/AGENTS.md) |
| [rules_tf](https://github.com/ggramal/rules_tf) | [`841d00e`](https://github.com/ggramal/rules_tf/tree/841d00ec6a9d619e278c6adf37b343bda7f1a554), commit pin | No root `AGENTS.md` |
| [rules_bin_tools](https://github.com/ggramal/rules_bin_tools) | [`d3fefc5`](https://github.com/ggramal/rules_bin_tools/tree/d3fefc59031fe28fcc41918f8e9df752c58c3c49), commit pin | No root `AGENTS.md` |

Update this registry when the reference template changes its pins. Sources: [`MODULE.bazel`](https://github.com/svetoch-dev/rod/blob/ff0c5d2636291268a5e5d8ecf6dc8f591f32a351/MODULE.bazel), [Terraform module source](https://github.com/svetoch-dev/rod/blob/ff0c5d2636291268a5e5d8ecf6dc8f591f32a351/terraform/environments/internal/k8s/main.tf.tpl), [Helm submodule gitlink](https://github.com/svetoch-dev/rod/tree/ff0c5d2636291268a5e5d8ecf6dc8f591f32a351/argocd/charts/infra).
