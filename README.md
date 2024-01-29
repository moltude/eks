
# ITHAKA Cloud take home test

```text
Scott Williams
January 2024
```

## Problem

Design (not build) an automated process that, given a cluster name, performs the following
tasks:

1. Creates a new EKS cluster.
2. Adds self-managed nodes to the cluster.
3. Implements RBAC to grant administrative rights to users with a specific IAM role.
4. Prepares the cluster to deploy an application capable of accepting traffic.

Additional Guidance

- Explicitly state any assumptions you make during the design process, including insights
into why you chose specific tools or approaches for your solution.

- If you encounter ambiguities or uncertainties in the scenario, feel free to document them along with your proposed design.

## Solution

### Documentation and external resources

Some of the sources I consulted during the process.

   1. [Terraform EKS module](https://github.com/terraform-aws-modules/terraform-aws-eks)
   2. [HashiCorp EKS](https://developer.hashicorp.com/terraform/tutorials/kubernetes/eks)
   3. [AWS Self-managed EKS nodes](https://docs.aws.amazon.com/eks/latest/userguide/worker.html)
   4. [AWS RBAC](https://docs.aws.amazon.com/eks/latest/userguide/add-user-role.html)
   5. [Kubernetes RBAC configuration](https://kubernetes.io/docs/reference/access-authn-authz/rbac/)
   6. [Terraform kubernetes_config_map](https://registry.terraform.io/providers/hashicorp/kubernetes/latest/docs/resources/config_map)
   7. Various Medium and Reddit posts

**Why use Terraform?**

My overall approach to this problem will be to use Terraform for the creation of the cluser, adding self-managed nodes, implementing RBAC with specific IAM roles and readying to cluster to accept traffic. At a high level this process will invovle implementing several supported external Terraform modules inside a local developed terraform module, which I will refer to as `ithaka_eks`.

There are other approaches which utilize a combination of Terraform, `kubectl`, `ekscli` and configuration management tools like Ansible. However, they would require user intervention at cetain steps and is not an end-to-end automated process which is a requirement of this test.

**Why Terraform?**

 1. All infrastructure should be under some kind of IaC control
 2. IaC allows us to apply infrastrutre changes in a consistent way and manage state in a distrubuted enviornment.
 3. We can leverage existing infrastructure in our build process (ex. VPCs, subnets, security groups).
 4. This approach allows for deploying the same cluster across multiple enviornments (test, staging or production)

### Step 1: Create a new EKS cluster

1. Import and implement the [Terraform EKS module](https://github.com/terraform-aws-modules/terraform-aws-eks) as a component `ithaka_eks`.

2. The cluster name would be a required input parameter. This could be achieved with a variables file, environmental variables or command line input.

    `terraform plan -var="cluster_name=MY_TEST_CLUSTER"`

    *For discussion* This value could also be used for the required `namespace` value. For a single app in a cluster it may be appropriate.

3. *For discussion* - There will be other required inputs for things like AZ / region and other vars necessary for spinning up this cluster but defining what and where they are loaded from is probably out of scope for this submission.

    There are many other possible inputs which may need to  be defined other than cluster name (min/max pods, instance types, allocation, IAM role ARN)

### Step 2: Adds self-managed nodes to the cluster

The standard [Terraform EKS module](https://github.com/terraform-aws-modules/terraform-aws-eks) supports using self-managed node groups in the EKS clusters.

In the implementation of the EKS module we can specify the node group type to be used by defining the `self_managed_node_groups` [inputs](https://registry.terraform.io/modules/terraform-aws-modules/eks/aws/latest/examples/self_managed_node_group) in the cluster resource as well as it's required and optional parameters.

Where appropriate these values should be pulled from variable files and/or have default values defined in the `.tfvars`:

- eks cluster name
- ec2 instance type
- min/max nodes
- subnets
- desired capacity

### Step 3: Implements RBAC to grant administrative rights to users with a specific IAM role

#### Assumptions

- The IAM role exists as a managed resouce and can be imported into the project so we can use the `IAM role ARN`.

- The IAM role has sufficent privileges to autheticate and access the cluster.

- The Kuberenetes 'administriave rights' (as defined in the activity requirement) is `system:masters` for the cluser. This is overly permissive and should be avoided but for the sake of this exercise I will use it.

To grant administrative permissions for an IAM role which did not create the cluster we will use the `kubernetes_config_map` resource and map the ARN of the IAM role to the Kubernetes group.

```text
resource "kubernetes_config_map" "auth" {
  metadata {
    name      = "auth"
    namespace = "kube-system"
  }

  data = {
    mapRoles = <<EOF
        - rolearn: arn:aws:iam::XXX:role/<IAM_ROLE>
        username: <KUBERNETES USERNAME>
        groups:
            - system:masters
    EOF
  }
}
```

### Step 4: Prepares the cluster to deploy an application capable of accepting traffic

This is often done with `kubectl` or some other deployment system like Ansible. For this exercise since I am trying to encapsulate all the operations in a single command I will utilize additional EKS resources to manage the service and deployment.

In Terraform we need to define the following resources.

- Kubernetes namespace (`kubernetes_namespace`)

    Required for `kubernetes_deployment` resouce. Used for grouping resource within a cluster, the value of the namespace could be reused from the cluster name.

- Kubernetes deployment (`kubernetes_deployment`)

  Inside this resouce we define where the container/application to deploy is located (ex. EMR)

- Kubernetes service (`kubernetes_service`)

  This will contain the load balancer and port mappings between the incoming traffic port and target port on the container. It also allows us to access all the pods via a single IP address.

## Conclusion

There are several areas which the plan does not fully address because they felt out of scope or were ambigious.

1. Overall project strucutre (filesystem)
   1. Seperation of each of the steps and resources into specific Terraform files, ITHAKA specific modules.
   2. Deploying across multiple enviornments (staging, test, prod).
   3. Inputs and variables beyond the `cluster_name`
2. Existing AWS infrastructure resources
   1. VPCs
   2. Security groups
   3. IG
   4. Subnets
