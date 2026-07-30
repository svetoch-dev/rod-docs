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
- Managed and referenced cloud IAM principals, including workload identities, users, and groups.
- Authentication methods for supported workload principals.
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
    manages or resolves cloud IAM authorization subjects

CloudPrincipalAuth Controller
    manages workload authentication for supported principal kinds

BucketAccess Controller
    manages permissions between buckets and principals
```

Relationship between resources:

```text
CloudPrincipal CR ───────┬───────────────┐
                         ↓               ↓
              CloudPrincipalAuth CR   BucketAccess CR
              (optional; workload      IAM binding /
               identities only)        bucket policy
                                         ↑
Bucket CR ──────────────────────────────┘
```

Conceptually:

```text
CloudPrincipal     = the cloud IAM subject that receives authorization
CloudPrincipalAuth = how a workload authenticates as a supported principal
BucketAccess       = what that principal can do to a bucket
Bucket             = the object storage resource
```

`CloudPrincipal` is an authorization-oriented abstraction. It may represent a controller-managed workload identity or a referenced external user or group.

`CloudPrincipalAuth` is not required for human users or groups. It applies only to principal kinds for which the controller manages workload authentication, such as service accounts and cloud roles.

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
  projectId: "dexfinance-internal"
  region: europe-west2
  method: StaticCredentials
  credentialsSecretRef:
    name: gcp-creds
    namespace: vedro-system
```

Example for AWS:

```yaml
apiVersion: vedro.svetoch.dev/v1alpha1
kind: ProviderConfig
metadata:
  name: aws-dev
spec:
  type: AWS
  projectId: "123456789012"
  method: StaticCredentials
  region: us-east-1
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
  projectId: b1g-example-folder-id
  region: ru-central1
  method: WorkloadIdentity
```

The controller should not require cloud credentials inside every resource. Resources should reference a `ProviderConfig`.


### 6.1 Provider Usage Policy

`ProviderConfig` defines a provider-level `usagePolicy`. This limits which Kubernetes namespaces may use the provider and which external buckets and principals may be managed or referenced.

```yaml
spec:
  usagePolicy:
    allowedNamespaces:
      names:
        - namespace1
        - namespace2

    bucketPolicy:
      allowExisting: false
      allowedNamePatterns:
        - vedro-.*

    principalPolicy:
      allowedKinds:
        - ServiceAccount
        - Role
        - User
        - Group
      allowManaged: true
      allowReferences: true
      allowAdoption: false
      allowedNamePatterns:
        - vedro-.*
      allowedReferencePatterns:
        - .*@example.com
```

`allowedNamespaces` defines which Kubernetes namespaces may reference this `ProviderConfig` from `Bucket`, `CloudPrincipal`, `CloudPrincipalAuth`, or `BucketAccess` resources.

Supported forms:

```yaml
allowedNamespaces:
  names:
    - namespace1
    - namespace2
```

```yaml
allowedNamespaces:
  all: true
```

`bucketPolicy.allowExisting` controls whether an existing bucket that is not owned by the same Kubernetes resource may be reconciled.

Principal policy fields:

```text
allowedKinds:
  Principal kinds that may be used through this provider.

allowManaged:
  Whether the controller may create and manage external principals.

allowReferences:
  Whether CloudPrincipal may reference existing external principals.

allowAdoption:
  Whether an existing principal may become controller-managed.
  Recommended default: false.

allowedNamePatterns:
  Allowed names for managed principals.

allowedReferencePatterns:
  Allowed external identifiers for referenced principals, such as user or group emails and provider IDs.
```

A referenced principal is intentionally external and is not considered an adopted resource. Therefore, `allowExisting` should not be used to model user or group references.

Recommended tenant-facing policy:

```yaml
principalPolicy:
  allowedKinds:
    - ServiceAccount
    - User
    - Group
  allowManaged: true
  allowReferences: true
  allowAdoption: false
  allowedNamePatterns:
    - vedro-.*
  allowedReferencePatterns:
    - .*@example.com
```

The validating webhook and reconcilers must enforce the same policy before cloud API calls are made.

### 6.2 Ownership Metadata for Managed Resources

For buckets with `allowExisting: false` and principals with `managementPolicy: Managed`, the controller must distinguish resources it created from arbitrary pre-existing resources with the same name. This should be enforced by writing controller-owned ownership metadata when external resources are created.

For tenant-facing providers, managed resources should have the following reconciliation behavior:

```text
External resource does not exist:
  Create it.
  Write ownership metadata.
  Reconcile it.

External resource exists and ownership metadata matches this Kubernetes resource:
  Reconcile it.

External resource exists and ownership metadata is missing:
  Fail reconciliation.
  Reason: ExternalResourceAlreadyExists

External resource exists and ownership metadata points to another Kubernetes resource:
  Fail reconciliation.
  Reason: ExternalResourceOwnedByAnotherResource
```

Recommended ownership metadata for external buckets:

```text
vedro.svetoch.dev/managed = true
vedro.svetoch.dev/kind = Bucket
vedro.svetoch.dev/namespace = <bucket namespace>
vedro.svetoch.dev/name = <bucket name>
vedro.svetoch.dev/uid = <bucket metadata.uid>
vedro.svetoch.dev/provider-config = <provider config name>
```

Recommended ownership metadata for managed external cloud principals:

```text
vedro.svetoch.dev/managed = true
vedro.svetoch.dev/kind = CloudPrincipal
vedro.svetoch.dev/namespace = <cloudprincipal namespace>
vedro.svetoch.dev/name = <cloudprincipal name>
vedro.svetoch.dev/uid = <cloudprincipal metadata.uid>
vedro.svetoch.dev/provider-config = <provider config name>
```

The `vedro.svetoch.dev/uid` value is the most important ownership value. Namespace and name are useful for debugging, but they are not sufficient as a stable ownership identity because Kubernetes resources can be deleted and recreated with the same namespace and name. The Kubernetes object UID is assigned by the API server and changes when the resource is recreated.

Provider-specific storage for ownership metadata may differ:

```text
GCP buckets:
  Use bucket labels for ownership metadata.

GCP service accounts:
  Use labels if supported by the chosen API path.
  Otherwise, store compact ownership metadata in the service account description.

AWS buckets, IAM roles, and IAM users:
  Use tags where supported.

Yandex Cloud buckets and service accounts:
  Use labels or provider-supported metadata fields where available.
```

Example compact description for a cloud principal when labels are not available:

```json
{"managedBy":"vedro","kind":"CloudPrincipal","namespace":"my-app","name":"app-logs-writer","uid":"7f2c...","providerConfig":"gcp-dev-apps"}
```

The controller should own the reserved metadata prefix. Users must not be allowed to set or override reserved ownership metadata through resource specs such as bucket labels, bucket tags, cloud principal labels, or provider-specific configuration. Admission should reject user-provided labels or tags that use reserved prefixes, for example:

```text
vedro.svetoch.dev/*
```

Ownership metadata is useful for preventing accidental adoption and for detecting whether an external resource is managed by the controller. It is not a complete security boundary if users also have direct cloud permissions to edit labels, tags, descriptions, IAM policies, or service account metadata on sensitive resources. Therefore, the full security model must also rely on Kubernetes admission, Kubernetes RBAC, and cloud-side IAM restrictions.

### 6.3 Name pattern semantics

```text
A bucket or principal name is allowed if it matches at least one configured pattern.
A bucket or principal name is denied if it matches none of the configured patterns.
An empty or missing pattern list should be treated as deny-all.
```

Example1:

```yaml
apiVersion: vedro.svetoch.dev/v1alpha1
kind: ProviderConfig
metadata:
  name: gcp-dev-apps
spec:
  type: GCP
  projectId: my-awesome-project

  usagePolicy:
    allowedNamespaces:
      names:
        - namespace1
        - namespace2

    bucketPolicy:
      allowExisting: false
      allowedNamePatterns:
        - vedro-.*

    principalPolicy:
      allowedKinds:
        - ServiceAccount
        - User
        - Group
      allowManaged: true
      allowReferences: true
      allowAdoption: false
      allowedNamePatterns:
        - vedro-.*
      allowedReferencePatterns:
        - .*@example.com
```

Example2:

```yaml
apiVersion: vedro.svetoch.dev/v1alpha1
kind: ProviderConfig
metadata:
  name: gcp-dev-infra
spec:
  type: GCP
  projectId: my-awesome-project

  usagePolicy:
    allowedNamespaces:
      all: true

    bucketPolicy:
      allowedNamePatterns:
        - .*

    principalPolicy:
      allowedKinds:
        - ServiceAccount
        - Role
        - User
        - Group
      allowManaged: true
      allowReferences: true
      allowAdoption: true
      allowedNamePatterns:
        - .*
      allowedReferencePatterns:
        - .*
```

The `gcp-dev-apps` provider restricts managed principal names to `vedro-*`, allows only approved referenced identities, and disables adoption. Application users therefore cannot make the controller take ownership of arbitrary existing principals.

The `gcp-dev-infra` provider is intentionally permissive and should be treated as platform or infrastructure-only. Because it allows all namespaces and all bucket/principal names, Kubernetes RBAC and admission policy should prevent ordinary application users from referencing it unless that is explicitly intended.

The controller should enforce `usagePolicy` during reconciliation. A validating admission webhook should also enforce the same policy when resources are created or updated, so unsafe resources are rejected before they are persisted.

Example validation behavior:

```text
User creates Bucket in namespace namespace1 with providerRef gcp-dev-apps and name vedro-logs
    ↓
Allowed, because namespace1 is listed in allowedNamespaces.names and vedro-logs matches vedro-.*

User creates Bucket in namespace namespace3 with providerRef gcp-dev-apps
    ↓
Denied, because namespace3 is not listed in allowedNamespaces.names

User creates CloudPrincipal in namespace namespace1 with providerRef gcp-dev-apps and name admin-service-account
    ↓
Denied, because admin-service-account does not match vedro-.*

User creates Bucket in namespace namespace1 with providerRef gcp-dev-apps and name vedro-existing-prod
External bucket vedro-existing-prod already exists but is not owned by the same Bucket resource
    ↓
Denied or reconciled as Ready=False, because bucketPolicy.allowExisting is false

User creates Bucket with providerRef gcp-dev-infra
    ↓
Allowed only if the requester is authorized to use the infrastructure provider
```

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

`CloudPrincipal` represents a cloud IAM authorization subject that may receive bucket access.

Supported portable kinds:

```text
ServiceAccount
Role
User
Group
```

Provider mappings may differ:

```text
GCP:
  ServiceAccount, User, Group

AWS:
  Role, User
  Group is not a valid principal in an S3 bucket policy and is initially unsupported.

Yandex Cloud:
  ServiceAccount, User, Group
```

`CloudPrincipal` has two independent dimensions:

```text
kind:
  What kind of IAM subject this is.

managementPolicy:
  Whether the controller owns the external identity or only references it.
```

Recommended management policies:

```text
Managed
Reference
```

### 8.1 Managed Principal Example

```yaml
apiVersion: vedro.svetoch.dev/v1alpha1
kind: CloudPrincipal
metadata:
  name: app-logs-writer
  namespace: my-app
spec:
  providerRef:
    name: gcp-dev

  kind: ServiceAccount
  managementPolicy: Managed

  managed:
    name: app-logs-writer
    deletionPolicy: Delete
```

A managed principal is created, updated, and optionally deleted by the controller.

Initially, managed principal support should be limited to workload-oriented identities:

```text
GCP: ServiceAccount
AWS: Role, optionally User
Yandex Cloud: ServiceAccount
```

The controller should not create or manage human users or groups in the initial implementation.

### 8.2 Referenced User Example

```yaml
apiVersion: vedro.svetoch.dev/v1alpha1
kind: CloudPrincipal
metadata:
  name: alice
  namespace: my-app
spec:
  providerRef:
    name: gcp-dev

  kind: User
  managementPolicy: Reference

  reference:
    externalId: alice@example.com
    verificationPolicy: BestEffort
```

### 8.3 Referenced Group Example

```yaml
apiVersion: vedro.svetoch.dev/v1alpha1
kind: CloudPrincipal
metadata:
  name: developers
  namespace: my-app
spec:
  providerRef:
    name: gcp-dev

  kind: Group
  managementPolicy: Reference

  reference:
    externalId: developers@example.com
    verificationPolicy: BestEffort
```

Recommended verification policies:

```text
Required:
  The provider must verify that the external principal exists.

BestEffort:
  Verify when the provider API and controller credentials support it.
  Otherwise accept the identifier after syntax and usage-policy validation.

None:
  Treat the identifier as opaque after local validation.
```

Recommended default for human users and groups:

```text
BestEffort
```

Directory-level lookup permissions should not be required merely to grant bucket access when the cloud IAM API accepts a stable principal identifier directly.

### 8.4 CloudPrincipal Spec Shape

```go
type PrincipalKind string

const (
    PrincipalKindServiceAccount PrincipalKind = "ServiceAccount"
    PrincipalKindRole           PrincipalKind = "Role"
    PrincipalKindUser           PrincipalKind = "User"
    PrincipalKindGroup          PrincipalKind = "Group"
)

type PrincipalManagementPolicy string

const (
    PrincipalManagementPolicyManaged   PrincipalManagementPolicy = "Managed"
    PrincipalManagementPolicyReference PrincipalManagementPolicy = "Reference"
)

type CloudPrincipalSpec struct {
    ProviderRef ProviderReference `json:"providerRef"`

    Kind             PrincipalKind             `json:"kind"`
    ManagementPolicy PrincipalManagementPolicy `json:"managementPolicy"`

    Managed   *ManagedPrincipalSpec    `json:"managed,omitempty"`
    Reference *ReferencedPrincipalSpec `json:"reference,omitempty"`
}

type ManagedPrincipalSpec struct {
    Name           string         `json:"name,omitempty"`
    DeletionPolicy DeletionPolicy `json:"deletionPolicy,omitempty"`
}

type ReferencedPrincipalSpec struct {
    ExternalID        string             `json:"externalId"`
    VerificationPolicy VerificationPolicy `json:"verificationPolicy,omitempty"`
}
```

Validation must enforce exactly one branch:

```text
managementPolicy=Managed:
  managed must be set
  reference must be absent

managementPolicy=Reference:
  reference must be set
  managed must be absent
```

### 8.5 CloudPrincipal Responsibilities

The `CloudPrincipal` controller should:

- Validate the requested principal kind against provider capabilities.
- Validate the requested management policy against `ProviderConfig.usagePolicy`.
- For `Managed`, ensure the external principal exists and apply ownership metadata.
- For `Reference`, resolve or validate the provider-native principal identifier without taking ownership.
- Never create, modify, or delete referenced users or groups.
- Normalize the identifier used by provider IAM APIs.
- Store both user-facing and provider-native identifiers in status.
- Apply deletion policy only to managed principals.

### 8.6 CloudPrincipal Status

Example for a referenced GCP group:

```yaml
status:
  kind: Group
  managementPolicy: Reference
  externalId: developers@example.com
  providerPrincipalId: group:developers@example.com
  observedProvider: GCP

  conditions:
    - type: Ready
      status: "True"
      reason: PrincipalResolved
```

Example for a managed AWS role:

```yaml
status:
  kind: Role
  managementPolicy: Managed
  externalName: app-logs-writer
  externalId: arn:aws:iam::123456789012:role/app-logs-writer
  providerPrincipalId: arn:aws:iam::123456789012:role/app-logs-writer
  observedProvider: AWS

  conditions:
    - type: Ready
      status: "True"
      reason: Reconciled
```

The fields have distinct purposes:

```text
externalId:
  Stable external identifier exposed to the user.

providerPrincipalId:
  Exact identifier used in provider IAM bindings or policies.
```

## 9. CloudPrincipalAuth Resource

`CloudPrincipalAuth` manages how a Kubernetes workload authenticates as a supported `CloudPrincipal`.

It is intentionally separate because authentication lifecycle differs from identity lifecycle.

`CloudPrincipalAuth` is valid only for workload-oriented principal kinds supported by the provider, normally:

```text
ServiceAccount
Role
```

It must be rejected for:

```text
User
Group
```

Human users and groups receive authorization through `BucketAccess`, but their login, federation, MFA, membership, and credentials remain outside this controller.

### 9.1 Recommended Authentication Methods

Recommended initial methods:

```text
WorkloadIdentity
StaticCredentials
```

Prefer `WorkloadIdentity` where possible. Static credentials remain an opt-in escape hatch.

Compatibility should be validated as a provider-specific matrix:

```text
ServiceAccount:
  WorkloadIdentity: commonly supported
  StaticCredentials: provider-dependent

Role:
  WorkloadIdentity/Federation: provider-dependent
  StaticCredentials: normally unsupported

User:
  Unsupported

Group:
  Unsupported
```

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

The referenced `CloudPrincipal` must be a compatible kind such as `ServiceAccount` or `Role`.

`serviceAccountMutationPolicy` values:

```text
ValidateOnly
Patch
```

Recommended default:

```text
ValidateOnly
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

Static credentials may only be generated for principal kinds and providers that explicitly support them.

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

`BucketAccess` manages the authorization relationship between a bucket and a `CloudPrincipal`.

It supports any principal kind that the provider can grant bucket access to, including referenced users and groups.

Example granting read access to a user:

```yaml
apiVersion: vedro.svetoch.dev/v1alpha1
kind: BucketAccess
metadata:
  name: alice-reports-reader
  namespace: my-app
spec:
  bucketRef:
    name: reports

  principalRef:
    name: alice

  access:
    level: ObjectReader
```

Example granting read access to a group:

```yaml
apiVersion: vedro.svetoch.dev/v1alpha1
kind: BucketAccess
metadata:
  name: developers-reports-reader
  namespace: my-app
spec:
  bucketRef:
    name: reports

  principalRef:
    name: developers

  access:
    level: ObjectReader
```

### 10.1 BucketAccess Responsibilities

The `BucketAccess` controller should:

- Resolve the referenced `Bucket`.
- Resolve the referenced `CloudPrincipal`.
- Wait until both are ready.
- Validate that both use the same provider.
- Validate that the provider supports bucket access for the principal kind.
- Map the portable access level to provider-native roles or policy statements.
- Ensure the exact desired binding exists.
- Avoid unnecessary cloud writes when the binding already exists.
- Remove only the binding owned by this `BucketAccess` on deletion.
- Never delete the bucket, principal, authentication binding, user, or group.

The controller should call `EnsureBucketAccess` on every reconcile. Provider implementations should observe current IAM state and mutate only when required.

Recommended provider behavior:

```text
GCP:
  Read the bucket IAM policy.
  Check for the exact member and role.
  Update the policy only when missing.
  Preserve unrelated bindings and use IAM concurrency metadata.

Yandex Cloud:
  List access bindings for the target bucket/resource.
  Check the exact subject type, subject ID, and role.
  Add the binding only when missing.

AWS:
  Reconcile deterministic bucket-policy statements for Role or User principals.
  IAM groups are initially unsupported as S3 bucket-policy principals.
```

### 10.2 Access Levels

Portable initial levels:

```text
ObjectReader
ObjectWriter
ObjectAdmin
BucketAdmin
```

Raw provider roles should not be exposed in the common API by default.

### 10.3 Access-Level Changes

When the desired access level changes, the controller must not leave an obsolete grant behind.

Recommended sequence:

```text
1. Ensure the new grant exists.
2. Remove the previously applied grant when it differs.
3. Update status after both operations succeed.
```

Granting the new role first avoids a temporary loss of access.

Status should record the applied provider role and principal identifier:

```yaml
status:
  applied:
    principalKind: User
    providerPrincipalId: user:alice@example.com
    providerRole: roles/storage.objectViewer
```

### 10.4 BucketAccess Status

```yaml
status:
  bindingId: bucket/reports:user/alice@example.com:ObjectReader

  applied:
    principalKind: User
    providerPrincipalId: user:alice@example.com
    providerRole: roles/storage.objectViewer

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

Provider capabilities must distinguish principal creation, principal reference, authentication, and authorization.

```go
type PrincipalCapabilities struct {
    ManagedKinds    map[PrincipalKind]bool
    ReferencedKinds map[PrincipalKind]bool
    AccessKinds     map[PrincipalKind]bool

    AuthMethods map[PrincipalKind]map[AuthMethod]bool
}
```

This answers four different questions:

```text
Can the provider implementation create this principal kind?
Can it reference an existing principal of this kind?
Can it configure authentication for this principal kind?
Can this principal kind receive bucket access?
```

Example initial capability matrix:

```text
GCP:
  Managed: ServiceAccount
  Referenced: ServiceAccount, User, Group
  Bucket access: ServiceAccount, User, Group
  Auth: ServiceAccount

AWS:
  Managed: Role, optionally User
  Referenced: Role, User
  Bucket access: Role, User
  Auth: Role
  Group: unsupported for S3 bucket-policy authorization

Yandex Cloud:
  Managed: ServiceAccount
  Referenced: ServiceAccount, User, Group
  Bucket access: ServiceAccount, User, Group
  Auth: ServiceAccount where supported
```

Bucket, authentication, and access capabilities remain explicit:

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

type AccessCapabilities struct {
    ObjectReader       bool
    ObjectWriter       bool
    ObjectAdmin        bool
    BucketAdmin        bool
    PrefixScopedAccess bool
    ConditionalAccess  bool
}
```

Unsupported principal-kind combinations must produce explicit conditions such as:

```text
Reason: UnsupportedPrincipalKind
Message: AWS IAM groups cannot be used as principals in S3 bucket policies
```

## 14. Provider Interface

Use common provider interfaces and provider-specific implementations.

```go
type PrincipalState struct {
    Kind                PrincipalKind
    ManagementPolicy    PrincipalManagementPolicy
    ExternalName        string
    ExternalID          string
    ProviderPrincipalID string
    Provider            ProviderType
}
```

Principal interface:

```go
type PrincipalProvider interface {
    ValidatePrincipalSpec(spec CloudPrincipalSpec) ValidationResult

    EnsureManagedPrincipal(
        ctx context.Context,
        spec CloudPrincipalSpec,
    ) (*PrincipalState, error)

    ResolveReferencedPrincipal(
        ctx context.Context,
        spec CloudPrincipalSpec,
    ) (*PrincipalState, error)

    DeleteManagedPrincipal(
        ctx context.Context,
        status CloudPrincipalStatus,
    ) error
}
```

Authentication interface:

```go
type PrincipalAuthProvider interface {
    ValidatePrincipalAuthSpec(
        principal PrincipalState,
        spec CloudPrincipalAuthSpec,
    ) ValidationResult

    EnsurePrincipalAuth(
        ctx context.Context,
        principal PrincipalState,
        spec CloudPrincipalAuthSpec,
    ) (*PrincipalAuthState, error)

    DeletePrincipalAuth(
        ctx context.Context,
        status CloudPrincipalAuthStatus,
    ) error
}
```

Bucket access interface:

```go
type BucketAccessProvider interface {
    ValidateBucketAccessSpec(
        principal PrincipalState,
        spec BucketAccessSpec,
    ) ValidationResult

    EnsureBucketAccess(
        ctx context.Context,
        bucket BucketState,
        principal PrincipalState,
        access AccessSpec,
    ) (*AccessState, error)

    RevokeBucketAccess(
        ctx context.Context,
        bucket BucketState,
        principal PrincipalState,
        applied AppliedAccessState,
    ) error
}
```

`EnsureBucketAccess` is called on every reconciliation, but should avoid unnecessary writes:

```text
observe current IAM state
compare with desired binding
mutate only when missing or different
```

Provider implementations:

```text
internal/cloud/gcp
internal/cloud/aws
internal/cloud/yandex
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
1. Fetch CloudPrincipal CR.
2. Fetch ProviderConfig.
3. Validate namespace and principal usage policy.
4. Validate principal kind and management policy against provider capabilities.
5. Add finalizer when required.
6. If deleting:
     - Managed: apply deletion policy and remove finalizer.
     - Reference: remove finalizer without touching the external principal.
7. If managementPolicy=Managed:
     - ensure the external principal exists
     - verify ownership metadata
8. If managementPolicy=Reference:
     - validate the external identifier
     - resolve or verify it according to verificationPolicy
     - never take ownership
9. Normalize providerPrincipalId.
10. Update status.
11. Set Ready=True.
```

## 18. CloudPrincipalAuth Controller Flow

```text
1. Fetch CloudPrincipalAuth CR.
2. Resolve referenced CloudPrincipal.
3. Wait until CloudPrincipal is Ready.
4. Reject principal kinds that cannot authenticate, including User and Group.
5. Build provider client from CloudPrincipal.providerRef.
6. Validate the requested auth method against provider and principal-kind capabilities.
7. Add finalizer if missing.
8. If deleting:
     - remove workload identity binding or revoke generated credentials
     - remove finalizer
9. For WorkloadIdentity:
     - fetch the referenced Kubernetes ServiceAccount
     - validate or patch required metadata
     - ensure the provider trust/federation relationship
10. For StaticCredentials:
     - create or rotate credentials
     - publish them to the configured Secret destination
11. Update status.
12. Set Ready=True.
```

## 19. BucketAccess Controller Flow

```text
1. Fetch BucketAccess CR.
2. Resolve referenced Bucket.
3. Resolve referenced CloudPrincipal.
4. Wait until both are Ready.
5. Check both resources use the same provider.
6. Validate provider support for the principal kind and access level.
7. Add finalizer if missing.
8. If deleting:
     - revoke only the previously applied IAM binding or policy statement
     - remove finalizer
9. Call EnsureBucketAccess.
10. Provider implementation reads current IAM state and writes only when needed.
11. If desired access differs from status.applied:
      - ensure the new grant
      - revoke the old applied grant
12. Update status.applied and conditions.
13. Set Ready=True.
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

Deletion behavior depends on `managementPolicy`.

```text
Managed:
  Apply managed.deletionPolicy.

Reference:
  Never delete or modify the external principal.
  Remove only Kubernetes-side bookkeeping and finalizers.
```

Recommended managed deletion policies:

```text
Retain
Delete
```

Human users and groups should normally use `managementPolicy: Reference` and therefore remain unaffected when the Kubernetes resource is deleted.

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
One managed workload principal can have multiple auth methods.
A referenced user or group normally has no CloudPrincipalAuth resource.
```

Ownership metadata applies only to `managementPolicy: Managed` principals.

Referenced users, groups, roles, and service accounts are not adopted, tagged, modified, or deleted by the `CloudPrincipal` controller.

Use `ownerReferences` only when a higher-level CR creates lower-level resources.

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
Default bucket deletionPolicy to Retain.
Use finalizers.
Store external IDs, never credential material, in status.
Validate provider-specific configuration.
Never silently ignore unsupported fields or principal kinds.
```

Principal-specific safety rules:

```text
Treat Managed and Reference as different lifecycle modes.
Do not create or delete human users or groups in the initial implementation.
Do not manage group membership.
Do not require directory-admin permissions merely to use a user or group as an IAM subject.
Restrict referenced external IDs through ProviderConfig usage policy.
Reject CloudPrincipalAuth for User and Group kinds.
Reject unsupported provider combinations, such as AWS Group for S3 bucket policies.
Never mutate referenced principal metadata.
```

Bucket-access reconciliation rules:

```text
Read and compare current IAM state before writing.
Preserve bindings owned by other systems.
Use concurrency controls and retry conflicts.
Avoid unconditional policy writes on every reconcile.
Record the applied provider role and principal identifier in status.
On access-level changes, grant the new role before revoking the old role.
On deletion, revoke only the binding represented by the BucketAccess resource.
```

Usage policy must be enforced by admission and again during reconciliation.

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

## 29. Example End-to-End Flows

### 29.1 Managed Workload Principal

```yaml
apiVersion: vedro.svetoch.dev/v1alpha1
kind: CloudPrincipal
metadata:
  name: app-logs-writer
  namespace: my-app
spec:
  providerRef:
    name: gcp-dev
  kind: ServiceAccount
  managementPolicy: Managed
  managed:
    name: app-logs-writer
    deletionPolicy: Delete
```

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

### 29.2 Referenced Cloud User

```yaml
apiVersion: vedro.svetoch.dev/v1alpha1
kind: CloudPrincipal
metadata:
  name: alice
  namespace: my-app
spec:
  providerRef:
    name: gcp-dev
  kind: User
  managementPolicy: Reference
  reference:
    externalId: alice@example.com
    verificationPolicy: BestEffort
```

```yaml
apiVersion: vedro.svetoch.dev/v1alpha1
kind: BucketAccess
metadata:
  name: alice-app-logs-reader
  namespace: my-app
spec:
  bucketRef:
    name: app-logs
  principalRef:
    name: alice
  access:
    level: ObjectReader
```

No `CloudPrincipalAuth` is created for the user.

### 29.3 Referenced Cloud Group

```yaml
apiVersion: vedro.svetoch.dev/v1alpha1
kind: CloudPrincipal
metadata:
  name: developers
  namespace: my-app
spec:
  providerRef:
    name: yc-dev
  kind: Group
  managementPolicy: Reference
  reference:
    externalId: aje-example-group-id
    verificationPolicy: BestEffort
```

```yaml
apiVersion: vedro.svetoch.dev/v1alpha1
kind: BucketAccess
metadata:
  name: developers-app-logs-reader
  namespace: my-app
spec:
  bucketRef:
    name: app-logs
  principalRef:
    name: developers
  access:
    level: ObjectReader
```

No `CloudPrincipalAuth` is created for the group, and the controller does not manage group membership.

## 30. Recommended Initial Version

For version 1, implement:

```text
ProviderConfig
Bucket
CloudPrincipal
CloudPrincipalAuth
BucketAccess
```

Principal model:

```text
CloudPrincipal.kind:
  ServiceAccount
  Role
  User
  Group

CloudPrincipal.managementPolicy:
  Managed
  Reference
```

Initial support matrix:

```text
GCP:
  Managed ServiceAccount
  Referenced ServiceAccount, User, Group
  Bucket access for ServiceAccount, User, Group

AWS:
  Managed/Referenced Role
  Referenced User
  Bucket access for Role and User
  Group unsupported initially

Yandex Cloud:
  Managed ServiceAccount
  Referenced ServiceAccount, User, Group
  Bucket access for ServiceAccount, User, Group
```

Authentication:

```text
CloudPrincipalAuth only for ServiceAccount and Role where supported.
WorkloadIdentity preferred.
StaticCredentials opt-in only.
User and Group rejected as unsupported authentication subjects.
```

Bucket access reconciliation:

```text
Call EnsureBucketAccess on every reconcile.
Read current provider IAM state.
Write only when the desired binding is missing or differs.
Preserve unrelated bindings.
Track applied role and provider principal ID in status.
```

Default safety choices:

```text
Bucket deletionPolicy: Retain
Unsupported feature policy: Fail
Drift policy: Correct
Referenced-principal verification: BestEffort
Managed human users/groups: unsupported
Group membership management: out of scope
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
