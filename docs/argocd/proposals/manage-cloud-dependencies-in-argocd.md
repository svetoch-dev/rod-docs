# Manage Cloud Dependencies in Argo CD

## Overview

Some Kubernetes applications depend on external cloud resources (e.g., object storage buckets, IAM/service accounts). Examples include Thanos and Loki, which require object storage for persistence.

Today, these dependencies are provisioned and managed via Terraform, while application deployment is handled by Argo CD. This split introduces operational complexity and weakens the declarative model we aim for in Kubernetes.

This document proposes managing cloud dependencies directly from Kubernetes using Crossplane.

---

## Current State

- Applications are deployed via Argo CD.
- Cloud resources (e.g., buckets, IAM accounts) are provisioned separately using Terraform.
- Applications consume these resources via manually wired configuration (e.g., secrets, environment variables).

---

## Problems

Managing a single application across Argo CD and Terraform introduces several issues:

1. **Configuration Bridging**
   - Outputs from Terraform (e.g., bucket names, credentials) must be passed into Kubernetes.
   - This often requires custom glue code, secret syncing, or manual steps.

2. **Lifecycle Fragmentation**
   - Application lifecycle spans multiple systems:
     - Argo CD → deploy app
     - Terraform → provision/update/delete cloud resources
   - Keeping them in sync is error-prone.

3. **Multi-Cloud Complexity**
   - Supporting multiple cloud providers requires duplicating logic in Terraform and adapting application configs.
   - There is no unified abstraction layer.

---

## Proposed Solution

Adopt Crossplane to manage cloud resources declaratively within Kubernetes.

### Key Idea

Define cloud resources (e.g., buckets, IAM roles) as Kubernetes custom resources. These are reconciled by Crossplane providers, which ensure the actual cloud infrastructure matches the desired state.

Applications then consume these resources natively via Kubernetes mechanisms (Secrets, ConfigMaps, Helm values).

---

## Desired State

- All application-related resources (Kubernetes + cloud) are defined in a single declarative system (Kubernetes manifests managed by Argo CD).
- Crossplane controllers provision and manage cloud resources.
- Applications consume credentials and endpoints directly from Kubernetes.

---

## Benefits

1. **Single Source of Truth**
   - Application and infrastructure dependencies are defined together in Kubernetes manifests.

2. **Declarative Reconciliation**
   - Crossplane controllers continuously ensure cloud resources match the desired state.

3. **Improved Security Isolation**
   - Each application can have its own dedicated cloud resources and credentials.
   - Reduces shared resource risks.

4. **Simplified Multi-Cloud Support**
   - Cloud-specific differences are encapsulated in resource definitions.
   - Enables a more consistent interface across providers.

---

## Trade-offs

1. **Migration Effort**
   - Existing Terraform-managed resources must be migrated or imported.

2. **Abstraction Limitations**
   - Not all cloud features may be equally supported across providers.

---

## Example

A simplified Helm template for provisioning a bucket across different cloud providers:

```yaml
{{- if eq .Values.cloudProvider "gcp" }}
apiVersion: storage.gcp.upbound.io/v1beta1
kind: Bucket
metadata:
  name: {{ .Values.bucket.name }}
spec:
  forProvider:
    location: {{ .Values.bucket.location }}

{{- else if eq .Values.cloudProvider "aws" }}
apiVersion: s3.aws.upbound.io/v1beta1
kind: Bucket
metadata:
  name: {{ .Values.bucket.name }}
spec:
  forProvider:
    region: {{ .Values.bucket.region }}
{{- end }}
