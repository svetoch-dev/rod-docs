# Design details

## 1. Overview

This document describes the architecture for a Kubernetes controller/operator that manages object storage buckets, cloud IAM principals, and permissions between them across multiple cloud providers.

The system is based on Kubernetes Custom Resources and follows a reconciliation model:

```text
Custom Resources
    ↓
Kubernetes Controllers
    ↓
Cloud Provider APIs
    ↓
Buckets / IAM Principals / IAM Bindings
    ↓
CR Status
```

The main goals are:

- Provide a Kubernetes-native API for managing cloud storage access.
- Support multiple cloud providers.
- Keep user-facing APIs mostly cloud-neutral.
- Allow provider-specific configuration where needed.
- Avoid unsafe deletion behavior.
- Make reconciliation idempotent and drift-aware.
- Clearly report unsupported features and reconciliation status.

## 2. Goals

### 2.1 Functional Goals

The controller should manage:

- Object storage buckets.
- Cloud IAM principals.
- Access grants between principals and buckets.
- Bucket attributes such as:
  - versioning
  - lifecycle rules
  - labels/tags
  - storage class
  - provider-specific settings

### 2.2 Multi-Cloud Goals

The design should support multiple providers, for example:

- GCP
- AWS
- Azure
- Yandex Cloud

The same high-level Kubernetes API should work across providers where possible.

### 2.3 Safety Goals

The controller should:

- Use finalizers for external resource cleanup.
- Default bucket deletion to `Retain`.
- Avoid silently ignoring unsupported features.
- Avoid exposing raw provider IAM details by default.
- Clearly distinguish between owned and referenced resources.
- Store external IDs in status.

## 3. Non-Goals

This design does not aim to expose every provider-specific storage or IAM feature in the common API.

It also does not aim to make all clouds behave identically. Provider-specific differences are expected and should be handled explicitly.

## 4. High-Level Architecture

```text
ProviderConfig
    ↓
Bucket Controller
    manages external buckets

CloudPrincipal Controller
    manages external cloud identities

BucketAccess Controller
    manages permissions between buckets and principals
```

Relationship between resources:

```text
Bucket CR ───────────────┐
                         ↓
                    BucketAccess CR
                         ↓
CloudPrincipal CR ───────┘
                         ↓
                  Cloud IAM binding / policy
```

## 5. Custom Resources

The recommended core CRDs are:

```text
ProviderConfig
Bucket
CloudPrincipal
BucketAccess
```

Optionally, higher-level convenience CRDs can be added later, for example:

```text
ApplicationBucket
ApplicationStorageAccess
```

These higher-level CRs can create lower-level `Bucket`, `CloudPrincipal`, and `BucketAccess` resources.

## 6. ProviderConfig

`ProviderConfig` describes the target cloud provider and the credentials/configuration required to manage resources there.

Example:

```yaml
apiVersion: platform.example.com/v1alpha1
kind: ProviderConfig
metadata:
  name: gcp-dev
spec:
  type: GCP
  gcp:
    projectId: my-gcp-project
    location: europe-west1
    credentialsSecretRef:
      name: gcp-creds
      namespace: platform-system
```

AWS example:

```yaml
apiVersion: platform.example.com/v1alpha1
kind: ProviderConfig
metadata:
  name: aws-dev
spec:
  type: AWS
  aws:
    accountId: "123456789012"
    region: eu-central-1
    credentialsSecretRef:
      name: aws-creds
      namespace: platform-system
```

The controller should not require cloud credentials inside every resource. Resources should reference a `ProviderConfig`.

## 7. Bucket Resource

`Bucket` owns the lifecycle of an external object storage bucket.

Example:

```yaml
apiVersion: platform.example.com/v1alpha1
kind: Bucket
metadata:
  name: app-logs
  namespace: my-app
spec:
  providerRef:
    name: gcp-dev

  name: app-logs-dev
  location: europe-west1

  deletionPolicy: Retain

  versioning:
    enabled: true

  lifecycle:
    rules:
      - name: delete-old-logs
        enabled: true
        prefix: logs/
        ageDays: 90
        action: Delete

  labels:
    app: payments
    env: dev

  unsupportedFeaturePolicy: Fail

  providerConfig:
    gcp:
      storageClass: STANDARD
      uniformBucketLevelAccess: true
      publicAccessPrevention: enforced
```

### 7.1 Bucket Responsibilities

The `Bucket` controller should:

- Ensure the external bucket exists.
- Apply supported bucket attributes.
- Validate provider compatibility.
- Detect unsupported features.
- Store external bucket identifiers in status.
- Handle deletion according to `deletionPolicy`.

### 7.2 Bucket Status

Example:

```yaml
status:
  externalName: app-logs-dev
  externalId: projects/_/buckets/app-logs-dev

  observedProvider: GCP

  applied:
    versioning:
      enabled: true
    lifecycle:
      rules:
        - name: delete-old-logs
          applied: true

  conditions:
    - type: Ready
      status: "True"
      reason: Reconciled
      message: Bucket is ready
```

## 8. CloudPrincipal Resource

`CloudPrincipal` represents a cloud IAM identity.

The name `CloudPrincipal` is preferred over `ServiceAccount` because `ServiceAccount` is already overloaded in Kubernetes and because not every cloud uses the same identity model.

Examples:

```text
GCP: service account
AWS: IAM role or IAM user
Azure: managed identity or app registration
Yandex: service account
```

Example:

```yaml
apiVersion: platform.example.com/v1alpha1
kind: CloudPrincipal
metadata:
  name: app-logs-writer
  namespace: my-app
spec:
  providerRef:
    name: gcp-dev

  type: WorkloadIdentity
  name: app-logs-writer

  deletionPolicy: Delete
```

### 8.1 CloudPrincipal Responsibilities

The `CloudPrincipal` controller should:

- Ensure the external cloud identity exists.
- Normalize names according to provider constraints.
- Store external identity information in status.
- Delete or retain the external identity according to `deletionPolicy`.

### 8.2 CloudPrincipal Status

Example for GCP:

```yaml
status:
  externalName: app-logs-writer
  externalId: app-logs-writer@my-project.iam.gserviceaccount.com

  conditions:
    - type: Ready
      status: "True"
      reason: Reconciled
```

Example for AWS:

```yaml
status:
  externalName: app-logs-writer
  externalId: arn:aws:iam::123456789012:role/app-logs-writer

  conditions:
    - type: Ready
      status: "True"
      reason: Reconciled
```

## 9. BucketAccess Resource

`BucketAccess` manages the permission relationship between a bucket and a principal.

It should not create buckets or principals. It should only reference them.

Example:

```yaml
apiVersion: platform.example.com/v1alpha1
kind: BucketAccess
metadata:
  name: app-logs-writer-access
  namespace: my-app
spec:
  bucketRef:
    name: app-logs

  principalRef:
    name: app-logs-writer

  access:
    level: ObjectWriter
```

### 9.1 BucketAccess Responsibilities

The `BucketAccess` controller should:

- Resolve the referenced `Bucket`.
- Resolve the referenced `CloudPrincipal`.
- Wait until both are ready.
- Validate that both use the same provider.
- Grant the requested access.
- Remove only the IAM binding/policy on deletion.
- Never delete the bucket or principal.

### 9.2 Access Levels

Use portable access levels instead of raw provider IAM roles.

Recommended initial access levels:

```text
ObjectReader
ObjectWriter
ObjectAdmin
BucketAdmin
```

Example:

```yaml
spec:
  access:
    level: ObjectWriter
```

Provider implementations map this to provider-specific permissions.

Example mapping:

```text
ObjectWriter:
  GCP:
    custom role or predefined role allowing storage.objects.create

  AWS:
    s3:PutObject on arn:aws:s3:::bucket/*

  Azure:
    Storage Blob Data Contributor or equivalent scoped role

  Yandex:
    provider-specific object write permission
```

Raw provider roles should be avoided in the common API unless an advanced escape hatch is explicitly added.

### 9.3 BucketAccess Status

Example:

```yaml
status:
  bindingId: bucket/app-logs-dev:principal/app-logs-writer:ObjectWriter

  conditions:
    - type: BucketReady
      status: "True"

    - type: PrincipalReady
      status: "True"

    - type: AccessGranted
      status: "True"

    - type: Ready
      status: "True"
      reason: AccessGranted
```

If a dependency is not ready:

```yaml
status:
  conditions:
    - type: Ready
      status: "False"
      reason: BucketNotReady
      message: Referenced Bucket app-logs is not Ready
```

## 10. Bucket Attributes and Multi-Cloud Differences

Not all clouds support the same bucket features. The API should therefore be split into:

```text
1. Common portable fields
2. Platform-level abstractions
3. Provider-specific configuration
```

### 10.1 Common Portable Fields

These fields should exist in the common `Bucket` spec because they are broadly useful across providers:

```yaml
spec:
  name: app-logs-dev
  location: europe-west1
  deletionPolicy: Retain

  labels:
    app: payments
    env: dev

  versioning:
    enabled: true

  lifecycle:
    rules:
      - name: expire-old-logs
        enabled: true
        prefix: logs/
        ageDays: 90
        action: Delete
```

### 10.2 Platform-Level Abstractions

For semi-portable concepts, use platform-defined values rather than provider-native names.

Example:

```yaml
spec:
  storageClass: Archive
```

The provider implementation maps this to cloud-specific values:

```text
GCP: ARCHIVE
AWS: GLACIER or DEEP_ARCHIVE
Azure: Archive tier
Yandex: provider-specific cold/archive equivalent
```

### 10.3 Provider-Specific Configuration

Provider-specific fields should be namespaced under `spec.providerConfig`.

Example:

```yaml
spec:
  providerConfig:
    aws:
      objectOwnership: BucketOwnerEnforced
      publicAccessBlock:
        blockPublicAcls: true
        blockPublicPolicy: true
        ignorePublicAcls: true
        restrictPublicBuckets: true

    gcp:
      uniformBucketLevelAccess: true
      publicAccessPrevention: enforced

    azure:
      allowBlobPublicAccess: false
      minimumTlsVersion: TLS1_2
```

Only the matching provider-specific section should be allowed.

For example, if `providerRef` points to GCP but `providerConfig.aws` is set, the controller should fail validation:

```text
Reason: InvalidProviderConfig
Message: providerConfig.aws is set but providerRef points to provider type GCP
```

## 11. Unsupported Feature Policy

The controller should never silently ignore unsupported fields.

Add:

```yaml
spec:
  unsupportedFeaturePolicy: Fail
```

Recommended values:

```text
Fail
Warn
Ignore
```

Recommended default:

```text
Fail
```

Behavior:

```text
Fail:
  Mark the resource Ready=False and do not apply partial unsupported config.

Warn:
  Apply supported config, report unsupported fields in status.

Ignore:
  Apply supported config and ignore unsupported fields.
```

`Ignore` should be used carefully, if at all.

Example status for unsupported feature:

```yaml
status:
  unsupported:
    - field: spec.lifecycle.rules[0].storageClass
      reason: UnsupportedStorageClass
      message: Archive storage class is not supported by this provider

  conditions:
    - type: Ready
      status: "False"
      reason: UnsupportedFeature
```

## 12. Provider Capabilities

Each provider implementation should expose capabilities.

Example Go type:

```go
type BucketCapabilities struct {
    Versioning               bool
    LifecycleExpiration      bool
    LifecycleTransition      bool
    ObjectLock               bool
    UniformBucketLevelAccess bool
    PublicAccessBlock        bool
    Tags                     bool
}
```

The controller should validate requested features against provider capabilities before applying changes.

Example behavior:

```text
User enables lifecycle transition
Provider does not support lifecycle transition
    ↓
Bucket Ready=False
Reason=UnsupportedFeature
Message=Provider does not support lifecycle transition rules
```

## 13. Provider Interface

Use a common provider interface and provider-specific implementations.

Example:

```go
type BucketProvider interface {
    ValidateBucketSpec(spec BucketSpec) ValidationResult
    EnsureBucket(ctx context.Context, spec BucketSpec) (*BucketState, error)
    DeleteBucket(ctx context.Context, status BucketStatus) error
}
```

For principals:

```go
type PrincipalProvider interface {
    ValidatePrincipalSpec(spec CloudPrincipalSpec) ValidationResult
    EnsurePrincipal(ctx context.Context, spec CloudPrincipalSpec) (*PrincipalState, error)
    DeletePrincipal(ctx context.Context, status CloudPrincipalStatus) error
}
```

For access:

```go
type BucketAccessProvider interface {
    ValidateBucketAccessSpec(spec BucketAccessSpec) ValidationResult
    EnsureBucketAccess(
        ctx context.Context,
        bucket BucketState,
        principal PrincipalState,
        access AccessSpec,
    ) (*AccessState, error)
    DeleteBucketAccess(ctx context.Context, status BucketAccessStatus) error
}
```

Provider implementations:

```text
internal/cloud/gcp
internal/cloud/aws
internal/cloud/azure
internal/cloud/yandex
```

Provider registry:

```go
func NewProvider(cfg ProviderConfig) (CloudProvider, error) {
    switch cfg.Spec.Type {
    case "GCP":
        return gcp.New(cfg)
    case "AWS":
        return aws.New(cfg)
    case "Azure":
        return azure.New(cfg)
    case "Yandex":
        return yandex.New(cfg)
    default:
        return nil, fmt.Errorf("unsupported provider type %q", cfg.Spec.Type)
    }
}
```

## 14. Reconciliation Model

All operations must be idempotent.

The controller should use `Ensure*` methods, not blind `Create*` methods.

Good:

```text
EnsureBucket
EnsurePrincipal
EnsureBucketAccess
```

Bad:

```text
CreateBucket
CreatePrincipal
CreateBucketAccess
```

Controllers must tolerate:

- retries
- restarts
- duplicate reconciliations
- already-existing resources
- eventual consistency
- manual drift

## 15. Bucket Controller Flow

```text
1. Fetch Bucket CR
2. Fetch ProviderConfig
3. Build provider client
4. Validate Bucket spec against provider capabilities
5. Add finalizer if missing
6. If deleting:
     - apply deletionPolicy
     - remove finalizer
7. Ensure external bucket exists
8. Apply bucket attributes
9. Update status
10. Set Ready=True
```

Pseudo-code:

```go
func (r *BucketReconciler) Reconcile(ctx context.Context, req ctrl.Request) (ctrl.Result, error) {
    bucket := getBucket(req)

    providerConfig := getProviderConfig(bucket.Spec.ProviderRef)
    provider := providerFactory(providerConfig)

    validation := provider.ValidateBucketSpec(bucket.Spec)
    if !validation.Valid {
        setCondition(bucket, "Ready", "False", "UnsupportedFeature", validation.Message)
        return ctrl.Result{}, nil
    }

    if bucket.MarkedForDeletion {
        cleanupBucket(ctx, provider, bucket)
        removeFinalizer(bucket)
        return ctrl.Result{}, nil
    }

    ensureFinalizer(bucket)

    state, err := provider.EnsureBucket(ctx, bucket.Spec)
    if err != nil {
        setCondition(bucket, "Ready", "False", "ReconcileError", err.Error())
        return ctrl.Result{}, err
    }

    updateBucketStatus(bucket, state)
    setCondition(bucket, "Ready", "True", "Reconciled", "Bucket is ready")

    return ctrl.Result{}, nil
}
```

## 16. CloudPrincipal Controller Flow

```text
1. Fetch CloudPrincipal CR
2. Fetch ProviderConfig
3. Build provider client
4. Validate principal spec
5. Add finalizer if missing
6. If deleting:
     - apply deletionPolicy
     - remove finalizer
7. Ensure external principal exists
8. Update status
9. Set Ready=True
```

## 17. BucketAccess Controller Flow

```text
1. Fetch BucketAccess CR
2. Resolve referenced Bucket
3. Resolve referenced CloudPrincipal
4. Check both resources are Ready
5. Check both resources use the same provider
6. Build provider client from referenced provider
7. Validate access level support
8. Add finalizer if missing
9. If deleting:
     - remove IAM binding/policy only
     - remove finalizer
10. Ensure access grant exists
11. Update status
12. Set Ready=True
```

## 18. Watches

The `BucketAccess` controller should reconcile when any of these change:

```text
BucketAccess
Bucket
CloudPrincipal
```

When a `Bucket` changes, enqueue all `BucketAccess` resources that reference it.

When a `CloudPrincipal` changes, enqueue all `BucketAccess` resources that reference it.

## 19. Finalizers and Deletion Policies

External cloud resources must be cleaned up explicitly.

Recommended finalizers:

```text
platform.example.com/bucket-finalizer
platform.example.com/cloudprincipal-finalizer
platform.example.com/bucketaccess-finalizer
```

### 19.1 Bucket Deletion Policy

Recommended values:

```text
Retain
DeleteIfEmpty
DeleteWithContents
```

Recommended default:

```text
Retain
```

`DeleteWithContents` should be avoided unless absolutely required and protected by additional safeguards.

### 19.2 CloudPrincipal Deletion Policy

Recommended values:

```text
Retain
Delete
```

### 19.3 BucketAccess Deletion

Deleting `BucketAccess` should only remove the access grant.

It should not delete:

- the bucket
- the principal

## 20. Ownership Model

Use references between reusable resources.

Do not make `BucketAccess` own `Bucket` or `CloudPrincipal`, because the relationship is many-to-many.

```text
One bucket can have many access grants.
One principal can access many buckets.
```

Use `ownerReferences` only when a higher-level CR creates lower-level CRs.

Example:

```text
ApplicationStorage
    owns Bucket
    owns CloudPrincipal
    owns BucketAccess
```

## 21. Drift Handling

The controller should detect and correct drift by default.

Example drift:

```text
Someone manually removes an IAM binding.
```

Expected behavior:

```text
BucketAccess controller restores the binding on next reconcile.
```

Optional future field:

```yaml
spec:
  driftPolicy: Correct
```

Possible values:

```text
Correct
DetectOnly
```

Recommended default:

```text
Correct
```

## 22. Naming

Do not blindly use Kubernetes `metadata.name` as the external cloud name.

Clouds have different naming constraints.

The controller should use a naming module:

```text
internal/naming/
  bucket.go
  principal.go
```

Recommended behavior:

```yaml
spec:
  name: app-logs-dev
```

or:

```yaml
spec:
  generateName: app-logs
```

The controller should validate or generate provider-safe names.

## 23. Recommended Code Layout

```text
cmd/
  manager/
    main.go

api/
  v1alpha1/
    providerconfig_types.go
    bucket_types.go
    cloudprincipal_types.go
    bucketaccess_types.go

controllers/
  bucket_controller.go
  cloudprincipal_controller.go
  bucketaccess_controller.go

internal/
  cloud/
    provider.go
    registry.go

    gcp/
      provider.go
      buckets.go
      principals.go
      access.go

    aws/
      provider.go
      buckets.go
      principals.go
      access.go

    azure/
      provider.go
      buckets.go
      principals.go
      access.go

    yandex/
      provider.go
      buckets.go
      principals.go
      access.go

  naming/
    bucket.go
    principal.go

  conditions/
    conditions.go

  validation/
    bucket.go
    principal.go
    access.go
```

## 24. Kubernetes RBAC

The controller needs permissions to:

- get/list/watch custom resources
- update status
- update finalizers
- read provider credential secrets
- create/patch events

Example:

```yaml
rules:
  - apiGroups: ["platform.example.com"]
    resources:
      - providerconfigs
      - buckets
      - cloudprincipals
      - bucketaccesses
    verbs: ["get", "list", "watch"]

  - apiGroups: ["platform.example.com"]
    resources:
      - buckets
      - cloudprincipals
      - bucketaccesses
    verbs: ["update", "patch"]

  - apiGroups: ["platform.example.com"]
    resources:
      - buckets/status
      - cloudprincipals/status
      - bucketaccesses/status
    verbs: ["update", "patch"]

  - apiGroups: [""]
    resources: ["secrets"]
    verbs: ["get"]

  - apiGroups: [""]
    resources: ["events"]
    verbs: ["create", "patch"]
```

## 25. Security Recommendations

The controller should follow these rules:

```text
Do not expose arbitrary IAM roles by default.
Use predefined portable access levels.
Use least-privilege credentials for the controller.
Default bucket deletionPolicy to Retain.
Use finalizers.
Store external IDs in status.
Validate provider-specific config.
Never silently ignore unsupported fields.
Use status conditions for all important states.
```

## 26. Example End-to-End Flow

User creates a bucket:

```yaml
apiVersion: platform.example.com/v1alpha1
kind: Bucket
metadata:
  name: app-logs
  namespace: my-app
spec:
  providerRef:
    name: aws-dev
  name: app-logs-dev
  location: eu-central-1
  deletionPolicy: Retain
  versioning:
    enabled: true
```

User creates a principal:

```yaml
apiVersion: platform.example.com/v1alpha1
kind: CloudPrincipal
metadata:
  name: app-logs-writer
  namespace: my-app
spec:
  providerRef:
    name: aws-dev
  type: WorkloadIdentity
  name: app-logs-writer
  deletionPolicy: Delete
```

User grants access:

```yaml
apiVersion: platform.example.com/v1alpha1
kind: BucketAccess
metadata:
  name: app-logs-writer-access
  namespace: my-app
spec:
  bucketRef:
    name: app-logs

  principalRef:
    name: app-logs-writer

  access:
    level: ObjectWriter
```

Expected reconciliation:

```text
Bucket controller:
  creates or updates external bucket

CloudPrincipal controller:
  creates or updates external cloud principal

BucketAccess controller:
  waits for both resources
  validates provider match
  grants ObjectWriter access
  updates status
```

## 27. Recommended Initial Version

For version 1, implement:

```text
ProviderConfig
Bucket
CloudPrincipal
BucketAccess
```

Support a small portable bucket API:

```text
name
location
labels
versioning
simple lifecycle expiration rules
deletionPolicy
providerConfig.<cloud>
unsupportedFeaturePolicy
```

Support a small portable access API:

```text
ObjectReader
ObjectWriter
ObjectAdmin
BucketAdmin
```

Default safety choices:

```text
Bucket deletionPolicy: Retain
Unsupported feature policy: Fail
Drift policy: Correct
```

## 28. Future Extensions

Possible future additions:

```text
Higher-level ApplicationStorage CR
Cross-cloud access support
Advanced lifecycle transition rules
Object lock / retention policies
Encryption configuration
Replication configuration
Import/adoption workflows
Admission webhooks
Policy-as-code integration
Provider capability discovery endpoint
```

## 29. Final Recommendation

Use separate CRs:

```text
Bucket
CloudPrincipal
BucketAccess
```

Use a cloud-neutral API for common functionality, and provider-specific sections for features that are not portable.

The most important design principles are:

```text
Keep ownership boundaries clear.
Make reconciliation idempotent.
Validate unsupported features explicitly.
Do not silently ignore requested config.
Default destructive behavior to safe choices.
Expose actual external state in status.
Use provider interfaces internally.
Keep the user-facing API intent-based.
```

This design gives a clean Kubernetes-native API while still allowing each cloud provider to behave correctly behind the scenes.
