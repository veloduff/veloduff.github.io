---
layout: post
title:  "Why Unified Cluster Management Matters for AWS HPC Workloads"
date:   2025-07-20 16:30:00 -0700
categories: hpc cluster-management
tags: [aws, hpc, parallelcluster, pdsh, cluster-management, devops]
---

## Introduction: The Challenge of HPC Cluster Management

High Performance Computing (HPC) workloads are increasingly moving to the cloud, but a cluster challenge remains: how do you efficiently manage dozens or hundreds of nodes as a single, cohesive system? While AWS ParallelCluster excels at deploying infrastructure, transforming these individual EC2 instances into a truly manageable cluster requires additional tooling and configuration.

In production environments, administrators need to:
- Execute commands across all nodes simultaneously
- Monitor system health and performance at scale
- Deploy software and updates efficiently
- Troubleshoot issues across the entire cluster

This is where unified cluster management becomes essential, and pdsh (Parallel Distributed Shell) can be used to address this issue. In this post, I'll show how my `cluster_setup.sh` script adds functionality to ParallelCluster deployment to make it a fully manageable HPC environment through automated pdsh configuration. ClusterShell (`clush`) is another option, but the reasons to choose `pdsh` over `clush` are beyond the scope of this blog. 

## The Power of Unified Management with pdsh

`pdsh` allows administrators to run commands across hundreds of nodes with a single instruction. For example:

```bash
# Check disk usage across all nodes
pdsh df -h | dshbak -c

# Monitor system load
pdsh uptime | dshbak -c

# Install software across the cluster
pdsh sudo yum install -y htop | dshbak -c
```

This capability dramatically simplifies cluster administration, reduces human error, and enables rapid response to issues. My approach automates the complex setup required to enable this functionality, including SSH trust configuration, host file management, and environment setup.

In this blog, I'll walk through:
1. Setting up the foundation with ParallelCluster
2. Implementing unified cluster management with pdsh
3. Validating the cluster's functionality
4. Leveraging pdsh for common administrative tasks

## Prerequisites

- AWS CLI installed and configured with appropriate permissions
- Basic understanding of Linux and SSH
- Python 3.11 or newer

## Step 1: ParallelCluster Installation

Let's start by creating a dedicated Python virtual environment for ParallelCluster.
Python3.11 is not a requirement, it's just what I'm using here:

```bash
python3.11 -m venv ~/Envs/ParallelCluster-01
source ~/Envs/ParallelCluster-01/bin/activate
```

Next, install ParallelCluster in the virtual environment (we'll use version ParallelCluster 3.12 for this example):

```bash
(ParallelCluster-01)$ pip install aws-parallelcluster==3.12
```

> **Note:** If you encounter issues with setuptools, you may need to fix the version:
> ```bash
> pip install setuptools==69.5.1
> ```

Verify your installation:

```bash
(ParallelCluster-01)$ pcluster version
{
  "version": "3.12.0"
}
```

## Step 2: Verify AWS CLI Configuration

Before proceeding, ensure your AWS CLI is properly configured:

```bash
$ aws sts get-caller-identity
```

You should see output similar to one of these examples:

For SSO login (using Identity Management):
```json
{
    "UserId": "AAAAAAAAAAAAAAAAAAAAA:jouser",
    "Account": "111111111111",
    "Arn": "arn:aws:sts::111111111111:assumed-role/AWSnnnnSSO_admin_1111111111111111/jouser"
}
```

For local credentials with IAM role:
```
111122223333    arn:aws:iam::111122223333:user/jouser           XXXXXXXXXXXXXXXXXXXXX
```

## Step 3: Create the ParallelCluster VPC

ParallelCluster can automatically create a VPC with appropriate networking for your cluster. This is the recommended approach for most users:

```bash
(ParallelCluster-01)$ pcluster configure --config cluster-01.yaml
```

When prompted, select the following options:
- Automate VPC creation: `y`
- Choose an Availability Zone (e.g., `us-west-2a`)
- Network Configuration: `Head node in a public subnet and compute fleet in a private subnet`

This creates a secure network architecture where:
- The head node is accessible via SSH from the internet (with security group restrictions)
- Compute nodes are in a private subnet with no direct internet access
- NAT Gateway enables compute nodes to access the internet for updates and package installation

After completion, you'll have a configuration file (`cluster-01.yaml`) with subnet IDs for both the head node and compute fleet:

```yaml
HeadNode:
  Networking:
    SubnetId: subnet-11111111111111111

Scheduling:
  Scheduler: slurm
  SlurmQueues:
  - Name: queue01
    Networking:
      SubnetIds:
        - subnet-22222222222222222
```

## Step 4: Customize Your Cluster Configuration

The auto-generated configuration is a starting point, but you'll want to customize it for your specific workload. Here's an example configuration for a medium-sized HPC cluster with dedicated storage and compute nodes:

```yaml
Region: us-west-2
Image:
  Os: rhel8
HeadNode:
  InstanceType: m6id.2xlarge
  DisableSimultaneousMultithreading: false
  Ssh:
    KeyName: ec2-key-pdx
  Networking:
    ElasticIp: true
    SubnetId: subnet-XXXXXXXXXXXXXXXXX
    AdditionalSecurityGroups:
      - sg-XXXXXXXXXXXXXXXXX
AdditionalPackages:
  IntelSoftware:
    IntelHpcPlatform: false
Scheduling:
  Scheduler: slurm
  SlurmSettings:
    ScaledownIdletime: 10
    Dns:
      DisableManagedDns: true
  SlurmQueues:
  - Name: storage01
    CapacityType: ONDEMAND
    Networking:
      SubnetIds:
        - subnet-XXXXXXXXXXXXXXXXX
      PlacementGroup:
        Enabled: true
        Name: cluster-01-placement-group-01
    ComputeResources:
    - Name: storage
      InstanceType: m6idn.2xlarge
      MinCount: 8
      MaxCount: 16
      DisableSimultaneousMultithreading: false
  - Name: batch01
    CapacityType: ONDEMAND
    Networking:
      SubnetIds:
        - subnet-XXXXXXXXXXXXXXXXX
      PlacementGroup:
        Enabled: true
        Name: cluster-01-placement-group-01
    ComputeResources:
    - Name: name
      InstanceType: m6idn.xlarge
      MinCount: 16
      MaxCount: 32
      DisableSimultaneousMultithreading: true
```

Key configuration elements:

1. **Instance Types**:
   - Head node: `m6id.2xlarge` provides a balance of compute, memory, and local storage
   - Storage nodes: `m6idn.2xlarge` with enhanced network performance
   - Compute nodes: `m6idn.xlarge` optimized for compute workloads

2. **Security**:
   - ElasticIP for consistent access
   - Additional security groups for organizational requirements
   - Private subnet for compute nodes

3. **Performance**:
   - Placement groups for low-latency networking
   - Selective disabling of simultaneous multithreading
   - Instance types with local NVMe storage

4. **Scaling**:
   - Minimum and maximum node counts for each resource type
   - ScaledownIdletime for cost optimization

> **Security Warning:** By default, the security group created for the head node allows SSH traffic from any source (0.0.0.0/0). This inbound rule should be updated to a specific IP address or range.

## Step 5: Create the Cluster

Now that you have a customized configuration, create your cluster:

```bash
(ParallelCluster-01)$ pcluster create-cluster -n test-cluster01 -c cluster-01.yaml --rollback-on-failure false
```

The `--rollback-on-failure false` flag is recommended for initial deployments as it allows for easier troubleshooting if issues arise.

## Step 6: Implementing Unified Cluster Management

This is where my approach truly differentiates from a basic ParallelCluster deployment. The `cluster_setup.sh` script transforms individual EC2 instances into a cohesive, manageable cluster:

1. **Prepare the required scripts**:
   - `cluster_setup.sh`: Main cluster setup and validation script
   - `install_pkgs.sh`: Package installation dependency script

   Copy these scripts to your head node:
   ```bash
   scp cluster_setup.sh install_pkgs.sh ec2-user@<head-node-ip>:~/
   ```

2. **Execute the setup script**:
   ```bash
   chmod +x cluster_setup.sh install_pkgs.sh
   ./cluster_setup.sh
   ```

### How the Unified Management System Works

The `cluster_setup.sh` script performs nine critical operations that transform individual EC2 instances into a cohesive, manageable cluster:

1. **Package Management**: Installs essential HPC tools and utilities across all nodes
2. **Cluster Communication**: Sets up pdsh (Parallel Distributed Shell) for parallel command execution with proper SSH configurations
3. **Host Management**: Creates and distributes cluster host files by extracting node information from Slurm
4. **SSH Configuration**: Enables passwordless root access across all cluster nodes for administrative tasks
5. **Environment Setup**: Configures `.bash_profile` for optimal cluster operations on all nodes
6. **Storage Management**: Cleans up Instance Store devices for filesystem use (Lustre/GPFS)
7. **Monitoring Tools**: Installs performance monitoring and debugging utilities for cluster-wide diagnostics
8. **MPI Validation**: Creates, compiles, and tests MPI applications to verify cluster functionality
9. **Slurm Testing**: Submits test jobs to validate scheduler functionality across the cluster

### Real-World Benefits of Unified Management

With this configuration in place, cluster administrators can now:

1. **Execute parallel commands** across the entire cluster with a single instruction
2. **Monitor system health** in real-time across all nodes
3. **Deploy software and updates** consistently and efficiently
4. **Troubleshoot issues** by quickly comparing outputs across nodes

For example, to check system load across all nodes:
```bash
pdsh uptime | dshbak -c
```

To verify disk space usage:
```bash
pdsh df -h | dshbak -c
```

To install software across the cluster:
```bash
pdsh sudo yum install -y htop | dshbak -c
```

## Step 7: Validating Your Unified Management System

After running the setup script, verify that your unified cluster management system is functioning correctly:

1. **Test pdsh functionality** by running a simple command across all nodes:
   ```bash
   $ pdsh hostname
   batch01-st-batch-01: batch01-st-batch-01
   batch01-st-batch-02: batch01-st-batch-02
   batch01-st-batch-03: batch01-st-batch-03
   ...
   ```

2. **Verify host file consistency** across the cluster:
   ```bash
   $ pdsh cat /etc/hosts | dshbak -c
   ```

3. **Check MPI test results** to confirm inter-node communication:
   ```bash
   $ cat mpi-test.out
   Currently Loaded Modulefiles:
    1) openmpi/4.1.7
   Hello world from processor batch01-st-batch-31, rank 30 out of 32 processors
   Hello world from processor batch01-st-batch-27, rank 26 out of 32 processors
   ...
   ```

## Advanced Unified Management Techniques

With your unified management system in place, you can now implement advanced cluster management techniques:

### 1. Cluster-Wide Monitoring

Create a simple monitoring script that runs via pdsh:

```bash
#!/bin/bash
echo "===== CLUSTER STATUS REPORT ====="
echo "\n== SYSTEM LOAD =="
pdsh uptime | dshbak -c

echo "\n== MEMORY USAGE =="
pdsh free -h | dshbak -c

echo "\n== DISK USAGE =="
pdsh df -h | grep -v tmpfs | dshbak -c

echo "\n== RUNNING PROCESSES =="
pdsh "ps -eo pcpu,pmem,pid,user,args | sort -k 1 -r | head -5" | dshbak -c
```

### 2. Parallel File Operations

Leverage pdcp (parallel copy) for efficient file distribution:

```bash
# Copy configuration files to all nodes
pdcp -w ^$HOME/cluster-ip-addr /path/to/config.file /destination/path/

# Distribute application binaries
pdcp -r -w ^$HOME/cluster-ip-addr /path/to/application/ /opt/apps/
```

### 3. Cluster Health Checks

Implement automated health checks that run periodically:

```bash
# Check for failed services
pdsh "systemctl list-units --state=failed" | dshbak -c

# Verify network connectivity between nodes
pdsh "ping -c 1 $(hostname)" | dshbak -c
```

## Conclusion: The Value of Unified Management

While AWS ParallelCluster provides an excellent foundation for HPC infrastructure, the unified management layer I've implemented transforms a collection of instances into a true HPC cluster. This approach delivers several key benefits:

1. **Operational Efficiency**: Administrators can manage hundreds of nodes as easily as they manage one
2. **Consistency**: Commands and configurations are applied uniformly across the cluster
3. **Troubleshooting Speed**: Problems can be quickly identified and resolved across the entire system
4. **Scalability**: The same management techniques work whether you have 10 nodes or 1,000

By implementing unified cluster management with pdsh, you've created not just a cluster of compute resources, but a cohesive, manageable system ready for production HPC workloads.

---

*This post is part of a series on building production-ready HPC environments on AWS. In my next post, I'll explore how to leverage this unified management approach for parallel filesystem deployment and optimization.*
