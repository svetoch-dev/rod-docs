# Decision Log: Managing Cloud Dependencies in Kubernetes

**Date:** 29.06.2026

**Status:** proposed

**Decision:** Build a custom lightweight Kubernetes operator for managing object storage buckets and IAM permissions.

---

## Context

The Rod project needs a Kubernetes-native way to provision and manage cloud resources (object storage buckets, IAM roles/service accounts) that application components like Thanos and Loki depend on. Today, these resources are managed via Terraform, creating a split-brain problem between infrastructure and application lifecycles.

Our target environment includes not only major clouds (AWS, GCP, Azure) but also **exotic providers** such as YandexCloud, TimeWeb, Hoster.by, and VKCloud. These providers often lack mature tooling in the Kubernetes ecosystem.

## Evaluation Criteria

Solutions were evaluated against the following acceptance criteria:

| # | Criterion | Description |
|---|-----------|-------------|
| 1 | **Exotic Provider Support** | Must support cloud providers like YandexCloud, TimeWeb, Hoster.by, VKCloud that often lack mature ecosystem tooling. Cannot depend on providers having existing Crossplane/operator support. |
| 2 | **Provider Maintenance Independence** | Must not be blocked by third-party provider maintenance cycles. We need direct control over adding new resources and attributes without waiting for upstream Terraform provider or upjet updates. |
| 3 | **Use Case Fit** | Must match our actual use case complexity. We only need to create buckets and grant permissions to cloud principals, not manage entire cloud infrastructure. |
| 4 | **Maintainability** | Must be feasible to create and maintain given modern development practices. |
| 5 | **Kubernetes-native** | Must integrate cleanly with Kubernetes CRDs, controllers, and the declarative reconciliation model. |

## Options Evaluated

### 1. Crossplane

[Crossplane](https://www.crossplane.io/) is an open-source Kubernetes add-on that transforms your cluster into a universal control plane.

- **Exotic Provider Support:** ❌ **No.** While Crossplane supports major clouds, exotic providers like YandexCloud, TimeWeb, Hoster.by, and VKCloud either lack providers entirely or have incomplete implementations. For example, the YandexCloud Crossplane provider does not support IAM binding resources.
- **Provider Maintenance Independence:** ❌ **No.** Crossplane providers are typically generated via [upjet](https://github.com/crossplane/upjet) from Terraform providers. Updating a provider to support new resources or attributes requires upstream Terraform provider updates, which are not guaranteed for exotic providers.
- **Use Case Fit:** ❌ **Poor fit.** Crossplane is designed to manage entire cloud infrastructure estates. Our use case is deliberately simple (buckets + IAM permissions), making Crossplane significant overkill.
- **Maintainability:** ⚠️ **Complex.** Crossplane introduces substantial operational complexity (CompositeResourceDefinitions, Compositions, ProviderConfigs, XRs, XRCs) that must be maintained indefinitely.
- **Kubernetes-native:** ✅ Yes.

**Verdict:** Rejected. Crossplane fails on the three most important criteria for our context: exotic provider support, maintenance independence, and use case fit.

### 2. Existing Open-Source Bucket Operators

Several open-source operators exist, but most are either unmaintained or support only specific object storage backends.

#### 2.1. [bucket-operator](https://github.com/jjaferson/bucket-operator)

- **Exotic Provider Support:** ❌ No. Supports only SeaweedFS.
- **Provider Maintenance Independence:** N/A. Does not support cloud providers.
- **Use Case Fit:** ❌ No. Does not solve our problem.
- **Maintainability:** ❌ Unmaintained (last commit ~2 years ago).
- **Kubernetes-native:** ✅ Yes, but irrelevant.

**Verdict:** Rejected.

#### 2.2. [k8s-s3-bucket-operator](https://github.com/DevangRadadiya/k8s-s3-bucket-operator)

- **Exotic Provider Support:** ❌ No. COSI driver for MinIO and AWS only.
- **Provider Maintenance Independence:** ❌ Limited to supported providers.
- **Use Case Fit:** ⚠️ Partial, but only for supported providers.
- **Maintainability:** ⚠️ Codebase quality is questionable.
- **Kubernetes-native:** ✅ Yes.

**Verdict:** Rejected. Limited provider support and questionable quality.

#### 2.3. [s3-operator](https://github.com/InseeFrLab/s3-operator)

- **Exotic Provider Support:** ❌ No. Supports only MinIO.
- **Provider Maintenance Independence:** ❌ Limited to MinIO.
- **Use Case Fit:** ⚠️ Partial for MinIO, none for exotic providers.
- **Maintainability:** ✅ Well-structured, but too limited.
- **Kubernetes-native:** ✅ Yes.

**Verdict:** Rejected. Well-made but limited to MinIO; does not meet exotic provider requirement.

### 3. Self-Made Lightweight Kubernetes Operator

Building a purpose-built Kubernetes operator that handles only our specific use case: creating object storage buckets and managing IAM permissions for cloud principals.

- **Exotic Provider Support:** ✅ **Yes.** We implement provider interfaces directly using cloud provider SDKs/APIs. No dependency on upstream Crossplane/Terraform providers. If a provider lacks an SDK, we use raw REST API calls.
- **Provider Maintenance Independence:** ✅ **Yes.** We control the entire codebase. Adding a new resource type or attribute requires only our own development cycle, not waiting for Terraform provider updates or upjet regeneration.
- **Use Case Fit:** ✅ **Perfect.** The operator does exactly two things: create buckets and grant permissions. No unnecessary abstractions or features.
- **Maintainability:** ✅ **Feasible.** Modern AI-assisted development ("vibecoding") dramatically lowers the barrier to creating and maintaining a clean, well-tested operator. A focused codebase with a single responsibility is easier to maintain than learning and operating Crossplane's complex machinery.
- **Kubernetes-native:** ✅ Yes. Built with standard Kubernetes controller patterns (CRDs, reconciliation loops, finalizers).

**Verdict:** **Selected.**

### 4. Self-Made COSI Drivers

Building custom drivers for the [Container Object Storage Interface (COSI)](https://github.com/kubernetes-sigs/container-object-storage-interface).

- **Exotic Provider Support:** ⚠️ Potentially, but COSI itself is still evolving and driver adoption is low.
- **Provider Maintenance Independence:** ⚠️ Would be maintained by us, but COSI spec constraints may limit flexibility.
- **Use Case Fit:** ❌ COSI is focused on provisioning storage for workloads, not managing cloud IAM and exotic provider-specific features.
- **Maintainability:** ❌ COSI is not yet stable; building drivers adds complexity without clear benefit.
- **Kubernetes-native:** ✅ Yes, but the standard is immature.

**Verdict:** Rejected. COSI is promising but too immature and narrowly scoped for our needs.

## Comparison Summary

| Option | Exotic Providers | Maintenance Independence | Use Case Fit | Maintainable | K8s-native | Verdict |
|--------|:--------------:|:----------------------:|:----------:|:----------:|:----------:|:-------:|
| **Self-made operator** | ✅ | ✅ | ✅ | ✅ | ✅ | **Selected** |
| Crossplane | ❌ | ❌ | ❌ | ⚠️ | ✅ | Rejected |
| bucket-operator | ❌ | N/A | ❌ | ❌ | ✅ | Rejected |
| k8s-s3-bucket-operator | ❌ | ❌ | ⚠️ | ⚠️ | ✅ | Rejected |
| s3-operator | ❌ | ❌ | ⚠️ | ✅ | ✅ | Rejected |
| Self-made COSI drivers | ⚠️ | ⚠️ | ❌ | ❌ | ✅ | Rejected |

## Decision

**Build a custom lightweight Kubernetes operator** for managing object storage buckets and IAM permissions across all supported cloud providers, including exotic ones.

## Rationale

1. **Exotic Provider Support Is Non-Negotiable:** Rod must run on YandexCloud, TimeWeb, Hoster.by, VKCloud, and similar providers. None of the existing ecosystem tools (Crossplane, existing operators) adequately support these. A custom operator allows us to implement direct API/SDK integrations for any provider.

2. **Independence from Upstream Provider Maintenance:** Crossplane providers depend on Terraform providers which depend on upjet generation. For exotic providers, this chain is brittle and slow. A custom operator breaks this dependency — we update when we need to, not when upstream decides to.

3. **Right-Sized Complexity:** Crossplane is a platform for managing entire cloud estates. We need to create buckets and grant permissions. Building a focused operator with ~2 CRDs and ~2 reconciliation loops is vastly simpler than operating Crossplane's XRD/Composition machinery.

4. **Vibecoding Changes the Calculus:** Modern AI-assisted development makes building and maintaining a clean, well-structured operator feasible without a large dedicated team. The codebase can remain small and focused, making ongoing maintenance manageable.

5. **No Ecosystem Lock-In:** By owning the provider implementations, we are not locked into Crossplane's provider ecosystem or COSI's evolving standard. We can adapt to any provider's API changes immediately.

## Trade-offs Accepted

- **Development Effort:** We must build and maintain the operator ourselves rather than consuming an existing project.
- **No Free Ecosystem:** We do not benefit from Crossplane's existing provider ecosystem or community contributions.
- **Testing Burden:** We are responsible for testing against each cloud provider's API, including handling edge cases and provider-specific quirks.
- **Operator Lifecycle:** We must manage the operator's own deployment, upgrades, and RBAC within Kubernetes.

## Mitigations

- **Keep It Small:** The operator must have a strictly limited scope — buckets and IAM permissions only. Resist feature creep.
- **Provider Interface:** Design a clean provider interface so adding a new cloud requires implementing only the interface methods.
- **Use AI Assistants:** Leverage vibecoding for boilerplate, tests, and provider SDK integration to keep development velocity high.
- **Start with One Provider:** Implement the first provider (e.g., YandexCloud) to validate the architecture before adding others.

## Next Steps

1. Define the operator's CRDs (`Bucket`, `BucketPolicy` or similar).
2. Design the provider interface abstraction.
3. Scaffold the operator using kubebuilder or operator-sdk.
4. Implement the first provider (YandexCloud) to validate the approach.
5. Create Helm chart for operator deployment.
6. Draft migration plan for moving existing Terraform-managed buckets to the operator.
7. Document provider addition guide for future exotic providers.
