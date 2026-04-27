# Creating root modules for DNS and registry definitions (❌ NOT IMPLEMENTED)

## Current state

DNS zones and container registries are currently created inside the cloud-specific modules:

```
modules/rod/cloud/{yc,gcp}
```

This makes DNS and registry management tightly coupled to the selected cloud provider.

## Issues

Different projects may have different infrastructure requirements or constraints. For example:

* DNS must be managed by Cloudflare, GoDaddy, Akamai, or another external provider
* A self-hosted container registry must be used
* GHCR must be used instead of a cloud-native registry
* DNS and registry providers may differ from the main cloud provider

The current structure makes these cases difficult to support because DNS and registry resources are embedded inside cloud modules.

## Desired state

### 1. Extend `terraform.tfvars.json` schema

Add dedicated `dns` and `registry` objects:

```json
{
  "registry": {
    "type": "gar",
    "url": "{env.cloud.location.region}-docker.pkg.dev/{env.cloud.id}/containers"
  },
  "dns": {
    "domain": "{env.short_name}.{company.domain}",
    "type": "gcp"
  }
}
```

### 2. Add provider-specific `rod` modules

Create standalone modules for DNS and registry providers:

```
modules/rod/dns/<provider>
modules/rod/registry/<provider>
```

Examples:

```
modules/rod/dns/gcp
modules/rod/dns/cloudflare
modules/rod/registry/gar
modules/rod/registry/ghcr
```

### 3. Add per-environment root modules in infrastructure repos

Create dedicated root modules for DNS and registry per environment:

```
terraform/environments/<env>/dns
terraform/environments/<env>/registry
```

### 4. Reference the selected DNS/registry provider module

Example:

```hcl
# terraform/environments/production/dns/main.tf.tpl

module "dns" {
  source = "git::https://github.com/svetoch-dev/tf-modules.git//modules/rod/dns/{env.dns.type}?ref=v0.16.0"

  company   = var.company
  ci        = var.ci
  int_env   = var.envs.internal
  env       = local.env
  overrides = local.overrides
}
```

## Benefits

* Decouples DNS and registry management from cloud provider modules
* Supports external DNS providers such as Cloudflare, GoDaddy, and Akamai
* Supports non-cloud or self-hosted container registries
* Allows projects to enforce provider-specific requirements
* Makes DNS and registry implementations easier to extend independently
* Keeps cloud modules focused on core cloud infrastructure

## Disadvantages

* Requires migration from the current cloud-module-based implementation
* Adds additional root modules per environment
* Requires schema changes in `terraform.tfvars.json`
* Existing projects may need state migration or resource imports
