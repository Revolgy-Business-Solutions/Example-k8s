# Multi-tenant Kubernetes Infrustructure

## Overview

This repository implements the GCP-best-practice infrastructure solution for multi-tenant Kubernetes cluster. A complete description of the project organization, networking and Kubernetes configuration can be found in the following documentation:
https://cloud.google.com/kubernetes-engine/docs/best-practices/enterprise-multitenancy

The diagram of the multi-tenant infrastructure can be seen on following image:

![alt text](https://cloud.google.com/kubernetes-engine/images/enterprise-project-architecture.svg)

## Terraform Infrastructure description

The Terraform module "tenant_k8s_infrastructure" represents a single environment for the multi-tenant GCP infrastructure. The module consists of the following resources

- Networking
- - Host VPC network for tenants subnets
- - Host VPC network for Kubernetes cluster
- - VPC Peering and Shared VPC configuration between Host VPCs and service projects
- Kubernetes cluster
- - Cluster configuration according to best-practice document
- - Tenants separation configuration

## Tenants creation

Tenants are created via Terraform module `modules/tenant`. Parameters for specific tenant are defined in `variables.tf` file. Example usage and input parameters description are below.

For each tenant is created a dedicated subnet on the host Shared VPC network. Furthermore a Kubernetes namespace on multi-tenant cluster is created and assigned with network policies, quotas and RBAC rules defined by input parameters.

RBAC rules are binded on Workspace/Cloud Indentity users and groups as well as GCP Service Account. Those can be separately listed in tenants variable.

All the resource quota list and description can be found at: https://kubernetes.io/docs/concepts/policy/resource-quotas/. The whole resource name, as stated in the documentation, has to be used for specifying a quota, Furthermore if the name contains a dot, it needs to be quoted, e.g. :

- "limits.cpu"
- "requests.cpu"
- pods

For each environment there is a Terraform Kubernetes provider configuration based on outputs from multi-tenant module. This provider is then passed into tenant module in order to deploy Kubernetes resources to a specific environment. This allows to reuse the tenant module for different environment without duplicating resources in modules

## Example usage

A single environment multi-tenant infrastructure can be deployed by following module configuration:

### multi-tenant environment module

```
source = "./modules/multi_tenant_infrastructure"

region = "europe-west1"

tenant_host_vpc_prj_id  = "<TENANT_SHARED_VPC_HOST_PROJECT_ID>"
cluster_host_vpc_prj_id = "<CLUSTER_SHARED_VPC_HOST_PROJECT_ID>"
cluster_service_prj_id  = "<CLUSTER_SHARED_VPC_SERVICE_PROJECT_ID>"

tenant_host_vpc_name = "dev-tenant-vpc"

cluster_host_vpc_name      = "dev-cluster-vpc"
cluster_host_sub_cidr      = "10.6.128.0/17"
cluster_host_sub_pods_cidr = "10.7.0.0/16"
cluster_host_sub_svcs_cidr = "10.8.0.0/16"

k8s_cluster_name                   = "dev-k8s-cluster"
k8s_cluster_zones                  = ["europe-west1-b", "europe-west1-c", "europe-west1-d"]
k8s_cluster_master_ipv4_cidr_block = "172.16.0.0/28"
k8s_cluster_master_authorized_networks = [
  { display_name : "access_ip", cidr_block : "<CIDR_BLOCK>" },
]

k8s_node_pools = [
  {
    name               = "default-node-pool"
    machine_type       = "e2-medium"
    node_locations     = "europe-west1-b,europe-west1-c,europe-west1-d"
    min_count          = 1
    max_count          = 100
    local_ssd_count    = 0
    disk_size_gb       = 100
    disk_type          = "pd-standard"
    image_type         = "COS"
    auto_repair        = true
    auto_upgrade       = true
    initial_node_count = 1
  },
]

k8s_version = "1.19.10-gke.1700"
nginx_ingress_controller_helm_chart_version = "3.33.0"
```

### environment specific Terraform Kubernetes provider

From module multi-tenant module output we can define a environment-specific Kubernetes provider

```
provider "kubernetes" {
  host                   = "https://${module.dev_k8s_tenant_infra.gke_endpoint}"
  token                  = data.google_client_config.default.access_token
  cluster_ca_certificate = base64decode(module.dev_k8s_tenant_infra.gke_ca_certificate)
  alias                  = "dev-cluster"
}
```

## Example tenant input variable

```
default = {
  tenant-a = {
    namespace     = "tenant-a"
    subnet_region = "europe-west1"
    dev = {
      host_vpc_subnet_cidr = "10.6.0.0/24"
      project_id           = "<TENANT_SERVICE_DEV_PROJECT_ID>"
      quotas               = {           
        pods = 10
      }
      admin_groups         = []
      admin_users          = []
      admin_sa             = []
      developer_groups     = []
      developer_users      = [
        "test.user@gmail.com",
      ]
      developer_sa         = []
    }
    stg = {
      host_vpc_subnet_cidr = "10.3.0.0/24"
      project_id           = "<TENANT_SERVICE_STG_PROJECT_ID>"
      quotas               = {           
        pods = 10
      }
      admin_groups         = []
      admin_users          = []
      admin_sa             = []
      developer_groups     = []
      developer_users      = []
      developer_sa         = []
    }
    prd = {
      host_vpc_subnet_cidr = "10.0.0.0/24"
      project_id           = "<TENANT_SERVICE_PRD_PROJECT_ID>"
      quotas               = {           
        pods = 10
      }
      admin_groups         = []
      admin_users          = []
      admin_sa             = []
      developer_groups     = []
      developer_users      = []
      developer_sa         = []
    }
  },
  tenant-b = {
    namespace     = "tenant-b"
    subnet_region = "europe-west1"
    dev = {
      host_vpc_subnet_cidr = "10.6.1.0/24"
      project_id           = "<TENANT_SERVICE_PRD_PROJECT_ID>"
      quotas               = {           
        pods              = 50
        services          = 20
        "limits.memory"   = "60Gi"
        "requests.cpu"    = 1000
        "requests.memory" = "40Gi"
      }
      admin_groups         = []
      admin_users          = []
      admin_sa             = []
      developer_groups     = []
      developer_users      = []
      developer_sa         = []
    }
    stg = {
      host_vpc_subnet_cidr = ""
      project_id           = ""
      quotas               = {}
      admin_groups         = []
      admin_users          = []
      admin_sa             = []
      developer_groups     = []
      developer_users      = []
      developer_sa         = []
    }
    prd = {
      host_vpc_subnet_cidr = "10.0.1.0/24"
      project_id           = "<TENANT_SERVICE_PRD_PROJECT_ID>"
      quotas               = {           
        pods = 10
      }
      admin_groups         = []
      admin_users          = []
      admin_sa             = []
      developer_groups     = []
      developer_users      = []
      developer_sa         = []
    }
  }
}
```

## Inputs

### multi_tenant_infrastructure module inputs


| Name | Description | Type | Default | Required |
|------|-------------|------|---------|:--------:|
| <a name="input_cluster_host_sub_cidr"></a> [cluster\_host\_sub\_cidr](#input\_cluster\_host\_sub\_cidr) | IP range of the host VPC subnet for the Kubernetes cluster | `string` | n/a | yes |
| <a name="input_cluster_host_sub_pods_cidr"></a> [cluster\_host\_sub\_pods\_cidr](#input\_cluster\_host\_sub\_pods\_cidr) | IP range of the host VPC secondary subnet for Kubernetes pods | `string` | n/a | yes |
| <a name="input_cluster_host_sub_svcs_cidr"></a> [cluster\_host\_sub\_svcs\_cidr](#input\_cluster\_host\_sub\_svcs\_cidr) | IP range of the host VPC secondary subnet for Kubernetes svsc | `string` | n/a | yes |
| <a name="input_cluster_host_vpc_name"></a> [cluster\_host\_vpc\_name](#input\_cluster\_host\_vpc\_name) | Name of the host VPC network for Kubernetes cluster | `string` | n/a | yes |
| <a name="input_cluster_host_vpc_prj_id"></a> [cluster\_host\_vpc\_prj\_id](#input\_cluster\_host\_vpc\_prj\_id) | GCP Project ID of the project hosting Shared VPC for Kubernetes cluster | `string` | n/a | yes |
| <a name="input_cluster_service_prj_id"></a> [cluster\_service\_prj\_id](#input\_cluster\_service\_prj\_id) | GCP Project ID of the project hosting Kueberes Cluster | `string` | n/a | yes |
| <a name="input_k8s_cluster_master_authorized_networks"></a> [k8s\_cluster\_master\_authorized\_networks](#input\_k8s\_cluster\_master\_authorized\_networks) | List of IP ranges to be allowed to access Kubernetes control plane | `list(any)` | n/a | yes |
| <a name="input_k8s_cluster_master_ipv4_cidr_block"></a> [k8s\_cluster\_master\_ipv4\_cidr\_block](#input\_k8s\_cluster\_master\_ipv4\_cidr\_block) | IP range for private Kubernetes cluster control plane | `string` | n/a | yes |
| <a name="input_k8s_cluster_name"></a> [k8s\_cluster\_name](#input\_k8s\_cluster\_name) | Name of the Kubernetes cluster | `string` | n/a | yes |
| <a name="input_k8s_cluster_zones"></a> [k8s\_cluster\_zones](#input\_k8s\_cluster\_zones) | GCP zone for Kubernetes nodes | `list(any)` | n/a | yes |
| <a name="input_k8s_node_pools"></a> [k8s\_node\_pools](#input\_k8s\_node\_pools) | Configuration of Kubernetes node pools | `list(any)` | n/a | yes |
| <a name="input_k8s_version"></a> [k8s\_version](#input\_k8s\_version) | The Kubernetes version of the masters. | `string` | `"latest"` | no |
| <a name="input_nginx_ingress_controller_helm_chart_version"></a> [nginx\_ingress\_controller\_helm\_chart\_version](#input\_nginx\_ingress\_controller\_helm\_chart\_version) | Version of the nginx-ingress controller helm chart to be installed into the cluster | `string` | n/a | yes |
| <a name="input_region"></a> [region](#input\_region) | GCP region for the VPC network to be configured in | `string` | "europe-west1" | yes |
| <a name="input_tenant_host_vpc_name"></a> [tenant\_host\_vpc\_name](#input\_tenant\_host\_vpc\_name) | Name of the host VPC network for tenants subnets | `string` | n/a | yes |
| <a name="input_tenant_host_vpc_prj_id"></a> [tenant\_host\_vpc\_prj\_id](#input\_tenant\_host\_vpc\_prj\_id) | GCP Project ID of the project hosting Shared VPC for tenants subnets | `string` | n/a | yes |

### Tenant module inputs

| Name | Description | Type | Default | Required |
|------|-------------|------|---------|:--------:|
| <a name="input_admin_groups"></a> [admin\_groups](#input\_admin\_groups) | List of workspace/Cloud Identity group to be assigned with admin permissions on tenant namespace | `list(any)` | n/a | yes |
| <a name="input_admin_sa"></a> [admin\_sa](#input\_admin\_sa) | List of workspace/Cloud Identity users to be assigned with admin permissions on tenant namespace | `list(any)` | n/a | yes |
| <a name="input_admin_users"></a> [admin\_users](#input\_admin\_users) | List of workspace/Cloud Identity users to be assigned with admin permissions on tenant namespace | `list(any)` | n/a | yes |
| <a name="input_developer_groups"></a> [developer\_groups](#input\_developer\_groups) | List of workspace/Cloud Identity group to be assigned with edit permissions on tenant namespace | `list(any)` | n/a | yes |
| <a name="input_developer_sa"></a> [developer\_sa](#input\_developer\_sa) | List of workspace/Cloud Identity users to be assigned with edit permissions on tenant namespace | `list(any)` | n/a | yes |
| <a name="input_developer_users"></a> [developer\_users](#input\_developer\_users) | List of workspace/Cloud Identity users to be assigned with edit permissions on tenant namespace | `list(any)` | n/a | yes |
| <a name="input_k8s_service_project_id"></a> [k8s\_service\_project\_id](#input\_k8s\_service\_project\_id) | Project which hosts Kubernetes cluster | `string` | n/a | yes |
| <a name="input_name"></a> [name](#input\_name) | Tenant name | `string` | n/a | yes |
| <a name="input_namespace"></a> [namespace](#input\_namespace) | Kubernetest namespace to be created for tenant | `string` | n/a | yes |
| <a name="input_quotas"></a> [quotas](#input\_quotas) | Resource quotas for namespace | `map(any)` | `null` | no |
| <a name="input_subnet_region"></a> [subnet\_region](#input\_subnet\_region) | GCP region that the subnet will be deployed in | `string` | n/a | yes |
| <a name="input_tenant_host_sub_cidr"></a> [tenant\_host\_sub\_cidr](#input\_tenant\_host\_sub\_cidr) | IP range for the subnet created for tenant in host VPC | `string` | n/a | yes |
| <a name="input_tenant_host_vpc"></a> [tenant\_host\_vpc](#input\_tenant\_host\_vpc) | Self link of the host Shared VPC for tenants subnets | `any` | n/a | yes |
| <a name="input_tenant_project_id"></a> [tenant\_project\_id](#input\_tenant\_project\_id) | GCP Project ID of the tenant project | `string` | n/a | yes |
