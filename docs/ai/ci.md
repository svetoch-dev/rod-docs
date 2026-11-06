# CI context path

CI triggers and permitted actions belong to the consuming repository. For a GitHub Actions consumer, read the affected `.github/workflows/` file, then follow its scripts and selected Bazel targets. Distinguish PR checks from protected-branch and manually dispatched jobs. A configured target does not prove it ran or succeeded.

Follow [Bazel](bazel.md) for target implementation, [Terraform](terraform.md) for root and module interfaces, or [Argo CD and Helm](argocd.md) for desired Kubernetes applications. Do not infer live infrastructure from CI definitions.
