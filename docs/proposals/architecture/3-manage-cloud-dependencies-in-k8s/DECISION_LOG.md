# Decision log

## Controller comparison

Solutions must meet this acceptance criteria:
1. Multicloud. Rod project is meant to support multiple clouds so we must have an interface to create buckets for any possible cloud
2. Maintained. Controllers we choose must be well maintained or support pull requests from us
3. Simple. Complex solutions become hard to manage very fast

## Existing bucket operators

* https://github.com/jjaferson/bucket-operator
  * Supports only buckets created by SeaweedFS
  * Unmaintained (last commit 2years ago)
**Conclusion**: Not suitable

* https://github.com/DevangRadadiya/k8s-s3-bucket-operator/tree/main
  * COSI driver for MinIO and AWS
  * Maintained and with clear road map
  * Codebase is kind of sus 
**Conclution**: Not suitable because its not planning to support other clouds

* https://github.com/InseeFrLab/s3-operator
  * Support only MinIO provider
  * Well maintained
  * Has helm chart 
