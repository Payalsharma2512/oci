# Deploy Oracle Cloud Infrastructure Kubernetes Engine (OKE) using Advanced Terraform Modules

## Introduction 

This is the second tutorial in our **Oracle Cloud Infrastructure Kubernetes Engine (OKE) automation series**, building on the foundational concepts from the first one, *[Create Oracle Cloud Infrastructure Kubernetes Engine Cluster using Terraform](https://docs.oracle.com/en/learn/oke-clstr-trfm/)*. In this article, we take OKE deployments to the next level by introducing **advanced automation** and **modular design** for provisioning scalable, resilient, and highly customizable environments on Oracle Cloud Infrastructure (OCI).  

Designed for Oracle engineers and customers, this guide empowers you to transform a basic Terraform configuration into a **structured, reusable module** that can be tailored to any deployment size. By leveraging [Infrastructure as Code (IaC)](https://github.com/resources/articles/devops/what-is-infrastructure-as-code) principles with [Terraform (community version)](https://developer.hashicorp.com/terraform/docs), combined with [Jenkins](https://www.jenkins.io/) and Oracle [Command Line Interface (CLI)](https://docs.oracle.com/en-us/iaas/Content/API/Concepts/cliconcepts.htm), we provide a `single-click` provisioning solution for OKE clusters, their configurations, and automated testing.  

This tutorial focuses on four key areas:  
1. **Modular Design and Automation**: Build reusable modules for core components such as VCN, OKE Clusters, and Bastion Hosts, with versioning for compatibility to simplify management and to promote code reuse.  
2. **Dynamic Configuration**: Manage environment-specific settings (dev, test, prod) from a single codebase for consistent and repeatable deployments.  
3. **Advanced Node Pools**: Optimize resource usage and costs with specialized shapes, compatible OKE images, and labels for intelligent workload placement.  
4. **Seamless CI/CD Integration**: Automate OKE provisioning and updates using pipelines that integrate Terraform, Jenkins, and OCI CLI for efficient, `one-click` deployments.  

The architecture outlined in this guide uses a **tiered network topology** to create a secure and scalable foundation. It separates control plane, worker nodes, pods, and load balancer traffic, ensuring flexibility for enterprise workloads of any scale. We’ll demonstrate how to fully automate the provisioning of such environment, including networking, clusters, and node pools, while providing the tools to customize and extend the solution to your needs.  

This tutorial assumes an understanding of Kubernetes, networking, and IaC principles, as well as working knowledge of tools such as Terraform, OCI CLI, and Jenkins. By the end, you’ll have the skills to deploy a **resilient, high-performing, and scalable** OKE environment on OCI, with the flexibility to adapt it to your specific requirements.

## Oracle Cloud Infrastructure Kubernetes Engine (OKE) Overview

Even though Oracle Cloud Infrastructure Kubernetes Engine (OKE) is a managed service, successfully running mission-critical workloads requires a deliberate and well-structured architectural approach. Transitioning from a basic setup to a scalable, enterprise-grade environment demands careful planning and the use of Infrastructure as Code (IaC) tools such as Terraform.  

Modularization and automation are critical for OKE to ensure scalability, security, and operational efficiency. By leveraging structured modules and automation tools, enterprises can deploy and manage mission-critical workloads with consistency, reduce manual errors, and accelerate time-to-market.  

The following diagram illustrates the OKE architecture, highlighting its tiered network topology, private API endpoint, and secure access controls:  

![OKE Architecture](./images/1-OKEArchitecture.jpg "OKE Architecture")

For a detailed example of the architecture, refer to the [Oracle documentation](https://docs.oracle.com/en-us/iaas/Content/ContEng/Concepts/contengnetworkconfigexample.htm#example-flannel-cni-privatek8sapi_privateworkers_publiclb).

**Architectural Considerations**  
Building a scalable OKE environment involves addressing at least the following key technical challenges:  
- **Security**: Implement a robust security posture using workload identity, network security groups, private API endpoints, and a Web Application Firewall (WAF) for secure isolation and strict traffic control.  
- **Scalability**: Optimize for high availability by distributing nodes across availability domains then fault domains and using the Kubernetes Cluster Autoscaler for dynamic scaling.  
- **Monitoring and Observability**: Integrate with OCI Logging and OCI Logging Analytics for comprehensive monitoring of cluster and pod-level behavior.  

**Key Design Elements**  
This modular solution is built on a tiered network architecture, providing a secure and scalable foundation for enterprise workloads on OKE:  
- **Network Segmentation**: Separate the environment with dedicated public and private subnets for the Kubernetes API, worker nodes, pods, and load balancers.  
- **Controlled Access**: Use a private API endpoint for secure control plane access and bastion hosts for managed SSH access.  
- **Full Automation**: Automate the provisioning of the entire environment, including networking, clusters, and node pools, using Terraform, OCI CLI, Bash, and Jenkins for efficient, `single-click` deployments.  
- **Advanced Operations**: Implement persistent volumes and automated node cycling for zero-downtime upgrades.

## Building a Modular Terraform Automation for OKE  

Transforming a `flat` Terraform configuration into a `structured`, modular design is essential for creating **repeatable and scalable environments**. This approach ensures better organization, code reusability, and maintainability at enterprise scale, with versioning for compatibility and collaboration across teams. 

### Transformation Process: Flat to Structured Module  

Starting from the flat module described in the first [tutorial](https://docs.oracle.com/en/learn/oke-clstr-trfm/), we refactor the code into a modular design by:  
1. **Restructuring the Directory**: Creating child modules (`vcn`, `oke`, `bastion`) and organizing resources into their respective folders.  
2. **Applying Key Principles**:  
   - **Structure and Logic**: Encapsulate resources in self-contained directories (for example `modules/vcn`, `modules/oke`, `modules/bastion`) and split monolithic code into `main.tf`, `variables.tf`, and `outputs.tf` for readability and maintainability.  
   - **Inputs/Outputs and Versioning**: Define inputs in `variables.tf`, expose outputs in `outputs.tf`, and use Terraform module versioning (`version` constraint in `source`) for seamless data flow and compatibility.  
   - **Orchestration**: Handle conditional logic such as `count` at the root module level, keeping child modules focused on their resources.  
  
### Directory Structure: Flat vs. Modular

**Flat Module**: A single, monolithic directory with all resources in a few files. Simple for proofs of concept but becomes unmanageable as complexity grows.  

![Flat Module Structure](./images/2-FlatModuleStructure.jpg "Flat Module Structure")  

**Structured Module**: Each resource group (VCN, OKE, Bastion) is in its own module directory. The root module orchestrates dependencies and top-level configuration.  

![Advanced Module Structure](./images/3-AdvancedModuleStructure.jpg "Advanced Module Structure")  

**Example Modules**  
- **VCN Module (`modules/vcn`)**:  
  - **Purpose**: Manages network (VCN, subnets, gateways, routes, security lists, etc.).  
  - **Key Files**:  
    - `variables.tf`: Defines inputs such as `vcn_cidr_block`.  
    - `main.tf`: Contains resource definitions.  
    - `outputs.tf`: Exposes VCN and subnet IDs for other modules.  

- **OKE Module (`modules/oke`)**:  
  - **Purpose**: Deploys the OKE cluster and node pools.  
  - **Key Files**:  
    - `variables.tf`: Includes `vcn_id` and subnet IDs from the VCN module.  
    - `main.tf`: Refactored cluster and node pool definitions.  
    - `outputs.tf`: Exposes OKE cluster and node pool IDs.  

- **Bastion Module (`modules/bastion`)**:  
  - **Purpose**: Creates a bastion host for secure access.  
  - **Key Files**:  
    - `variables.tf`: Defines inputs such as `bastion_subnet_id`.  
    - `main.tf`: Refactored bastion host resources.  
    - `outputs.tf`: Exposes bastion host ID and public IP.    

**Why Modules?**  
- **Reusability and Collaboration**: Modules can be shared across projects, facilitating teamwork.  
- **Maintainability and Versioning**: Updates are applied consistently, reducing drift and ensuring compatibility.  
- **Scalability and Consistency**: Modular designs handle complexity efficiently, standardize deployments, and remove duplication.  

### Modules Orchestration - root `main.tf`   

The root `main.tf` orchestrates the deployment of three key modules (`modules/vcn`, `modules/oke`, and `modules/bastion`) in a sequenced manner. Each module is conditionally invoked based on configuration flags (`is_vcn_created`, `is_oke_created`, `is_bastion_created`), providing flexibility in deployment. Below is a simplified version of the orchestration flow, highlighting key module logic without detailing the full `main.tf`.  

**Orchestration Flow**:  
1. **VCN Module (`modules/vcn`)**:  
   - Provisions the virtual cloud network (VCN) and related subnets (for example, private subnets for Kubernetes API and worker nodes, public subnets for load balancers and bastion).  
   - Controlled by `is_vcn_created`. If enabled, it creates the VCN; otherwise, it assumes an existing VCN (you need to provide its used subnet OCID).  
   - Example snippet:  
     ```terraform
		module "vcn" {
			count = var.is_vcn_created ? 1 : 0  
			source = "./modules/vcn?ref=v1.0.0"  # Specify the module version  
			# Key variables: compartment_id, vcn_cidr_block, subnet configs, ...  
		}
     ```  
	**Notes:**
		- This blog assumes you are creating a new network using the provided VCN module.
		- If you want to use an existing VCN, refer to the first blog in this series, [Create Oracle Cloud Infrastructure Kubernetes Engine Cluster using Terraform](https://docs.oracle.com/en/learn/oke-clstr-trfm/), which introduces the `is_vcn_created` flag. 
		- To add this flexibility here, add/set `is_vcn_created` to `false` and provide the OCID of your VCN along with the OCIDs of subnets for the Kubernetes API endpoint, worker nodes, pods, load balancer, and bastion host. 

2. **OKE Module (`modules/oke`)**:  
   - Deploys the OCI Kubernetes Engine (OKE) cluster, including the control plane and optional managed node pools.  
   - Depends on the VCN module for subnet IDs. Only invoked if `is_oke_created` and `is_vcn_created` are true.  
   - Example snippet:  
     ```terraform
		module "oke" {
			count = var.is_oke_created && var.is_vcn_created ? 1 : 0  
			source = "./modules/oke?ref=v1.0.0"  # Specify the module version   
			# Key variables: vcn_id, subnet IDs, k8 version, node pool config, ...  
		}
     ```  

3. **Bastion Module (`modules/bastion`)**:  
   - Creates a bastion host for secure SSH access to private resources.  
   - Depends on the VCN module for the public subnet ID. Only invoked if `is_bastion_created` and `is_vcn_created` are true.  
   - Example snippet:  
     ```terraform
		module "bastion" {
			count = var.is_bastion_created && var.is_vcn_created ? 1 : 0  
			source = "./modules/bastion?ref=v1.0.0"  # Specify the module version   
			# Key variables: bastion_subnet_id, SSH keys, parameters...  
		}
     ```  

**Key Notes**:  
- **Module Dependencies**: Outputs from the VCN module such as `module.vcn[0].vcn_id` are passed as inputs to the OKE and Bastion modules, ensuring a clear dependency chain.  
- **Configuration Logic**: Simplified parameter maps (for example, `node_pool_param`, `bastion_params`) streamline configuration and readability.  
- **Versioning**: Using version constraints in `source` ensures modules are deployed with the correct and tested versions, ensuring compatibility.

After establishing a modular Terraform structure for OKE, the next step is to automate its deployment. Automation ensures consistency, reduces manual errors, accelerates the provisioning process, and directly improves Time-to-Market (TTM) by enabling rapid delivery of new features and services.

## Automations Options

Several tools can automate OKE deployments, including Terraform CLI, OCI  Resource Manager (ORM), OCI CLI, Ansible OCI modules, and Helm. However, this guide focuses on the two most prominent infrastructure as code (IaC) approaches in Oracle Cloud Infrastructure (OCI): **Terraform CLI** and **OCI Resource Manager (ORM)**.  

Both tools leverage the same declarative HashiCorp Configuration Language ([HCL](https://developer.hashicorp.com/terraform/language)) but differ in their operational models :
- **Terraform CLI**: A [Command Line Interface](https://developer.hashicorp.com/terraform/cli) tool offering direct control over infrastructure and state files, ideal for individual developers or small teams.  
- **OCI Resource Manager (ORM)**: A [console-based](https://docs.oracle.com/en-us/iaas/Content/ResourceManager/home.htm), fully managed, OCI-native service that centralizes state management and provides a secure, collaborative environment, making it the preferred choice for enterprise-scale deployments.  

Let’s explore each option in detail.  

###  Lab 1: Deploy OKE Resources with Terraform CLI

The **Terraform CLI** is ideal when you need complete control over your local environment. It’s best suited for individual developers or small teams who can manage the state file and collaborate effectively using a shared backend. Its portability allows you to run it from any machine: local, VM, container, OCI CloudShell, or CI/CD runners such as Jenkins. However, this flexibility comes with responsibilities, such as managing state files and ensuring consistent local setups across team members.

To begin, **download and unzip the Terraform CLI source code package** into your Terraform working directory. This package includes `main.tf`, a `terraform.tfvars` sample, and detailed module configurations: Download [`oke_advanced_module.zip`](./files/oke_advanced_module.zip "file").

**Deploying OKE with Terraform CLI involves seven key tasks**, from configuring variables and networking to setting up the OKE cluster, node pools, and bastion host. Below are the detailed steps to provision and verify your OKE environment using Terraform CLI.

#### Task 1.1: Configure Terraform Variables 
   Update the `terraform.tfvars` file with environment-specific details such as tenancy_ocid, region, compartment_ocid, and network_compartment_ocid. Enable the following flags to control resource creation:  
   - `is_vcn_created`: Create new or reuse an existing VCN.  
   - `is_okecluster_created`: Provision an OKE cluster.  
   - `is_nodepool_created`: Create one or more node pools.  
   - `is_bastion_created`: Deploy a bastion host.  

#### Task 1.2: Networking Configuration 
   Define CIDR blocks for the VCN and its subnets:  
   - `vcn_cidr_block`: VCN CIDR block.  
   - `k8apiendpoint_private_subnet_cidr_block`: Kubernetes API endpoint subnet.  
   - `workernodes_private_subnet_cidr_block`: Worker nodes private subnet.  
   - `pods_private_subnet_cidr_block`: Pods private subnet
   - `serviceloadbalancers_public_subnet_cidr_block`: Load balancer subnet.  
   - `bastion_public_subnet_cidr_block`: Bastion host subnet.  

#### Task 1.3: OKE Cluster Configuration  
   - Specify `control_plane_kubernetes_version`, `worker_nodes_kubernetes_version` and `cluster_type` (`BASIC_CLUSTER` or `ENHANCED_CLUSTER`).  
   - Choose a CNI type:  
     - `OCI_VCN_IP_NATIVE`: Pods get native OCI IPs.  
     - `FLANNEL_OVERLAY`: Pods get IPs from Flannel.  
   - Set `control_plane_is_public` to `true` or `false`.  

#### Task 1.4: Node Pool Configuration  
   - Define node pools under `node_pools` with:  
     - Shape, version, boot volume size, and AD placement (set 3 ADs if applicable)
     - SSH keys for access to the worker nodes in the pool
   - Enable `node_cycle_config` for safe node updates:  
     - `node_cycling_enabled`: Enable rolling node replacement.  
     - `maximum_surge` and `maximum_unavailable`: Control scaling during updates (e.g.: 1:0).  
     - `cycle_modes`: Choose `BOOT_VOLUME_REPLACE` or `INSTANCE_REPLACE`.  
	 Note: only `maximum_unavailable` parameter is needed for `BOOT_VOLUME_REPLACE`.

#### Task 1.5 Bastion Host Configuration  
   - If `is_bastion_created` is `true`, Terraform provisions a Linux bastion in the public subnet.  
   - Provide: `shape`, `hostname`, `boot_volume_size`, Linux image OCID, and SSH key paths.  

#### Task 1.6: Run Terraform Commands
Execute the following commands to deploy the infrastructure:  
   ```bash
	terraform init
	terraform validate	
	terraform plan
	terraform apply
   ```  

After `terraform apply` completes, your OKE cluster will be provisioned with:  
- A VCN with associated networking components (route tables, security lists, gateways), private subnets for worker nodes and the API endpoint, and a public subnet for load balancers.  
- An `ENHANCED_CLUSTER` with the specified CNI type and Kubernetes version, a managed node pool, and a bastion host (if configured) for secure SSH access.  

#### Task 1.7: Verification and Cleaning
1. Navigate to the OCI Console to verify the cluster configuration.  
2. Run `terraform destroy` to clean up resources when done.  

>**Automating OKE Deployment with Jenkins CI/CD Pipeline**  
> For integrating OKE deployment into DevOps pipelines, Terraform CLI is an excellent choice. Our approach, detailed in *[Create Oracle Cloud Infrastructure Kubernetes Engine Cluster using Terraform](https://docs.oracle.com/en/learn/oke-clstr-trfm/)*, uses bash scripting to orchestrate the process. This workflow can be consolidated into a Jenkins pipeline for automated execution. 


