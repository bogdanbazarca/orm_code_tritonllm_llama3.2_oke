# orm_stack_oke_tritonllm_llama3.2_models

## Getting started

This stack deploys an OKE cluster with two nodepools:
- one nodepool with flexible shapes
- one nodepool with GPU shapes

The included helm module facilitates the deployment of helm charts to the OKE cluster.

**Note:** For helm deployments it's necessary to create bastion and operator host (with the associated policy for the operator to manage the clsuter), **or** configure a cluster with public API endpoint.

In case the bastion and operator hosts are not created, is a prerequisite to have the following tools already installed and configured:
- bash
- helm
- jq
- kubectl
- oci-cli

**Note:** All the tools are already available in the ORM runner.

## OKE Cluster

The OKE cluster is called from the file `main.tf`.

This is a fork of an older version of the [OKE Terraform module](https://github.com/oracle-terraform-modules/terraform-oci-oke) with support for terraform 1.2.9 (the latest version available in ORM).

For more informations about the OKE module, please read the documentation available [here](https://oracle-terraform-modules.github.io/terraform-oci-oke/).

The references to GPUs in the files `main.tf` (4-11), `datasources.tf` (22-32) provide support for automatic discovery of the ADs supporting the selected GPU shape. (specific GPU shapes are not available in all the ADs  - for multi-AD OCI regions)


## Helm Deployments

If you want to create a new helm deployment, create one more module resource referencing the `helm-module` in the file `helm-deployment.tf`. You can use the existing nginx helm deployment as an example (this is commented)

```
module "nginx" {
  count  = var.deploy_nginx ? 1 : 0
  source = "./helm-module"

  ## Connectivity details required for remote-exec provisioners. 
  ## Are used only when the bastion and operator hosts are created.
  bastion_host    = module.oke.bastion_public_ip
  bastion_user    = var.bastion_user
  operator_host   = module.oke.operator_private_ip
  operator_user   = var.bastion_user
  ssh_private_key = tls_private_key.stack_key.private_key_openssh

  ## Local variables used to determine how the helm deployment will be executed (from operator via bastion OR from local/ORM runner).
  deploy_from_operator = local.deploy_from_operator
  deploy_from_local    = local.deploy_from_local

  ## Helm Charts parameters
  deployment_name           = "ingress-nginx"  # The name of the helm deployment
  helm_chart_name           = "ingress-nginx"  # The name of the helm chart to be used (required only when usnig helm_repository_url)
  namespace                 = "nginx" # The namespace to use for the deployment
  helm_repository_url       = "https://kubernetes.github.io/ingress-nginx" # Fetch helm chart from this Helm repository URL
  helm_chart_path           = "" # Can be used for deployments of helm charts available locally, as remote http tgz file or OCI repository. If present, and not empty, will override `helm_repository_url` 
  operator_helm_values_path = local.operator_helm_values_path # Local variable used to determine where the required files are stored on operator
  pre_deployment_commands   = [] # A list of bash commands that will be executed before the helm deployment
  post_deployment_commands  = [] # A list of bash commands that will be executed after the helm deployment.

  ## Helm values override file generated using Terraform template
  helm_template_values_override = templatefile(
    "${path.root}/helm-values-templates/nginx-values.yaml.tpl",
    {
      min_bw        = 100,
      max_bw        = 100,
      pub_lb_nsg_id = module.oke.pub_lb_nsg_id
      state_id      = local.state_id
    }
  )

  ## Helm values override file provided by the user. base64decode() is used for compatibility with ORM variables of type `file` (Optional).
  helm_user_values_override = try(base64decode(var.nginx_user_values_override), var.nginx_user_values_override)
  
  ## Kubeconfig for the OKE cluster
  kube_config = one(data.oci_containerengine_cluster_kube_config.kube_config.*.content)

  depends_on  = [module.oke]
}
```

## What is being deployed

This code deployes [**NVIDIA_Triton_LLM_with_llama2-7b**](https://github.com/triton-inference-server/tutorials/blob/main/Popular_Models_Guide/Llama2/trtllm_guide.md) on an OKE GPU A10 shape worker node

- The helm deployment uses a local helm chart (in folder oci)
- The code is copying the helm chart folder to operator and deploy it from there
- Under _helm-values-templates_ the file _value_triton_ override the helm chart values
- The triton server runs on ports 8000, 8001 and 8002 and exposes them on worker node which then uses a Load balancer to get external access
- To run triton server only on worker nodes with GPU it uses a _resource_type_ (resource key: nvidia.com/gpu) which is set by the code that deploys the OKE ()
- The triton inference will be available to query via external Load Balancer IP in this manner: ```curl -X POST http:/<external_LB_IP>:8000/v2/models/tensorrt_llm_bls/generate -H "Content-Type: application/json" -d '{"text_input": "Meta is", "max_tokens": 53, "bad_words": "", "stop_words": ""}'```


## How to deploy?

1. Deploy via ORM
- Create a new stack
- Upload the TF configuration files
- Configure the variables
- Apply

2. Local deployment

- Create a file called `terraform.auto.tfvars` with the required values.

```
# ORM injected values

region            = "eu-frankfurt-1"
tenancy_ocid      = "ocid1.tenancy.oc1..aaaaaaaaiyavtwbz4kyu7g7b6wglllccbflmjx2lzk5nwpbme44mv54xu7dq"
compartment_ocid  = "<compartment_id>"
current_user_ocid = "test"

# OKE Terraform module values
create_iam_resources     = false
create_iam_tag_namespace = false
ssh_public_key           = "<ssh-public-key>"

cluster_name                = "oke"
vcn_name                    = "oke-vcn"
simple_np_flex_shape        = { "instanceShape" = "VM.Standard.E4.Flex", "ocpus" = 2, "memory" = 16 }
compartment_id              = "<compartment_id>"
create_operator_and_bastion = false
control_plane_is_public     = true
deploy_nginx                = true
nginx_user_values_override  = <<-EOT
controller:
  metrics:
    enabled: true
EOT

```

- Execute the commands

```
terraform init
terraform plan
terraform apply
```

## Known Issues

- On change of the helm chart values (the values generated by Terraform using the templates or the values provided by the user), the existing helm deployment is removed (`helm uninstall`) and a new one is created (this behavior is caused by the on-destroy provisioner). If this behavior is not desired, and you want to update the Helm deployment in place, comment out the on-destroy provisoners in the `helm-module/helm-deployment.tf` files.

- Commenting out `on-destroy` provisioner may cause the `terraform destroy` to fail as the helm deployments are not removed and there might be LBs using the created VCN/subnet.