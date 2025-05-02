**Key Points**

- It seems likely that deploying OpenStack with high availability on a single EC2 instance is not feasible for true HA, as it typically requires multiple nodes.
- Research suggests using multiple EC2 instances for a multi-node setup to achieve HA for Compute (Nova), Storage (Swift), and Database (Trove) services.
- The evidence leans toward interpreting "on an EC2 instance" as the deployment host, allowing OpenStack to be deployed across multiple EC2 instances for HA.

### Deployment Overview

To address the complexity of deploying OpenStack with Ansible on EC2 for high availability, we recommend launching multiple EC2 instances running Ubuntu 22.04. This approach ensures that Compute, Storage, and Database services can be configured for resilience and failover, which is not possible on a single node. Here's a simplified guide:

#### Launch EC2 Instances

- Launch at least 3 controller nodes, 2 compute nodes, and 3 storage nodes on EC2 with Ubuntu 22.04, ensuring each has sufficient resources (e.g., t3.xlarge for controllers, t3.2xlarge for computes, i3.xlarge for storage).

#### Set Up Deployment Host

- Choose one controller node as the deployment host, install Ansible, and clone the OpenStack-Ansible repository from [GitHub](https://github.com/openstack/openstack-ansible).

#### Configure and Deploy

- Configure the inventory file to assign roles (controller, compute, storage) to the EC2 instances.
- Run OpenStack-Ansible playbooks to deploy the environment, ensuring HA with Pacemaker for controllers, Galera for MySQL, and Swift replication across storage nodes.

#### Validate and Document

- Validate by creating VMs, testing Swift storage, and using Trove databases.
- Document the architecture, deployment steps, and troubleshooting tips for maintenance.

This approach ensures a robust, highly available OpenStack environment, suitable for the case study's requirements.

#### Background and Requirements

The task involves deploying OpenStack on an EC2 instance running Ubuntu Jammy (22.04) using Ansible, with specific requirements for HA and resilience. The document attached to the query outlines a case study for a Senior System Engineer, emphasizing a fully automated deployment including Nova, Swift, and Trove, with HA architecture, networking, security, error handling, monitoring, logging, and scalability. Notably, it mentions "potentially using multiple DevStack nodes or simulating HA within a single node," which introduces ambiguity regarding the number of EC2 instances.

Given that true HA typically requires multiple nodes for failover and redundancy, and considering the document's mention of "on an EC2 instance" (singular), we initially explored whether HA could be simulated on a single node. However, research suggests that simulating HA within a single node, such as using containerization or replication within the same hardware, does not provide true failover capabilities, as a single node failure would affect all services. Therefore, we interpret "on an EC2 instance" as referring to the deployment host being on EC2, while the OpenStack environment itself is deployed across multiple EC2 instances to achieve HA, aligning with standard practices.

#### Deployment Architecture Design

To meet the HA requirements, we propose a multi-node OpenStack deployment using OpenStack-Ansible, with the following architecture:

- **Deployment Host**: One EC2 instance running Ubuntu 22.04, serving as the Ansible control node.
- **Controller Nodes**: At least 3 EC2 instances (e.g., controller-1, controller-2, controller-3) for HA, hosting services like Keystone, Glance, Nova API, Neutron, and database services.
- **Compute Nodes**: At least 2 EC2 instances (e.g., compute-1, compute-2) for Nova compute services, supporting auto-scaling and instance migration.
- **Storage Nodes**: At least 3 EC2 instances (e.g., storage-1, storage-2, storage-3) for Swift object storage, ensuring replication for HA.

This setup ensures redundancy and failover capabilities, with Pacemaker and Corosync managing HA for controller services, Galera for MySQL replication, and Swift configured for replication across storage nodes.

#### Step-by-Step Implementation

Below is a detailed guide for implementing the deployment, based on OpenStack-Ansible documentation and best practices:

##### Step 1: Launch EC2 Instances

- Use the AWS Management Console or AWS CLI to launch EC2 instances with Ubuntu 22.04 LTS AMI. Recommended instance types:
  - Controller nodes: t3.xlarge (4 vCPUs, 16 GiB memory).
  - Compute nodes: t3.2xlarge (8 vCPUs, 32 GiB memory) for better performance.
  - Storage nodes: i3.xlarge (4 vCPUs, 30.5 GiB memory, 1x950 NVMe SSD) for high I/O.
- Ensure all instances are in the same VPC with appropriate security groups allowing SSH (port 22) and OpenStack service ports (e.g., 80, 443, 5000, etc.).
- Assign static private IPs and ensure connectivity, using AWS key pairs for SSH access.

##### Step 2: Set Up the Deployment Host

- Select one controller node (e.g., controller-1) as the deployment host for simplicity, or launch a separate small instance (e.g., t3.medium).
- Update and upgrade the system:

sudo apt update && sudo apt dist-upgrade -y

sudo reboot

 Install required packages and configure SSH for key-based authentication without passphrases, using the AWS key pair.

 Install Ansible and clone the OpenStack-Ansible repository:

sudo apt install ansible -y

git clone <https://github.com/openstack/openstack-ansible.git> /opt/openstack-ansible

cd /opt/openstack-ansible

scripts/bootstrap-ansible.sh

Ensure the deployment host can SSH to all target hosts without password prompts, using ssh-agent if needed.

##### Step 3: Configure Inventory and Roles

- Create the inventory file at /etc/openstack_deploy/openstack_user_config.yml with the following structure:
- \- hosts: controller
-   hosts:
-     controller-1:
-       ip: &lt;private-ip-of-controller-1&gt;
-     controller-2:
-       ip: &lt;private-ip-of-controller-2&gt;
-     controller-3:
-       ip: &lt;private-ip-of-controller-3&gt;
- \- hosts: compute
-   hosts:
-     compute-1:
-       ip: &lt;private-ip-of-compute-1&gt;
-     compute-2:
-       ip: &lt;private-ip-of-compute-2&gt;
- \- hosts: storage
-   hosts:
-     storage-1:
-       ip: &lt;private-ip-of-storage-1&gt;
-     storage-2:
-       ip: &lt;private-ip-of-storage-2&gt;
-     storage-3:
-       ip: &lt;private-ip-of-storage-3&gt;

Configure user variables in /etc/openstack_deploy/user_variables.yml for HA, such as enabling Pacemaker and Galera:

galera_cluster_members:

&nbsp; - controller-1

&nbsp; - controller-2

&nbsp; - controller-3

pacemaker_enabled: true

swift_replication_count: 3

##### Step 4: Deploy OpenStack Using Ansible

- Run the OpenStack-Ansible playbooks in sequence:
  - Setup hosts:
- openstack-ansible setup-hosts.yml

Setup infrastructure (e.g., RabbitMQ, Galera):

openstack-ansible setup-infrastructure.yml

Setup OpenStack services:

openstack-ansible setup-openstack.yml

- Ensure HA configurations are applied:
  - Pacemaker and Corosync manage controller services for failover.
  - Galera ensures MySQL replication across controller nodes for Trove.
  - Swift is configured with a replication factor of 3 across storage nodes.

##### Step 5: Configure Additional Requirements

- **Networking**: Use OpenStack Neutron to set up private and public networks, implement security groups, and configure firewall rules. Ensure proper routing between networks.
- **Security**: Enable key-pair authentication for compute instances, set up RBAC in Keystone, and configure Swift storage encryption.
- **Error Handling and Rollback**: Ensure Ansible playbooks are idempotent and include rollback mechanisms for failed deployments.
- **Monitoring and Logging**: Deploy Ceilometer for monitoring and centralize logs for Nova, Swift, and Trove using tools like Logstash or ELK stack.
- **Scalability**: Configure Nova for auto-scaling using Heat templates and ensure Swift supports dynamic scaling by adding storage nodes.

##### Step 6: Validate the Deployment

- Verify all OpenStack services are operational:
  - Access Horizon dashboard on port 443 (or 80 if SSL disabled) using admin credentials from /etc/openstack_deploy/user_secrets.yml.
- Test Compute (Nova):
  - Launch a VM and access it via SSH using a floating IP, ensuring auto-scaling works.
- Test Storage (Swift):
  - Upload and download files, verify replication across storage nodes.
- Test Database (Trove):
  - Create a database instance and execute queries, ensuring MySQL replication is active.
- Test HA:
  - Simulate a controller node failure by stopping one (e.g., sudo shutdown -h now on controller-2) and verify services failover to other nodes using Pacemaker.

##### Step 7: Document the Solution

- **Architecture**: Document the multi-node setup, including IP addresses, roles, and HA configurations. Use diagrams if possible.
- **Playbook Execution**: Provide detailed commands and expected execution times (e.g., 30-50 minutes on bare metal SSD).
- **Validation Methods**: Include steps for verifying each service and HA, such as checking Pacemaker status with pcs status.
- .

#### Addressing Single-Node Simulation

If strictly limited to a single EC2 instance, HA cannot be truly achieved, as a single node failure would affect all services. However, we can simulate HA by:

- Running multiple instances of services (e.g., Nova compute) in containers using LXC, though this does not provide failover.
- Configuring MySQL with replication within the same node, which is not true HA but simulates the configuration.
- Configuring Swift with replication within the same node, again not providing true HA.

#### Tables for Clarity

Below is a table summarizing the recommended EC2 instance types and roles:

| **Role** | **Number of Instances** | **Recommended Instance Type** | **Purpose** |
| --- | --- | --- | --- |
| Controller | 3   | t3.xlarge | HA for control plane services (Keystone, etc.) |
| Compute | 2   | t3.2xlarge | Run VMs, support auto-scaling |
| Storage | 3   | i3.xlarge | Swift object storage with replication |
| Deployment Host | 1   | t3.xlarge | Run Ansible for deployment |

Another table for key HA configurations:

| **Service** | **HA Configuration** | **Notes** |
| --- | --- | --- |
| Nova (Compute) | Pacemaker for nova-compute failover | Requires shared storage for instance HA |
| Swift (Storage) | Replication across 3 storage nodes | Ensures data availability on node failure |
| Trove (Database) | Galera for MySQL replication | Ensures database resilience across nodes |

**This was a suggested solution, but to answer the question, it was running on a single node :**

## 🖥️ Recommended System Requirements

- **CPU**: 8 cores with hardware-assisted virtualization support (VT-x or AMD-V)
- **RAM**: 16 GB minimum
- **Disk**: 80 GB free space on the root partition or 60 GB on a secondary disk (requires setting bootstrap_host_data_disk_device)
- **Operating System**: Ubuntu 20.04 or 22.04 LTS​

Note: AIO deployments can be performed on virtual machines; however, performance may degrade without nested virtualization support.

## Deployment Steps

### 1\. Prepare the Host

Update all system packages and reboot into the latest kernel:

sudo apt update && sudo apt dist-upgrade -y

sudo reboot

**2\. Clone the OpenStack-Ansible Repository**

Clone the repository and navigate to its directory:

git clone <https://opendev.org/openstack/openstack-ansible> /opt/openstack-ansible

cd /opt/openstack-ansible

**3\. Bootstrap Ansible and Required Roles**

Run the bootstrap script to install Ansible and necessary roles:

scripts/bootstrap-ansible.sh

**4\. Bootstrap the AIO Configuration**

Prepare the AIO configuration:

scripts/bootstrap-aio.sh

**5\. Run Playbooks to Deploy OpenStack**

Execute the playbooks to set up OpenStack services:​

cd /opt/openstack-ansible/playbooks

openstack-ansible setup-hosts.yml

openstack-ansible setup-infrastructure.yml

openstack-ansible setup-openstack.yml

## 🔍 Post-Deployment Verification

- **List LXC Containers**: lxc-ls -f
- **Access a Container**: lxc-attach --name &lt;container-name&gt;
- **Check OpenStack Services**: openstack service list

For a streamlined setup, you can use the one-step AIO build script:​

curl <https://raw.githubusercontent.com/openstack/openstack-ansible/master/scripts/run-aio-build.sh> | sudo bash

# Comprehensive Roadmap for Deploying Rancher and RKE2 on OpenStack

This roadmap provides a structured approach to deploying Rancher's enterprise Kubernetes distribution (RKE2) on OpenStack cloud infrastructure. It outlines the necessary steps, considerations, and best practices to ensure a successful implementation.

## **Introduction to Rancher and RKE2**

RKE2 is Rancher's enterprise-ready next-generation Kubernetes distribution, previously known as RKE Government. It is a fully conformant Kubernetes distribution that focuses on security and compliance, particularly within the U.S. Federal Government sector. RKE2 provides several security-focused features, including:

- Default configurations that enable clusters to pass the CIS Kubernetes Benchmark v1.7 or v1.8 with minimal operator intervention
- FIPS 140-2 compliance capabilities
- Regular vulnerability scanning of components using trivy in the build pipeline

Unlike its predecessor RKE1, RKE2 doesn't rely on Docker for deploying and managing control plane components. Instead, it launches control plane components as static pods managed by the kubelet, with containerd as the embedded container runtime. This architectural difference brings RKE2 closer to standard Kubernetes deployments while maintaining the usability and ease of operations inherited from K3s.

## **Prerequisites and Planning**

## Infrastructure Requirements

Before beginning deployment, ensure you have:

1. Access to an OpenStack environment with sufficient privileges
2. OpenStack CLI tools installed and configured
3. Adequate quota allocations for compute, network, and storage resources
4. A well-defined network topology and security plan

## Resource Planning

For a production environment, plan the following resources:

- Compute: At least three nodes for a highly available cluster (for etcd and control plane components)
- Network: Isolated network with appropriate subnets
- Storage: Persistent storage options for stateful workloads
- Security Groups: Properly configured to allow necessary traffic between nodes

## **Setting Up OpenStack Infrastructure**

## Project Creation

Start by creating a dedicated project to isolate your Kubernetes resources:

openstack project create --domain default --description "RKE2 Cluster" rke2

openstack role add --project rke2 --user admin admin

Update your environment variables to use this project:

export OS_PROJECT_ID=&lt;project_id&gt;

export OS_PROJECT_NAME=rke2

## Network Configuration

Create the necessary network infrastructure:

1. Create a network:

text

openstack network create --project rke2 rke2

1. Create a subnet:

text

openstack subnet create rke2-subnet --project rke2 --network rke2 --subnet-range 172.31.0.0/28

1. Create a router to provide external connectivity:

text

openstack router create rke2-router --project rke2

## Security Group Configuration

Create security groups to control network access:

1. Allow internal communication between cluster nodes
2. Allow necessary external access for management and application traffic
3. Configure specific rules for Kubernetes API server, etcd, and worker node communication

## **Installing and Configuring RKE2**

## Node Preparation

For each node in your OpenStack environment:

1. Deploy instances using an appropriate image (Ubuntu, RHEL, etc.)
2. Ensure each node has sufficient resources (CPU, memory, storage)
3. Install required dependencies

Unlike RKE1, which requires Docker, RKE2 uses containerd as its container runtime, eliminating the need to install Docker on the nodes.

## RKE2 Server Installation

For the first server node:

1. Install RKE2 server:

text

curl -sfL <https://get.rke2.io> | sh -

1. Enable and start the service:

text

systemctl enable rke2-server.service

systemctl start rke2-server.service

1. Configure the OpenStack cloud provider integration in the RKE2 configuration file (/etc/rancher/rke2/config.yaml)

## Adding Additional Server Nodes

For high availability, add additional server nodes:

1. Obtain the token from the first server:

text

cat /var/lib/rancher/rke2/server/node-token

1. Install RKE2 on additional servers with the token and first server URL

## Adding Worker Nodes

For worker nodes:

1. Install RKE2 agent:

text

curl -sfL <https://get.rke2.io> | INSTALL_RKE2_TYPE="agent" sh -

1. Configure the agent to connect to the server cluster

## **Integrating with OpenStack Cloud Provider**

The OpenStack cloud provider enables Kubernetes to interact with OpenStack resources, such as volumes and load balancers. When configuring RKE2 for OpenStack:

1. Disable the default RKE2 cloud controller if you're planning to use the OpenStack cloud controller manager, as they may conflict by binding to the same port
2. Configure the OpenStack cloud provider with the necessary credentials and endpoint information
3. Create a cloud-config file with your OpenStack credentials:

text

\[Global\]

auth-url=&lt;OS_AUTH_URL&gt;

username=&lt;OS_USERNAME&gt;

password=&lt;OS_PASSWORD&gt;

tenant-id=&lt;OS_PROJECT_ID&gt;

domain-name=&lt;OS_USER_DOMAIN_NAME&gt;

region=&lt;OS_REGION_NAME&gt;

## **Deploying Rancher on RKE2**

## Rancher Installation

Once your RKE2 cluster is operational:

1. Install the Helm package manager
2. Add the Rancher Helm repository:

text

helm repo add rancher-latest <https://releases.rancher.com/server-charts/latest>

1. Create a namespace for Rancher:

text

kubectl create namespace cattle-system

1. Install Rancher using Helm:

text

helm install rancher rancher-latest/rancher \\

\--namespace cattle-system \\

\--set hostname=&lt;RANCHER_HOSTNAME&gt; \\

\--set bootstrapPassword=&lt;INITIAL_ADMIN_PASSWORD&gt;

## Rancher Configuration

After installation:

1. Access the Rancher UI using the configured hostname
2. Complete the initial setup and set an admin password
3. Enable the OpenStack node driver in Rancher through the UI (Tools > Drivers > Node Drivers > OpenStack > Activate)

## **Post-Deployment Validation**

## Validation Checks

Verify the deployment with the following checks:

1. Ensure all RKE2 components are running:

text

kubectl get pods -A

1. Verify the Rancher deployment:

text

kubectl get pods -n cattle-system

1. Check integration with OpenStack cloud provider:

text

kubectl get nodes

- - Nodes should not show the taint node.cloudprovider.kubernetes.io/uninitialized if cloud provider integration is working correctly

## Troubleshooting Common Issues

1. If Rancher remains in a pending state, check if there's a conflict between the default RKE2 cloud controller and the OpenStack cloud controller manager
2. For networking issues, verify your OpenStack security groups and network configuration
3. For authentication problems with the OpenStack cloud provider, check your credentials and endpoint configurations

## **Conclusion and Best Practices**

Deploying Rancher and RKE2 on OpenStack provides a powerful, secure, and compliant Kubernetes platform. For optimal performance and reliability:

- Implement proper high availability for both RKE2 and Rancher
- Regularly update RKE2 and Rancher to benefit from security patches and new features
- Set up monitoring and logging for the entire stack
- Implement proper backup strategies for etcd and Rancher data
- Consider using infrastructure as code tools like Terraform for reproducible deployments

By following this roadmap and adhering to best practices, you can successfully deploy and manage Rancher and RKE2 on OpenStack, leveraging the strengths of both platforms for your containerized workloads.
