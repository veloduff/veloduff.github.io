---
layout: post
title:  "Creating an HPC Cluster with AWS ParallelCluster"
date:   2024-06-15 10:00:00 -0700
categories: aws hpc
tags: [aws, hpc, parallelcluster, cloud-computing, infrastructure]
---

## Getting Started with AWS ParallelCluster

AWS ParallelCluster is a powerful tool for deploying and managing High Performance Computing (HPC) clusters in the AWS cloud. This guide walks through the essential steps to create your first HPC cluster using ParallelCluster.

## Prerequisites

Before you begin, ensure you have:
- AWS CLI installed and configured with appropriate permissions
- Basic understanding of Linux and SSH
- Python 3.11 or newer

## Step 1: ParallelCluster Installation

Let's start by creating a dedicated Python virtual environment for ParallelCluster:

```bash
python3.11 -m venv ~/Envs/ParallelCluster-01
source ~/Envs/ParallelCluster-01/bin/activate
```

Next, install ParallelCluster in the virtual environment (we'll use version 3.12 for this example):

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

## What's Next?

After your cluster is created, you'll have a basic HPC environment with:
- A head node accessible via SSH
- Compute nodes managed by the Slurm scheduler
- Basic job submission capabilities

However, to transform this into a production-ready HPC environment, you'll need additional customization. Check out my follow-up post on [Customizing an HPC cluster with ParallelCluster](/hpc/cluster-management/2025/07/20/unified-cluster-management-aws-hpc.html) to learn how to implement advanced cluster customization techniques.

---

*This post is part of a series on building production-ready HPC environments on AWS.*