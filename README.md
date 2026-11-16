# gha-workflows
Reusable GHA workflows

## Docker Bake workflow

The reusable `docker-bake.yaml` workflow builds and publishes Bake targets,
adds OCI annotations, scans each published platform with Trivy, and publishes
unsigned in-toto scan attestations as OCI referrers. It verifies the platform
and index attestations before applying the findings policy.

The `fail-on-scan-findings` input defaults to `true`; set it to `false` to
report findings without failing the workflow. The `trivyignore-file` input
defaults to `.trivyignore` in the calling repository. If the configured path
does not exist, the scan runs without an ignore file.
