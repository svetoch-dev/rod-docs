# Tf rod module inheritance (❌ NOT IMPLEMENTED)

## Current state

Currently our tf rod cloud module structure looks like this

```
                                  |                                                              -> tf-modules/aws/s3
                                  |-> tf-modules/modules/rod/cloud/aws -> tf-modules/modules/aws -> tf-modules/aws/networking
                                  |                                                              -> ...
                                  |                                    
                                  |                                     
                                  |                                                              -> tf-modules/modules/gcp/networking
environtments/<some_env>/cloud -> |-> tf-modules/modules/rod/cloud/gcp -> tf-modules/modules/gcp -> tf-modules/modules/gcp/gcs
                                  |                                                              -> ...
                                  |                                    
                                  |                                     
                                  |                                                              -> tf-modules/modules/yc/ycs
                                  |-> tf-modules/modules/rod/cloud/yc  -> tf-modules/modules/yc  -> tf-modules/modules/yc/networking
                                  |                                                              -> ...

```

`<some_env>` can be of 2 types 

* Internal (monitoring, observability, operational tools)
* Product (dev,prd,pre,sandbox etc)

Those 2 type of envs have common resources like (k8s clusters, loki bucket, thanos bucket etc) and resources that are created in a specific env type (example: Internal environment needs argocd service account, internal environment needs runner k8s node pool etc)

To solve this we add logic to our `tf-modules/modules/rod/cloud/{cloud}` modules that looks like this

```
argocd = var.env.short_name == "int" ? {
  description = "argocd service account"
  roles = [
  ]
  ....
  generate_key = false
} : null,
```

```
locals {
  gcp_registries = {
    containers = {
      location = var.env.cloud.location.region
      ...
      writers = var.env.short_name != "int" ? [
        "serviceAccount:runner-app@${var.int_env.cloud.id}.iam.gserviceaccount.com"
      ] : []
    }
  }
}
```

## Issues

Adding logic like described in the `Current state` section will quickly become cumbersome and error prone. Especially dealing with increaseing amount of clouds we want to support

## Desired State

To address these issues, we 

1. Store common resources in `tf-modules/modules/rod/cloud/{cloud}`

2. We introduce new modules
   * `tf-modules/modules/rod/cloud/{cloud}/internal` where we
     * source cloud module to get common resource
     * use overrides to set internal specific resource
   * `tf-modules/modules/rod/cloud/{cloud}/product`
     * source cloud module to get common resource
     * use overrides to set product pecific resource




Module structure will look like this

```
                           |                                                                           -> modules/aws/s3
                           |-> modules/rod/cloud/aws/internal -> modules/rod/cloud/aws -> modules/aws  -> modules/aws/networking
                           |                                                                           -> ...
                           |                                                           
                           |                                                           
                           |                                                                           -> modules/gcp/networking
environtments/int/cloud    |-> modules/rod/cloud/gcp/internal -> modules/rod/cloud/gcp -> modules/gcp  -> modules/gcp/gcs
                           |                                                                           -> ...
                           |                                                           
                           |                                                           
                           |                                                                           -> modules/yc/ycs
                           |-> modules/rod/cloud/yc/internal  -> modules/rod/cloud/yc  -> modules/yc   -> modules/yc/networking
                           |                                                                           -> ...



                           |                                                                          -> modules/aws/s3
                           |-> modules/rod/cloud/aws/product -> modules/rod/cloud/aws -> modules/aws  -> modules/aws/networking
                           |                                                                          -> ...
                           |                                                           
                           |                                                           
                           |                                                                          -> modules/gcp/networking
environtments/prd/cloud    |-> modules/rod/cloud/gcp/product -> modules/rod/cloud/gcp -> modules/gcp  -> modules/gcp/gcs
                           |                                                                          -> ...
                           |                                                           
                           |                                                           
                           |                                                                          -> modules/yc/ycs
                           |-> modules/rod/cloud/yc/product  -> modules/rod/cloud/yc  -> modules/yc   -> modules/yc/networking
                           |                                                                          -> ...

```


### Benefits

- Instead of adding more and more logic to `modules/rod/cloud/{cloud}` we inherit from it and override what we need
- Code becomes easier to understand

### Disadvantages

- Need to duplicate input variables in `modules/rod/cloud/{cloud}`, `modules/rod/cloud/{cloud}/internal`, `modules/rod/cloud/{cloud}/product`

### Setup


1. `modules/rod/cloud/gcp/internal` main.tf file

```
...
module "cloud" {
  source = "../"
  company   = var.company
  ci        = var.ci
  int_env   = var.internal_env
  env       = var.env
  overrides = local.overrides
}
```

2. `modules/rod/cloud/gcp/internal` overrides.tf file

```
locals {
  overrides_internal = {
    gcp_iam = {
      service_accounts = {
        argocd = {
          description = "argocd service account"
          ....
        }
      }
    }
  }

  overrides = provider::deepmerge::mergo(local.overrides_internal, var.overrides)
}
```

