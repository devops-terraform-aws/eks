# AWS EKS Terraform module

Terraform module which creates Amazon EKS (Kubernetes) resources

## Usage

### EKS Auto Mode

```hcl
module "eks" {
  source  = "git::https://github.com/devops-terraform-aws/eks.git?ref=v1.0.0"

  cluster_name    = "example"
  cluster_version = "1.31"

  # Optional
  cluster_endpoint_public_access = true

  # Optional: Adds the current caller identity as an administrator via cluster access entry
  enable_cluster_creator_admin_permissions = true

  cluster_compute_config = {
    enabled    = true
    node_pools = ["general-purpose"]
  }

  vpc_id     = "vpc-1234556abcdef"
  subnet_ids = ["subnet-abcde012", "subnet-bcde012a", "subnet-fghi345a"]

  tags = {
    Environment = "dev"
    Terraform   = "true"
  }
}
```

### EKS Hybrid Nodes

```hcl
locals {
  # RFC 1918 IP ranges supported
  remote_network_cidr = "172.16.0.0/16"
  remote_node_cidr    = cidrsubnet(local.remote_network_cidr, 2, 0)
  remote_pod_cidr     = cidrsubnet(local.remote_network_cidr, 2, 1)
}

# SSM and IAM Roles Anywhere supported - SSM is default
module "eks_hybrid_node_role" {
  source  = "git::https://github.com/devops-terraform-aws/vpc.git?ref=v1.0.0//modules/hybrid-node-role"

  tags = {
    Environment = "dev"
    Terraform   = "true"
  }
}

module "eks" {
  source  = ""git::https://github.com/devops-terraform-aws/vpc.git?ref=v1.0.0""

  cluster_name    = "example"
  cluster_version = "1.31"

  cluster_addons = {
    coredns                = {}
    eks-pod-identity-agent = {}
    kube-proxy             = {}
  }

  # Optional
  cluster_endpoint_public_access = true

  # Optional: Adds the current caller identity as an administrator via cluster access entry
  enable_cluster_creator_admin_permissions = true

  create_node_security_group = false
  cluster_security_group_additional_rules = {
    hybrid-all = {
      cidr_blocks = [local.remote_network_cidr]
      description = "Allow all traffic from remote node/pod network"
      from_port   = 0
      to_port     = 0
      protocol    = "all"
      type        = "ingress"
    }
  }

  # Optional
  cluster_compute_config = {
    enabled    = true
    node_pools = ["system"]
  }

  access_entries = {
    hybrid-node-role = {
      principal_arn = module.eks_hybrid_node_role.arn
      type          = "HYBRID_LINUX"
    }
  }

  vpc_id     = "vpc-1234556abcdef"
  subnet_ids = ["subnet-abcde012", "subnet-bcde012a", "subnet-fghi345a"]

  cluster_remote_network_config = {
    remote_node_networks = {
      cidrs = [local.remote_node_cidr]
    }
    # Required if running webhooks on Hybrid nodes
    remote_pod_networks = {
      cidrs = [local.remote_pod_cidr]
    }
  }

  tags = {
    Environment = "dev"
    Terraform   = "true"
  }
}
```

### EKS Managed Node Group

```hcl
module "eks" {
  source  = "git::https://github.com/devops-terraform-aws/vpc.git?ref=v1.0.0"

  cluster_name    = "my-cluster"
  cluster_version = "1.31"

  bootstrap_self_managed_addons = false
  cluster_addons = {
    coredns                = {}
    eks-pod-identity-agent = {}
    kube-proxy             = {}
    vpc-cni                = {}
  }

  # Optional
  cluster_endpoint_public_access = true

  # Optional: Adds the current caller identity as an administrator via cluster access entry
  enable_cluster_creator_admin_permissions = true

  vpc_id                   = "vpc-1234556abcdef"
  subnet_ids               = ["subnet-abcde012", "subnet-bcde012a", "subnet-fghi345a"]
  control_plane_subnet_ids = ["subnet-xyzde987", "subnet-slkjf456", "subnet-qeiru789"]

  # EKS Managed Node Group(s)
  eks_managed_node_group_defaults = {
    instance_types = ["m6i.large", "m5.large", "m5n.large", "m5zn.large"]
  }

  eks_managed_node_groups = {
    example = {
      # Starting on 1.30, AL2023 is the default AMI type for EKS managed node groups
      ami_type       = "AL2023_x86_64_STANDARD"
      instance_types = ["m5.xlarge"]

      min_size     = 2
      max_size     = 10
      desired_size = 2
    }
  }

  tags = {
    Environment = "dev"
    Terraform   = "true"
  }
}
```

### Cluster Access Entry

When enabling `authentication_mode = "API_AND_CONFIG_MAP"`, EKS will automatically create an access entry for the IAM role(s) used by managed node group(s) and Fargate profile(s). There are no additional actions required by users. For self-managed node groups and the Karpenter sub-module, this project automatically adds the access entry on behalf of users so there are no additional actions required by users.

On clusters that were created prior to cluster access management (CAM) support, there will be an existing access entry for the cluster creator. This was previously not visible when using `aws-auth` ConfigMap, but will become visible when access entry is enabled.

```hcl
module "eks" {
  source  = "git::https://github.com/devops-terraform-aws/vpc.git?ref=v1.0.0"

  # Truncated for brevity ...

  access_entries = {
    # One access entry with a policy associated
    example = {
      principal_arn = "arn:aws:iam::123456789012:role/something"

      policy_associations = {
        example = {
          policy_arn = "arn:aws:eks::aws:cluster-access-policy/AmazonEKSViewPolicy"
          access_scope = {
            namespaces = ["default"]
            type       = "namespace"
          }
        }
      }
    }
  }
}
```

### Bootstrap Cluster Creator Admin Permissions

Setting the `bootstrap_cluster_creator_admin_permissions` is a one time operation when the cluster is created; it cannot be modified later through the EKS API. In this project we are hardcoding this to `false`. If users wish to achieve the same functionality, we will do that through an access entry which can be enabled or disabled at any time of their choosing using the variable `enable_cluster_creator_admin_permissions`

### Enabling EFA Support

When enabling EFA support via `enable_efa_support = true`, there are two locations this can be specified - one at the cluster level, and one at the node group level. Enabling at the cluster level will add the EFA required ingress/egress rules to the shared security group created for the node group(s). Enabling at the node group level will do the following (per node group where enabled):

1. All EFA interfaces supported by the instance will be exposed on the launch template used by the node group
2. A placement group with `strategy = "clustered"` per EFA requirements is created and passed to the launch template used by the node group
3. Data sources will reverse lookup the availability zones that support the instance type selected based on the subnets provided, ensuring that only the associated subnets are passed to the launch template and therefore used by the placement group. This avoids the placement group being created in an availability zone that does not support the instance type selected.

> [!TIP]
> Use the [aws-efa-k8s-device-plugin](https://github.com/aws/eks-charts/tree/master/stable/aws-efa-k8s-device-plugin) Helm chart to expose the EFA interfaces on the nodes as an extended resource, and allow pods to request the interfaces be mounted to their containers.
>
> The EKS AL2 GPU AMI comes with the necessary EFA components pre-installed - you just need to expose the EFA devices on the nodes via their launch templates, ensure the required EFA security group rules are in place, and deploy the `aws-efa-k8s-device-plugin` in order to start utilizing EFA within your cluster. Your application container will need to have the necessary libraries and runtime in order to utilize communication over the EFA interfaces (NCCL, aws-ofi-nccl, hwloc, libfabric, aws-neuornx-collectives, CUDA, etc.).

If you disable the creation and use of the managed node group custom launch template (`create_launch_template = false` and/or `use_custom_launch_template = false`), this will interfere with the EFA functionality provided. In addition, if you do not supply an `instance_type` for self-managed node group(s), or `instance_types` for the managed node group(s), this will also interfere with the functionality. In order to support the EFA functionality provided by `enable_efa_support = true`, you must utilize the custom launch template created/provided by this module, and supply an `instance_type`/`instance_types` for the respective node group.

The logic behind supporting EFA uses a data source to lookup the instance type to retrieve the number of interfaces that the instance supports in order to enumerate and expose those interfaces on the launch template created. For managed node groups where a list of instance types are supported, the first instance type in the list is used to calculate the number of EFA interfaces supported. Mixing instance types with varying number of interfaces is not recommended for EFA (or in some cases, mixing instance types is not supported - i.e. - p5.48xlarge and p4d.24xlarge). In addition to exposing the EFA interfaces and updating the security group rules, a placement group is created per the EFA requirements and only the availability zones that support the instance type selected are used in the subnets provided to the node group.

In order to enable EFA support, you will have to specify `enable_efa_support = true` on both the cluster and each node group that you wish to enable EFA support for:

```hcl
module "eks" {
  source  = "git::https://github.com/devops-terraform-aws/vpc.git?ref=v1.0.0"

  # Truncated for brevity ...

  # Adds the EFA required security group rules to the shared
  # security group created for the node group(s)
  enable_efa_support = true

  eks_managed_node_groups = {
    example = {
      # The EKS AL2023 NVIDIA AMI provides all of the necessary components
      # for accelerated workloads w/ EFA
      ami_type       = "AL2023_x86_64_NVIDIA"
      instance_types = ["p5.48xlarge"]

      # Exposes all EFA interfaces on the launch template created by the node group(s)
      # This would expose all 32 EFA interfaces for the p5.48xlarge instance type
      enable_efa_support = true

      # Mount instance store volumes in RAID-0 for kubelet and containerd
      # https://github.com/awslabs/amazon-eks-ami/blob/master/doc/USER_GUIDE.md#raid-0-for-kubelet-and-containerd-raid0
      cloudinit_pre_nodeadm = [
        {
          content_type = "application/node.eks.aws"
          content      = <<-EOT
            ---
            apiVersion: node.eks.aws/v1alpha1
            kind: NodeConfig
            spec:
              instance:
                localStorage:
                  strategy: RAID0
          EOT
        }
      ]

      # EFA should only be enabled when connecting 2 or more nodes
      # Do not use EFA on a single node workload
      min_size     = 2
      max_size     = 10
      desired_size = 2
    }
  }
}
```