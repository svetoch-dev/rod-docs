# Design

## Design Pillars

1. We use **cloud providers** for thier Infrastructure as a Service (IaaS).
2. We **minimize the use of cloud-specific services**, ideally relying only on core primitives such as networking, object storage, and virtual machines.
3. Each environment (for example, `development`, `preprod`, and `production`) lives in a **separate, isolated project or account**.
4. We use a dedicated environment named `internal` for **internal tooling**, such as monitoring, log aggregation, and CI/CD systems.
5. Every part of the infrastructure is **defined as code**.
6. Our primary tools are:
    1. `terraform` ([more info](../tf/)) + `helm` for Infrastructure as Code
    2. `argocd` ([more info](../argocd/)) for continuous delivery
    3. `bazel` for:
        - dependency management
        - building and testing only things that have changed
        - running scripts reproducibly across platforms and operating systems
        - as a way to abstract from CI tools
7.  We use **Kubernetes** to run all of our workloads.
8.  We follow the [**Operator Pattern**](https://kubernetes.io/docs/concepts/extend-kubernetes/operator/) wherever it makes sense.
9.  We use the **App of Apps** pattern with Argo CD.
10. We use **Python** for all scripts.


## Design principals

1. We do **not** intentionally introduce technical debt.
2. We keep systems and solutions as simple as possible.
3. We do **not hardcode** company-, environment-, or cloud-specific information in code.
    1. The **only allowed locations** are:
        - `argocd/envs.yaml`
        - `terraform.tfvars.json`

