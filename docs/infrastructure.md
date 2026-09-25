# Infrastructure provisioning

`infra-provisioning.yml` is a reusable workflow called by CI after backend tests
and chart validation. It initializes Azure Blob state, validates Terraform, saves
a plan, and applies that exact plan. Only success allows image publishing, then
monitoring and staging deployment. This ordering supports a fresh AKS/ACR/storage
installation. Helm remains responsible for Prometheus and Grafana.

## Configure before the first run

Create a separate Azure storage account and private blob container for state.
They must exist before Terraform initializes and must not be resources managed by
this configuration. Use a new, unused blob key for the fresh environment. Do not
migrate the copied Week06 local state. Never change the key after deployment to
force a fresh apply: that would lose resource ownership.

Repository secret: `AZURE_CREDENTIALS`, containing clientId, clientSecret,
tenantId and subscriptionId. The identity needs permission to create the resource
group and its resources and create role assignments (AKS receives AcrPull), plus
Storage Blob Data Contributor on the pre-created state account. GitHub runners
must be able to access the backend and AKS API. Enable backend blob versioning.

Repository variables:

| Variable | Purpose |
| --- | --- |
| TF_STATE_RESOURCE_GROUP | Existing state storage resource group |
| TF_STATE_STORAGE_ACCOUNT | Existing state storage account |
| TF_STATE_CONTAINER | Existing private container |
| TF_STATE_KEY | Stable blob name, e.g. week08.tfstate |
| AKS_RESOURCE_GROUP | New application resource group |
| AKS_CLUSTER_NAME | New cluster name; also used as DNS prefix |
| ACR_NAME | New globally unique registry name |
| ACR_LOGIN_SERVER | Registry hostname, normally `<ACR_NAME>.azurecr.io` |
| AZURE_STORAGE_ACCOUNT_NAME | New globally unique application storage account |

Use repository-level variables so CI and both deployment environments agree.
Do not shadow them with different environment variables. Existing staging and
production application secrets (database, JWT, admin) are still required.
Deployment retrieves the application storage connection string from Azure and
masks it; the old AZURE_STORAGE_CONNECTION_STRING GitHub secret is no longer used.

The imported Terraform defaults remain: Australia East, three Standard_D2s_v3
nodes, and Kubernetes 1.36.1. Confirm region support/quota and adjust the Terraform
configuration before first deployment if necessary. Names must be unused; existing
resources would require an explicit import, which this fresh-install flow does not do.

## Version control and verification

Commit `.tf` files and `.terraform.lock.hcl`. Local `.terraform/`, state and backups,
variable files, backend configuration files, plans, and crash logs are ignored.
Ignored files stay on disk; do not copy sensitive outputs into documentation.
No state or binary plan is uploaded as a workflow artifact. Authentication uses
Entra credentials and Azure Blob locking; workflow concurrency serializes applies.

After setup, run CI and retain plan/apply logs. Repeat with unchanged configuration
to verify a no-change plan. Confirm ACR image pushes, monitoring installation, and
staging deployment succeed. Local validation is not evidence of a live Azure apply.
