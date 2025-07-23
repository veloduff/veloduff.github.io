---
layout: post
title:  "Lustre Deployment with Ansible"
date:   2025-07-21 07:00:00 -0700
categories: aws hpc lustre parallelcluster ansible automation
---

Deploying high-performance Lustre filesystems on AWS ParallelCluster traditionally requires extensive manual configuration and coordination across multiple components. This Ansible-based automation process provides an interactive deployment that handles everything from cluster sizing to post-installation configuration.

## What This Automation Does

The `run-pcluster-lustre.sh` script provides a complete end-to-end deployment solution:
- **Interactive configuration** with intelligent defaults
- **Pre-configured cluster sizes** optimized for different workloads
- **Automated Lustre setup** with proper component distribution
- **Post-installation scripts** for immediate usability
- **Comprehensive validation** of prerequisites and credentials


## Getting Started

### Prerequisites
```bash
# Install required tools
pip install ansible aws-parallelcluster

# Configure AWS credentials
aws configure
```
#### Custom AMI
This process depends on the Lustre, DKMS, and ZFS modules being installed. It will handle loading of the modules, but they need to be installed. For customizing a ParallelCluster AMI, see my [Building Custom ParallelCluster AMIs with Lustre Server Support]({% post_url 2025-01-22-custom-parallelcluster-ami-lustre-server %}) blog post.

#### ParallelCluster VPC

This process depends on ParallelCluster being configured, and using ParallelCluster to setup the VPC is the recommended way of setting up the VPC. See my 
[Creating an HPC Cluster with AWS ParallelCluster]({% post_url 2024-06-15-creating-hpc-cluster-aws-parallelcluster %}) blog post for setting up a ParallelCluster VPC.

### Basic Deployment
```bash
# Clone the repository and navigate to the playbook directory
cd ansible-playbooks/pcluster-lustre

# Run the interactive setup
$ ./run-pcluster-lustre.sh
ParallelCluster Lustre Cluster Ansible Setup
============================================
Verifying AWS credentials... verified
Cluster name [lustre-cluster-Jul23-20250642]:
AWS region []: us-west-2
Custom AMI []: ami-111122223333
Operating System []: rhel8
SSH key file path []: /path/to/my-key.pem
EC2 key pair name [my-key]:
Head node subnet ID []: subnet-12121212
Compute subnet ID []: subnet-23232323
Placement group name []: my-placement-group-01
File system size (small/medium/large/xlarge/local) [small]: large
```

### Example file system, shown for a 1PB file system:

```sh
$ lfs df -h

...
testfs-OST0032_UUID        15.7T        8.0M       15.7T   1% /mnt/lustre[OST:50]
testfs-OST0033_UUID        15.7T        8.0M       15.7T   1% /mnt/lustre[OST:51]
testfs-OST0034_UUID        15.7T        8.0M       15.7T   1% /mnt/lustre[OST:52]
testfs-OST0035_UUID        15.7T        8.0M       15.7T   1% /mnt/lustre[OST:53]
testfs-OST0036_UUID        15.7T        8.0M       15.7T   1% /mnt/lustre[OST:54]
testfs-OST0037_UUID        15.7T        8.0M       15.7T   1% /mnt/lustre[OST:55]
testfs-OST0038_UUID        15.7T        8.0M       15.7T   1% /mnt/lustre[OST:56]
testfs-OST0039_UUID        15.7T        8.0M       15.7T   1% /mnt/lustre[OST:57]
testfs-OST003a_UUID        15.7T        8.0M       15.7T   1% /mnt/lustre[OST:58]
testfs-OST003b_UUID        15.7T        8.0M       15.7T   1% /mnt/lustre[OST:59]
testfs-OST003c_UUID        15.7T        8.0M       15.7T   1% /mnt/lustre[OST:60]
testfs-OST003d_UUID        15.7T        8.0M       15.7T   1% /mnt/lustre[OST:61]
testfs-OST003e_UUID        15.7T        8.0M       15.7T   1% /mnt/lustre[OST:62]
testfs-OST003f_UUID        15.7T        8.0M       15.7T   1% /mnt/lustre[OST:63]

filesystem_summary:      1007.1T       11.3G     1007.1T   1% /mnt/lustre
```

## Pre-Configured File System Sizes

The total size of the file system will depend on the size of the cluster and the size of each OST. 

In the `ansible-playbooks/pcluster-lustre/lustre_fs_settings.sh` file the size of the OST and number of OSTs per OSS can be changed. Here are the settings for the ***small*** file system, that is 96TB file system with 40K IOPS:

```sh
        "small")
            # High performance: 40K IOPS, 96TB capacity
            MDT_USE_LOCAL=false
            OST_USE_LOCAL=false
            
            # MGT will be mirrored volumes
            MGT_SIZE=1                         # Size (GB) for MGT volumes
            MGT_VOLUME_TYPE="gp3"              # Volume type for MDT (io1, io2, gp3)
            MGT_THROUGHPUT=125                 # MDT Throughput in MiB/s
            MGT_IOPS=3000                      # MDT IOPS
            
            # Settings for MDTs when *NOT* using local disk (see MDT_USE_LOCAL)
            MDTS_PER_MDS=1                     # Number of MDTs to create per MDS server
            MDT_VOLUME_TYPE="io2"              # Volume type for MDT (io1, io2, gp3)
            MDT_THROUGHPUT=1000                # MDT Throughput in MiB/s
            MDT_SIZE=512                       # Size (GB) for MDT volumes
            MDT_IOPS=12000                     # MDT IOPS
            
            # Settings for OSTs when *NOT* using local disk (see OST_USE_LOCAL)
            OSTS_PER_OSS=1                     # Number of OSTs to create per OSS server
            OST_VOLUME_TYPE="io1"              # Volume type for OST (io1, io2, gp3) 
            OST_THROUGHPUT=250                 # Throughput in MiB/s
            OST_SIZE=1200                      # Size (GB) for OST volumes 
            OST_IOPS=3000                      # IOPS
            ;;
```

Note: Each MDS will get one MDT, and the MGS has mirrored MGT volumes.

## Pre-Configured Cluster Sizes

The script includes five optimized configurations for different use cases:

| Configuration | Use Case | Head Node | MGS | MDS | OSS | Compute |
|---------------|----------|-----------|-----|-----|-----|---------|
| **Small** | Development, testing, small workloads | m6idn.xlarge | 1x m6idn.large | 2-8x m6idn.xlarge | 8-16x m6idn.xlarge | 4-32x m6idn.large |
| **Medium** | Production workloads, moderate scale | m6idn.xlarge | 1x m6idn.xlarge | 4-8x m6idn.xlarge | 20-40x m6idn.xlarge | 8-128x m6idn.large |
| **Large** | High-performance computing, large datasets | m6idn.2xlarge | 1x m6idn.xlarge | 8-16x m6idn.2xlarge | 40-128x m6idn.2xlarge | 16-256x m6idn.xlarge |
| **XLarge** | Extreme scale, mission-critical workloads | m6idn.2xlarge | 1x m6idn.xlarge | 16x m6idn.2xlarge (fixed) | 40-128x m6idn.2xlarge | 16-256x m6idn.xlarge |
| **Local** | Maximum performance with local NVMe storage | m6idn.2xlarge | 1x m6idn.xlarge | 16-32x m6idn.2xlarge | 40-64x m6idn.2xlarge | 16-256x m6idn.xlarge |


## Automated Post-Installation Pipeline

The script orchestrates the post-installation process:

### 1. Cluster Setup
- **Package installation** via `cluster_setup.sh`
- **System configuration** and optimization
- **Dependency management** for Lustre components

### 2. Lustre Host Configuration
- **Host file management** via `fix_lustre_hosts_files.sh`
- **Network configuration** for Lustre communication
- **Service discovery** setup

### 3. Lustre Filesystem Creation
- **Component creation** via `setup_lustre.sh`
- **MGS/MDS/OSS deployment** across designated nodes
- **Filesystem mounting** and validation

### 4. Supporting Scripts
- **EBS volume management** for storage provisioning
- **Lustre component configuration** with proper settings
- **Performance tuning** and optimization
