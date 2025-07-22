---
layout: post
title:  "Automated Remote Workstation Deployment in the Cloud"
date:   2024-02-10 22:00:00 -0700
categories: aws hpc remote-desktop automation
---

Remote desktop access to powerful cloud instances is essential for HPC workloads, CAD applications, and GPU-intensive tasks. This automation script eliminates the manual setup complexity by deploying a complete NICE DCV remote desktop environment on AWS with a single command.

## What This Script Does

The `launch_dcv_instance.sh` script automates the entire process of creating a high-performance remote desktop workstation on AWS:

- **Launches** an `m5.8xlarge` instance (32 vCPUs, 256GB RAM)
- **Installs** Ubuntu Desktop with NICE DCV Server
- **Configures** security groups and networking automatically
- **Creates** a dedicated DCV user with secure password
- **Sets up** virtual desktop sessions
- **Provides** connection details for immediate access

## Key Features

The script automatically:
- **Automatically uses latest Ubuntu 22.04 AMI**
- **Generates unique key pairs** with timestamps
- **Creates security groups** with proper DCV and SSH access
- **Uses default VPC** for simplified networking
- **Configures 100GB GP3 storage** for adequate workspace

### Complete DCV Installation
The user data script handles the entire desktop environment setup:
- **Ubuntu Desktop** with GDM3 display manager
- **NICE DCV Server** with web viewer support
- **Virtual session creation** for headless operation
- **Automatic session startup** on boot
- **Secure password generation** for the DCV user

### Security Considerations
- **Unique key pairs** generated per deployment
- **Timestamped security groups** prevent conflicts
- **Random passwords** for DCV user accounts
- **Restricted access** to SSH (22) and DCV (8443) ports

## Usage

Simply run the script and wait for deployment:

```bash
./launch_dcv_instance.sh
```

The script provides complete connection information:
```
Instance i-1234567890abcdef0 is now running!
Public IP: 54.123.45.67
Connect to DCV: https://54.123.45.67:8443
SSH: ssh -i dcv-instance-key-20250121220000.pem ubuntu@54.123.45.67
DCV User: dcvuser
```

You will have to use the SSH connection string above, and cat the `dcvuser` password, for example:

```
ubuntu@ip-172-12-34-56:~$ sudo cat dcv_password.txt
dlj82/+mwsdfiw90
```

## Connecting

The [DCV Client](https://docs.aws.amazon.com/dcv/latest/userguide/client.html) or a web browser can be used connect. 

![DCV Desktop Interface](/assets/images/dcv_01.png)

Once connected, you'll have access to a full Ubuntu desktop environment, complete with all the power of your chosen EC2 instance.

The script uses `m5.8xlarge` instances for serious workloads, but you can easily modify the instance type:

```bash
INSTANCE_TYPE="m5.4xlarge"  # 16 vCPUs, 64GB RAM (lower cost)
INSTANCE_TYPE="m5.2xlarge"  # 8 vCPUs, 32GB RAM (budget option)
```

## Cleanup
Remember to terminate instances when not in use to avoid unnecessary charges.
