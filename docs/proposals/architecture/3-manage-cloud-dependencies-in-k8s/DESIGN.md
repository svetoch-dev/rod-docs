# Design details

## 1. Overview

This document describes the architecture for a Kubernetes controller/operator that manages object storage buckets, cloud IAM principals, authentication methods for those principals, and permissions between principals and buckets across multiple cloud providers.

The system is based on Kubernetes Custom Resources and follows a reconciliation model:

```text
Custom Resources
    ↓
Kubernetes Controllers
    ↓
Cloud Provider APIs
    ↓
Buckets / IAM Principals / Auth Bindings / IAM Grants
    ↓
CR Status
```

The main goals are:

- Provide a Kubernetes-native API for managing cloud storage access.
- Support multiple cloud providers.
- Keep user-facing APIs mostly cloud-neutral.
- Allow provider-specific configuration where needed.
- Separate identity, authentication, and authorization concerns.
- Avoid unsafe deletion behavior.
- Make reconciliation idempotent and drift-aware.
- Clearly report unsupported features and reconciliation status.

## 2. Goals

### 2.1 Functional Goals

The controller should manage:

- Object storage buckets.
- Cloud IAM principals.
- Authentication methods for cloud principals.
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
- Yandex Cloud

The same high-level Kubernetes API should work across providers where possible.

### 2.3 Safety Goals

The controller should:

- Use finalizers for external resource cleanup.
- Default bucket deletion to `Retain`.
- Avoid silently ignoring unsupported features.
- Avoid exposing raw provider IAM details by default.
- Prefer keyless workload identity over static credentials where possible.
- Clearly distinguish between owned and referenced resources.
- Store external IDs in status.

## 3. Non-Goals

This design does not aim to expose every provider-specific storage or IAM feature in the common API.

It also does not aim to make all clouds behave identically. Provider-specific differences are expected and should be handled explicitly.

This design does not require COSI as the primary API. COSI may be added later as an optional compatibility layer if needed.

## 4. High-Level Architecture

```text
ProviderConfig
    ↓
Bucket Controller
    manages external buckets

CloudPrincipal Controller
    manages external cloud identities

CloudPrincipalAuth Controller
    manages authentication method for cloud identities

BucketAccess Controller
    manages permissions between buckets and principals
```

Relationship between resources:

```text
CloudPrincipal CR ───────┬───────────────┐
                         ↓               ↓
              CloudPrincipalAuth CR   BucketAccess CR
                         ↓               ↓
              Workload Identity /      IAM binding /
              Static Credentials       bucket policy
                                         ↑
Bucket CR ──────────────────────────────┘
```

Conceptually:

```text
CloudPrincipal     = who exists in the cloud
CloudPrincipalAuth = how a workload authenticates as that principal
BucketAccess       = what that principal can do
Bucket             = the object storage resource
```

## 5. Custom Resources

The recommended core CRDs are:

```text
ProviderConfig
Bucket
CloudPrincipal
CloudPrincipalAuth
BucketAccess
```

Optional higher-level convenience CRDs can be added later, for example:

```text
ApplicationBucket
ApplicationStorageAccess
ApplicationCloudIdentity
```

These higher-level CRs can create lower-level `Bucket`, `CloudPrincipal`, `CloudPrincipalAuth`, and `BucketAccess` resources.

## 6. ProviderConfig

`ProviderConfig` describes the target cloud provider and the credentials/configuration required to manage resources there.

Example for GCP:

```yaml
apiVersion: vedro.svetoch.dev/v1alpha1
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

Example for AWS:

```yaml
apiVersion: vedro.svetoch.dev/v1alpha1
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

Example for Yandex Cloud:

```yaml
apiVersion: vedro.svetoch.dev/v1alpha1
kind: ProviderConfig
metadata:
  name: yc-dev
spec:
  type: Yandex
  yandex:
    cloudId: b1g-example-cloud-id
    folderId: b1g-example-folder-id
    region: ru-central1
    credentialsSecretRef:
      name: yc-creds
      namespace: platform-system
```

The controller should not require cloud credentials inside every resource. Resources should reference a `ProviderConfig`.

## 7. Bucket Resource

`Bucket` owns the lifecycle of an external object storage bucket.

Example:

```yaml
apiVersion: vedro.svetoch.dev/v1alpha1
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

  cloudSpecificConfig:
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
Yandex: service account
```

Example:

```yaml
apiVersion: vedro.svetoch.dev/v1alpha1
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

The `CloudPrincipal` controller should not manage bucket permissions directly.

The `CloudPrincipal` controller should not create workload identity bindings or static credentials directly. Those belong to `CloudPrincipalAuth`.

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

## 9. CloudPrincipalAuth Resource

`CloudPrincipalAuth` manages how a Kubernetes workload or consumer authenticates as a `CloudPrincipal`.

It is intentionally separate from `CloudPrincipal` because authentication lifecycle is different from identity lifecycle.

For example, a single cloud principal may have:

- one workload identity binding for an application Kubernetes ServiceAccount
- another workload identity binding for a job Kubernetes ServiceAccount
- zero or more static credentials if static keys are unavoidable

`CloudPrincipalAuth` supports both keyless authentication and static credentials under a single conceptual API.

### 9.1 Recommended Authentication Methods

Recommended initial methods:

```text
WorkloadIdentity
StaticCredentials
```

Prefer `WorkloadIdentity` where possible.

Static credentials should be treated as an escape hatch for systems that cannot use workload identity.

### 9.2 Workload Identity Example

```yaml
apiVersion: vedro.svetoch.dev/v1alpha1
kind: CloudPrincipalAuth
metadata:
  name: app-logs-writer-workload-auth
  namespace: my-app
spec:
  principalRef:
    name: app-logs-writer

  method: WorkloadIdentity

  workloadIdentity:
    kubernetesServiceAccountRef:
      name: app
      namespace: my-app

    serviceAccountMutationPolicy: ValidateOnly
```

`serviceAccountMutationPolicy` controls whether the controller only validates required Kubernetes ServiceAccount metadata or also patches it.

Recommended values:

```text
ValidateOnly
Patch
```

Recommended default:

```text
ValidateOnly
```

Behavior:

```text
ValidateOnly:
  Check that the referenced Kubernetes ServiceAccount has the required provider-specific annotations.
  If required annotations are missing, mark CloudPrincipalAuth Ready=False and do not mutate the ServiceAccount.

Patch:
  Patch the referenced Kubernetes ServiceAccount with the required provider-specific annotations.
  This mode should be used only when the platform team explicitly wants this controller to manage ServiceAccount metadata.
```

For GCP Workload Identity, the controller should require or apply the Kubernetes ServiceAccount annotation that points to the target GCP service account.

Example Kubernetes ServiceAccount metadata for GCP:

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: app
  namespace: my-app
  annotations:
    iam.gke.io/gcp-service-account: app-logs-writer@my-project.iam.gserviceaccount.com
```

Provider mappings:

```text
GCP:
  Kubernetes ServiceAccount can impersonate a GCP service account.

AWS:
  Kubernetes ServiceAccount can assume an IAM role through IRSA or Pod Identity.

Yandex:
  Provider-specific federation or workload identity mechanism, if supported.
```

### 9.3 Static Credentials Example

```yaml
apiVersion: vedro.svetoch.dev/v1alpha1
kind: CloudPrincipalAuth
metadata:
  name: app-logs-writer-static-auth
  namespace: my-app
spec:
  principalRef:
    name: app-logs-writer

  method: StaticCredentials

  staticCredentials:
    secretRef:
      name: app-logs-writer-cloud-creds

    rotation:
      enabled: true
      maxAgeDays: 90

    deletionPolicy: Delete
```

The resulting Kubernetes Secret should contain provider-specific credential material.

The exact Secret keys can be provider-specific, but should be documented and reflected in status.

### 9.4 CloudPrincipalAuth Responsibilities

The `CloudPrincipalAuth` controller should:

- Resolve the referenced `CloudPrincipal`.
- Wait until the referenced `CloudPrincipal` is ready.
- Build a provider client from the referenced principal's `providerRef`.
- Validate that the requested authentication method is supported by the provider.
- Resolve and validate the referenced Kubernetes ServiceAccount for `WorkloadIdentity`.
- Compute required provider-specific Kubernetes ServiceAccount annotations for `WorkloadIdentity`.
- Patch required Kubernetes ServiceAccount annotations only when `serviceAccountMutationPolicy: Patch` is set.
- Configure workload identity/federation bindings for `WorkloadIdentity`.
- Create, rotate, and revoke static credentials for `StaticCredentials`.
- Publish static credentials to a Kubernetes Secret or another configured secret backend.
- Store auth binding or credential metadata in status.
- Use finalizers to remove external auth bindings or revoke generated credentials.

### 9.5 CloudPrincipalAuth Status

Example for workload identity:

```yaml
status:
  method: WorkloadIdentity
  boundSubject:
    kind: KubernetesServiceAccount
    name: app
    namespace: my-app

  requiredServiceAccountMetadata:
    annotations:
      iam.gke.io/gcp-service-account: app-logs-writer@my-project.iam.gserviceaccount.com

  serviceAccountMutationPolicy: ValidateOnly

  conditions:
    - type: PrincipalReady
      status: "True"

    - type: ServiceAccountConfigured
      status: "True"
      reason: RequiredAnnotationsPresent

    - type: Ready
      status: "True"
      reason: WorkloadIdentityBound
      message: Workload identity binding is ready
```

Example status when the referenced Kubernetes ServiceAccount is missing required annotations and mutation is not enabled:

```yaml
status:
  method: WorkloadIdentity
  boundSubject:
    kind: KubernetesServiceAccount
    name: app
    namespace: my-app

  requiredServiceAccountMetadata:
    annotations:
      iam.gke.io/gcp-service-account: app-logs-writer@my-project.iam.gserviceaccount.com

  serviceAccountMutationPolicy: ValidateOnly

  conditions:
    - type: PrincipalReady
      status: "True"

    - type: ServiceAccountConfigured
      status: "False"
      reason: MissingRequiredAnnotation
      message: Kubernetes ServiceAccount my-app/app is missing annotation iam.gke.io/gcp-service-account

    - type: Ready
      status: "False"
      reason: KubernetesServiceAccountNotConfigured
```

Example for static credentials:

```yaml
status:
  method: StaticCredentials

  secretRef:
    name: app-logs-writer-cloud-creds

  credentialId: projects/my-project/serviceAccounts/app-logs-writer/keys/abc123
  lastRotatedAt: "2026-06-03T10:00:00Z"
  expiresAt: "2026-09-01T10:00:00Z"

  conditions:
    - type: PrincipalReady
      status: "True"

    - type: Ready
      status: "True"
      reason: StaticCredentialsCreated
```

### 9.6 Why CloudPrincipalAuth Should Be Separate

This separation keeps the model clean:

```text
CloudPrincipal     = identity lifecycle
CloudPrincipalAuth = authentication lifecycle
BucketAccess       = authorization lifecycle
```

This avoids making `CloudPrincipal` responsible for credential rotation, Secret publishing, workload identity configuration, and auth binding deletion.

It also allows multiple auth methods per principal without changing the principal itself.

## 10. BucketAccess Resource

`BucketAccess` manages the permission relationship between a bucket and a principal.

It should not create buckets or principals. It should only reference them.

Example:

```yaml
apiVersion: vedro.svetoch.dev/v1alpha1
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

### 10.1 BucketAccess Responsibilities

The `BucketAccess` controller should:

- Resolve the referenced `Bucket`.
- Resolve the referenced `CloudPrincipal`.
- Wait until both are ready.
- Validate that both use the same provider.
- Grant the requested access.
- Remove only the IAM binding/policy on deletion.
- Never delete the bucket or principal.
- Never manage authentication or credentials.

### 10.2 Access Levels

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

  Yandex:
    provider-specific object write permission
```

Raw provider roles should be avoided in the common API unless an advanced escape hatch is explicitly added.

### 10.3 BucketAccess Status

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

## 11. Bucket Attributes and Multi-Cloud Differences

Not all clouds support the same bucket features. The API should therefore be split into:

```text
1. Common portable fields
2. Platform-level abstractions
3. Provider-specific configuration
```

### 11.1 Common Portable Fields

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

### 11.2 Platform-Level Abstractions

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
Yandex: provider-specific cold/archive equivalent
```

### 11.3 Provider-Specific Configuration

Provider-specific fields should be namespaced under `spec.cloudSpecificConfig`.

Example:

```yaml
spec:
  cloudSpecificConfig:
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

    yandex:
      defaultStorageClass: STANDARD
      maxSizeBytes: 10737418240
```

Only the matching provider-specific section should be allowed.

For example, if `providerRef` points to GCP but `cloudSpecificConfig.aws` is set, the controller should fail validation:

```text
Reason: InvalidProviderConfig
Message: cloudSpecificConfig.aws is set but providerRef points to provider type GCP
```

## 12. Unsupported Feature Policy

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

## 13. Provider Capabilities

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

Authentication capabilities should also be explicit:

```go
type AuthCapabilities struct {
    WorkloadIdentity  bool
    StaticCredentials bool
    CredentialRotation bool
}
```

Access capabilities should also be explicit:

```go
type AccessCapabilities struct {
    ObjectReader bool
    ObjectWriter bool
    ObjectAdmin  bool
    BucketAdmin  bool
    PrefixScopedAccess bool
    ConditionalAccess  bool
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

Example auth behavior:

```text
User creates CloudPrincipalAuth with method WorkloadIdentity
Provider does not support workload identity
    ↓
CloudPrincipalAuth Ready=False
Reason=UnsupportedAuthMethod
Message=Provider does not support WorkloadIdentity for this principal type
```

## 14. Provider Interface

Use a common provider interface and provider-specific implementations.

Example bucket interface:

```go
type BucketProvider interface {
    ValidateBucketSpec(spec BucketSpec) ValidationResult
    EnsureBucket(ctx context.Context, spec BucketSpec) (*BucketState, error)
    DeleteBucket(ctx context.Context, status BucketStatus) error
}
```

Principal interface:

```go
type PrincipalProvider interface {
    ValidatePrincipalSpec(spec CloudPrincipalSpec) ValidationResult
    EnsurePrincipal(ctx context.Context, spec CloudPrincipalSpec) (*PrincipalState, error)
    DeletePrincipal(ctx context.Context, status CloudPrincipalStatus) error
}
```

Authentication interface:

```go
type PrincipalAuthProvider interface {
    ValidatePrincipalAuthSpec(spec CloudPrincipalAuthSpec) ValidationResult
    EnsurePrincipalAuth(
        ctx context.Context,
        principal PrincipalState,
        spec CloudPrincipalAuthSpec,
    ) (*PrincipalAuthState, error)
    DeletePrincipalAuth(ctx context.Context, status CloudPrincipalAuthStatus) error
}
```

Bucket access interface:

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
    case "Yandex":
        return yandex.New(cfg)
    default:
        return nil, fmt.Errorf("unsupported provider type %q", cfg.Spec.Type)
    }
}
```

## 15. Reconciliation Model

All operations must be idempotent.

The controller should use `Ensure*` methods, not blind `Create*` methods.

Good:

```text
EnsureBucket
EnsurePrincipal
EnsurePrincipalAuth
EnsureBucketAccess
```

Bad:

```text
CreateBucket
CreatePrincipal
CreateCredentials
CreateBucketAccess
```

Controllers must tolerate:

- retries
- restarts
- duplicate reconciliations
- already-existing resources
- eventual consistency
- manual drift

## 16. Bucket Controller Flow

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

## 17. CloudPrincipal Controller Flow

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

## 18. CloudPrincipalAuth Controller Flow

```text
1. Fetch CloudPrincipalAuth CR
2. Resolve referenced CloudPrincipal
3. Wait until CloudPrincipal is Ready
4. Build provider client from CloudPrincipal.providerRef
5. Validate requested auth method
6. Add finalizer if missing
7. If deleting:
     - remove workload identity binding or revoke generated credentials
     - optionally delete generated Secret depending on deletion policy
     - remove finalizer
8. For WorkloadIdentity:
     - fetch the referenced Kubernetes ServiceAccount
     - compute required provider-specific ServiceAccount annotations
     - if annotations are missing and serviceAccountMutationPolicy is ValidateOnly, set Ready=False
     - if annotations are missing and serviceAccountMutationPolicy is Patch, patch the ServiceAccount
9. Ensure auth method exists
10. Publish or update Secret if method is StaticCredentials
11. Update status
12. Set Ready=True
```

For `WorkloadIdentity`, the controller configures the relevant trust/federation relationship and validates or patches required Kubernetes ServiceAccount annotations.

For `StaticCredentials`, the controller creates, rotates, and revokes credential material and writes it to the configured Secret target.

## 19. BucketAccess Controller Flow

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

## 20. Watches

The `BucketAccess` controller should reconcile when any of these change:

```text
BucketAccess
Bucket
CloudPrincipal
```

When a `Bucket` changes, enqueue all `BucketAccess` resources that reference it.

When a `CloudPrincipal` changes, enqueue all `BucketAccess` resources that reference it.

The `CloudPrincipalAuth` controller should reconcile when any of these change:

```text
CloudPrincipalAuth
CloudPrincipal
Kubernetes ServiceAccount, if workload identity subject references it
Secret, if static credential output is managed by the controller
```

When a `CloudPrincipal` changes, enqueue all `CloudPrincipalAuth` resources that reference it.

## 21. Finalizers and Deletion Policies

External cloud resources must be cleaned up explicitly.

Recommended finalizers:

```text
vedro.svetoch.dev/bucket-finalizer
vedro.svetoch.dev/cloudprincipal-finalizer
vedro.svetoch.dev/cloudprincipalauth-finalizer
vedro.svetoch.dev/bucketaccess-finalizer
```

### 21.1 Bucket Deletion Policy

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

### 21.2 CloudPrincipal Deletion Policy

Recommended values:

```text
Retain
Delete
```

### 21.3 CloudPrincipalAuth Deletion

Deleting `CloudPrincipalAuth` should remove only the authentication mechanism it manages.

For `WorkloadIdentity`, it should remove the workload identity binding/federation relationship.

For `StaticCredentials`, it should revoke generated credentials. Secret deletion should be controlled by an explicit policy.

Recommended static credential Secret deletion policy values:

```text
Retain
Delete
```

Recommended default:

```text
Delete
```

### 21.4 BucketAccess Deletion

Deleting `BucketAccess` should only remove the access grant.

It should not delete:

- the bucket
- the principal
- any auth binding
- any generated credentials

## 22. Ownership Model

Use references between reusable resources.

Do not make `BucketAccess` own `Bucket` or `CloudPrincipal`, because the relationship is many-to-many.

```text
One bucket can have many access grants.
One principal can access many buckets.
One principal can have multiple auth methods.
```

Use `ownerReferences` only when a higher-level CR creates lower-level CRs.

Example:

```text
ApplicationStorage
    owns Bucket
    owns CloudPrincipal
    owns CloudPrincipalAuth
    owns BucketAccess
```

## 23. Drift Handling

The controller should detect and correct drift by default.

Example drift:

```text
Someone manually removes an IAM binding.
```

Expected behavior:

```text
BucketAccess controller restores the binding on next reconcile.
```

Example auth drift:

```text
Someone manually removes a workload identity binding.
```

Expected behavior:

```text
CloudPrincipalAuth controller restores the binding on next reconcile.
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

## 24. Naming

Do not blindly use Kubernetes `metadata.name` as the external cloud name.

Clouds have different naming constraints.

The controller should use a naming module:

```text
internal/naming/
  bucket.go
  principal.go
  auth.go
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

## 25. Recommended Code Layout

```text
cmd/
  main.go

api/
  v1alpha1/
    providerconfig_types.go
    bucket_types.go
    cloudprincipal_types.go
    cloudprincipalauth_types.go
    bucketaccess_types.go

internal/
  controllers/
    bucket_controller.go
    cloudprincipal_controller.go
    cloudprincipalauth_controller.go
    bucketaccess_controller.go


  cloud/
    provider.go
    registry.go

    gcp/
      provider.go
      buckets.go
      principals.go
      auth.go
      access.go

    aws/
      provider.go
      buckets.go
      principals.go
      auth.go
      access.go

    yandex/
      provider.go
      buckets.go
      principals.go
      auth.go
      access.go

  naming/
    bucket.go
    principal.go
    auth.go

  conditions/
    conditions.go

  validation/
    bucket.go
    principal.go
    auth.go
    access.go
```

## 26. Kubernetes RBAC

The controller needs permissions to:

- get/list/watch custom resources
- update status
- update finalizers
- read provider credential secrets
- read Kubernetes ServiceAccounts referenced by `CloudPrincipalAuth`
- patch Kubernetes ServiceAccounts only if `serviceAccountMutationPolicy: Patch` is supported and enabled
- create/update/delete Secrets for static credential output, if enabled
- create/patch events

Example:

```yaml
rules:
  - apiGroups: ["vedro.svetoch.dev"]
    resources:
      - providerconfigs
      - buckets
      - cloudprincipals
      - cloudprincipalauths
      - bucketaccesses
    verbs: ["get", "list", "watch"]

  - apiGroups: ["vedro.svetoch.dev"]
    resources:
      - buckets
      - cloudprincipals
      - cloudprincipalauths
      - bucketaccesses
    verbs: ["update", "patch"]

  - apiGroups: ["vedro.svetoch.dev"]
    resources:
      - buckets/status
      - cloudprincipals/status
      - cloudprincipalauths/status
      - bucketaccesses/status
    verbs: ["update", "patch"]

  - apiGroups: [""]
    resources: ["secrets"]
    verbs: ["get", "create", "update", "patch", "delete"]

  - apiGroups: [""]
    resources: ["serviceaccounts"]
    verbs: ["get", "list", "watch"]

  # Required only if CloudPrincipalAuth supports serviceAccountMutationPolicy: Patch.
  - apiGroups: [""]
    resources: ["serviceaccounts"]
    verbs: ["patch"]

  - apiGroups: [""]
    resources: ["events"]
    verbs: ["create", "patch"]
```

If static credentials are disabled, Secret write permissions can be removed or scoped down.

If `serviceAccountMutationPolicy: Patch` is not supported, or if the platform only wants validation, ServiceAccount patch permissions should be omitted.

## 27. Security Recommendations

The controller should follow these rules:

```text
Do not expose arbitrary IAM roles by default.
Use predefined portable access levels.
Prefer WorkloadIdentity over StaticCredentials.
Make StaticCredentials opt-in.
Support credential rotation if StaticCredentials are used.
Use least-privilege credentials for the controller.
Default bucket deletionPolicy to Retain.
Use finalizers.
Store external IDs in status.
Validate provider-specific config.
Never silently ignore unsupported fields.
Use status conditions for all important states.
```

For static credentials:

```text
Avoid static credentials unless required.
Write generated credentials only to explicitly requested destinations.
Never store credential material in CR status.
Expose only credential IDs, timestamps, and Secret references in status.
Support revocation on deletion.
Support rotation.
```

## 28. COSI Integration

COSI can be used as an optional compatibility layer, but it should not force the internal platform model to become weaker.

Recommended approach:

```text
Platform CRDs remain canonical:
  ProviderConfig
  Bucket
  CloudPrincipal
  CloudPrincipalAuth
  BucketAccess

Optional COSI adapter:
  COSI BucketClaim -> Bucket
  COSI BucketAccess -> CloudPrincipalAuth / BucketAccess
```

Avoid creating one COSI driver per bucket. Drivers should represent provider/back-end implementations, not individual buckets.

## 29. Example End-to-End Flow

User creates a bucket:

```yaml
apiVersion: vedro.svetoch.dev/v1alpha1
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
apiVersion: vedro.svetoch.dev/v1alpha1
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

User creates authentication for the principal:

```yaml
apiVersion: vedro.svetoch.dev/v1alpha1
kind: CloudPrincipalAuth
metadata:
  name: app-logs-writer-auth
  namespace: my-app
spec:
  principalRef:
    name: app-logs-writer

  method: WorkloadIdentity

  workloadIdentity:
    kubernetesServiceAccountRef:
      name: app
      namespace: my-app
```

User grants bucket access:

```yaml
apiVersion: vedro.svetoch.dev/v1alpha1
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

CloudPrincipalAuth controller:
  configures workload identity authentication for the principal

BucketAccess controller:
  waits for bucket and principal
  validates provider match
  grants ObjectWriter access
  updates status
```

## 30. Recommended Initial Version

For version 1, implement:

```text
ProviderConfig
Bucket
CloudPrincipal
CloudPrincipalAuth
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
cloudSpecificConfig.<cloud>
unsupportedFeaturePolicy
```

Support a small portable access API:

```text
ObjectReader
ObjectWriter
ObjectAdmin
BucketAdmin
```

Support one preferred auth API:

```text
CloudPrincipalAuth.method = WorkloadIdentity
```

Add static credentials only if needed:

```text
CloudPrincipalAuth.method = StaticCredentials
```

Default safety choices:

```text
Bucket deletionPolicy: Retain
Unsupported feature policy: Fail
Drift policy: Correct
Authentication preference: WorkloadIdentity
Static credentials: opt-in only
```

## 31. Future Extensions

Possible future additions:

```text
Higher-level ApplicationStorage CR
Higher-level ApplicationCloudIdentity CR
Cross-cloud access support
Advanced lifecycle transition rules
Object lock / retention policies
Encryption configuration
Replication configuration
Import/adoption workflows
Admission webhooks
Policy-as-code integration
Provider capability discovery endpoint
COSI compatibility adapter
External Secrets integration
Credential rotation controller
```

## 32. Final Recommendation

Use separate CRs:

```text
Bucket
CloudPrincipal
CloudPrincipalAuth
BucketAccess
```

Use a cloud-neutral API for common functionality, and provider-specific sections for features that are not portable.

Keep identity, authentication, and authorization separate:

```text
CloudPrincipal     = identity
CloudPrincipalAuth = authentication
BucketAccess       = authorization
```

The most important design principles are:

```text
Keep ownership boundaries clear.
Make reconciliation idempotent.
Validate unsupported features explicitly.
Do not silently ignore requested config.
Default destructive behavior to safe choices.
Prefer keyless auth over static credentials.
Expose actual external state in status.
Use provider interfaces internally.
Keep the user-facing API intent-based.
```

This design gives a clean Kubernetes-native API while still allowing each cloud provider to behave correctly behind the scenes.
