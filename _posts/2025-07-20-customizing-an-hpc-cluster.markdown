---
layout: post
title:  "Customizing an HPC cluster" 
date:   2025-07-20 16:30:00 -0700
categories: hpc cluster-management
tags: [aws, hpc, parallelcluster, customization, automation, devops]
---

## Beyond Basic Cluster Deployment

AWS ParallelCluster provides an excellent foundation for deploying HPC infrastructure in the cloud, but out-of-the-box clusters often need additional customization to meet production requirements. While ParallelCluster handles the initial deployment of compute resources, network configuration, and scheduler setup, transforming these components into a production-ready environment requires additional automation.

In this post, I'll introduce my `cluster_setup.sh` script that automates post-deployment customization for AWS ParallelCluster environments. This script addresses common requirements for production HPC workloads that aren't covered by the default ParallelCluster deployment.

**GitHub Repository**: [https://github.com/veloduff/hpc/tree/main/Cluster_Setup/cluster_setup.sh](https://github.com/veloduff/hpc/blob/main/Cluster_Setup/cluster_setup.sh)


The complete cluster setup script and supporting tools are available in my HPC repository.

## Common Production Requirements

Production HPC environments typically need several customizations beyond the basic cluster deployment:

- Consistent software environments across all nodes
- Efficient cluster-wide administration tools
- Performance monitoring and diagnostics
- Optimized storage configuration
- MPI library setup and validation
- Scheduler customization and testing

My `cluster_setup.sh` script automates all these customizations in a single operation, ensuring consistency and reducing manual configuration errors.

## Prerequisites

- An existing AWS ParallelCluster deployment (see my [Creating an HPC Cluster with AWS ParallelCluster]({% post_url 2024-06-15-creating-hpc-cluster-aws-parallelcluster %}) guide)
- Basic understanding of Linux and SSH

## Automating Cluster Customization

To customize your ParallelCluster deployment:

1. **Prepare the required scripts**:
   - `cluster_setup.sh`: Main cluster customization script
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

## What the Customization Script Does

The `cluster_setup.sh` script performs nine key operations to transform a basic ParallelCluster deployment into a production-ready HPC environment:

1. **Package Management**: Installs essential HPC tools and utilities (pdsh, nvme-cli, monitoring tools)
2. **Cluster Communication**: Sets up pdsh (Parallel Distributed Shell) for parallel command execution
3. **Host Management**: Creates and distributes cluster host files for node-to-node communication
4. **SSH Configuration**: Enables passwordless root access across all cluster nodes
5. **Environment Setup**: Configures .bash_profile for optimal cluster operations
6. **Storage Management**: Cleans up Instance Store devices for filesystem use (Lustre/GPFS)
7. **Monitoring Tools**: Installs performance monitoring and debugging utilities (htop, nmon, iperf3)
8. **MPI Validation**: Creates, compiles, and tests MPI applications for cluster verification
9. **Slurm Testing**: Submits test jobs to validate scheduler functionality

## Benefits of Automated Customization

This automated approach to cluster customization provides several key benefits:

1. **Consistency**: Ensures all nodes have identical configurations
2. **Efficiency**: Reduces setup time from hours to minutes
3. **Reproducibility**: Creates predictable environments across multiple clusters
4. **Validation**: Automatically verifies that the cluster is functioning correctly

## Validating the Customized Cluster

After running the setup script, you can verify that your customizations were applied correctly:

1. **Verify parallel command execution** with pdsh:
   ```bash
   $ pdsh hostname
   batch01-st-batch-01: batch01-st-batch-01
   batch01-st-batch-02: batch01-st-batch-02
   batch01-st-batch-03: batch01-st-batch-03
   ...
   ```

2. **Check host file consistency** across the cluster:
   ```bash
   $ pdsh cat /etc/hosts | dshbak -c
   ```

3. **Verify MPI functionality** with the test results:
   ```bash
   $ cat mpi-test.out
   Currently Loaded Modulefiles:
    1) openmpi/4.1.7
   Hello world from processor batch01-st-batch-31, rank 30 out of 32 processors
   Hello world from processor batch01-st-batch-27, rank 26 out of 32 processors
   ...
   ```

## Leveraging the Customized Environment

With your customized cluster in place, you can now take advantage of several advanced capabilities:

### 1. Cluster-Wide Monitoring

Use pdsh to create comprehensive monitoring scripts:

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

### 2. Efficient File Distribution

Use pdcp for parallel file operations:

```bash
# Copy configuration files to all nodes
pdcp -w ^$HOME/cluster-ip-addr /path/to/config.file /destination/path/

# Distribute application binaries
pdcp -r -w ^$HOME/cluster-ip-addr /path/to/application/ /opt/apps/
```

### 3. Automated Health Checks

Implement periodic health checks across the cluster:

```bash
# Check for failed services
pdsh "systemctl list-units --state=failed" | dshbak -c

# Verify network connectivity
pdsh "ping -c 1 $(hostname)" | dshbak -c
```

## Conclusion: From Basic to Production-Ready

While AWS ParallelCluster provides an excellent starting point for HPC in the cloud, the automated customizations provided by the `cluster_setup.sh` script transform a basic deployment into a production-ready environment. This approach delivers several key benefits:

1. **Reduced Setup Time**: Automates hours of manual configuration
2. **Improved Reliability**: Ensures consistent configuration across all nodes
3. **Enhanced Functionality**: Adds critical tools for cluster management and monitoring
4. **Validated Environment**: Confirms that all components are working correctly

By automating these customizations, you can quickly deploy consistent, production-ready HPC environments that meet the demanding requirements of real-world scientific and engineering workloads.

---

*This post is part of a series on building production-ready HPC environments on AWS. In my next post, I'll explore how to leverage this customized environment for parallel filesystem deployment and optimization.*
