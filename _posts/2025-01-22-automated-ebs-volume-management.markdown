---
layout: post
title:  "Automated EBS Volume Creation and Attachment" 
date:   2025-01-22 15:00:00 -0700
categories: aws ebs storage automation
---

The `ebs_create_attach.sh` script automates the entire process of creating, attaching, and configuring EBS volumes with intelligent device management and comprehensive error handling.

## What This Script Does

The script handles the complete EBS volume lifecycle:
- **Creates** EBS volumes with configurable specifications
- **Automatically detects** EC2 instance metadata (region, AZ, instance ID)
- **Finds available device names** to avoid conflicts
- **Attaches volumes** with proper configuration
- **Maps NVMe devices** on modern instance types
- **Applies tags** for organization and tracking
- **Configures DeleteOnTermination** to prevent orphaned volumes

## Key Features

- The script automatically finds the next available device name, starting from `/dev/sdf` and incrementing through `/dev/sdz`. It checks both the local system and AWS to ensure no conflicts exist.
- The script enables DeleteOnTermination by default to prevent orphaned volumes that continue to incur charges after instance termination.
-  Use tagging for cost allocation and resource management, for example:
   ```bash
   ./ebs_create_attach.sh --tag "Environment:Production" --tag "CostCenter:Engineering" --tag "Project:DataPipeline"
   ```

## Usage Examples

### Basic Usage
Create a standard 8GB gp3 volume with default settings:
```bash
./ebs_create_attach.sh
```

### High-Performance Volume
Create a 100GB io2 volume with 10,000 IOPS:
```bash
./ebs_create_attach.sh --volume-type io2 --size 100 --iops 10000
```

### Optimized gp3 Volume
Create a gp3 volume with custom IOPS and throughput:
```bash
./ebs_create_attach.sh --volume-type gp3 --size 50 --iops 4000 --throughput 500
```

### Tagged Volume
Create a volume with custom tags for organization:
```bash
./ebs_create_attach.sh --tag "Environment:Production" --tag "Project:DataAnalysis"
```

### Specific Device and Region
Override automatic detection for specific requirements:
```bash
./ebs_create_attach.sh --device /dev/sdh --region us-east-1
```

## Command Line Options

The script provides extensive configuration options:

| Option | Description | Default |
|--------|-------------|---------|
| `--volume-type` | EBS volume type (gp2\|gp3\|io1\|io2\|st1\|sc1) | gp3 |
| `--size` | Volume size in GB | 8 |
| `--iops` | IOPS for gp3/io1/io2 volumes | 3000 |
| `--throughput` | Throughput for gp3 volumes (MiB/s) | - |
| `--device` | Device name (auto-increments if unavailable) | /dev/sdf |
| `--tag` | Custom tag in "KEY:VALUE" format | - |
| `--delete-on-termination` | Delete volume when instance terminates | true |
| `--region` | AWS region (auto-detected on EC2) | - |
| `--az` | Availability zone (auto-detected on EC2) | - |
| `--instance-id` | EC2 instance ID (auto-detected on EC2) | - |

## Verify

```sh
$ sudo nvme list
Node                  SN                   Model                                    Namespace Usage                      Format           FW Rev
--------------------- -------------------- ---------------------------------------- --------- -------------------------- ---------------- --------
/dev/nvme0n1          vol00000000000000000 Amazon Elastic Block Store               1          48.32  GB /  48.32  GB    512   B +  0 B   2.0
/dev/nvme1n1          AWS11111111111111111 Amazon EC2 NVMe Instance Storage         1         237.00  GB / 237.00  GB    512   B +  0 B   0
```


