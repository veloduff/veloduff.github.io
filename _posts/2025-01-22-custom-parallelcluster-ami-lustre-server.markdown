---
layout: post
title:  "Building Custom ParallelCluster AMIs with Lustre Server Support"
date:   2025-01-22 14:00:00 -0700
categories: aws hpc lustre parallelcluster storage
---

Standard ParallelCluster AMIs come with Lustre client support, but building Lustre servers requires additional components:
- **Lustre server packages** for MGS, MDS, and OSS (and their targets) 
- **DKMS** for dynamic kernel modules
- **ZFS backend** used to build the lustre file system 
- **Proper kernel modules** compiled for the specific environment

Generic information is here: [Modify an AWS ParallelCluster AMI](https://docs.aws.amazon.com/parallelcluster/latest/ug/building-custom-ami-v3.html#modify-an-aws-parallelcluster-ami-v3)

## Package Conflicts with Client Packages

The default ParallelCluster AMI includes Lustre client packages that conflict with server installations. They should be removed for this process.

## Step-by-Step Customization Process

### Starting Point
Begin with a recent ParallelCluster AMI:
```
ami-068c41ec88596d8b4 
(aws-parallelcluster-3.12.0-rhel8-hvm-x86_64-202412170018)
```

Launch an EC2 instance with the ParallelCluster AMI.

### 1. Remove Conflicting Packages

Once booted, ssh to the instance and remove the lustre client packages.
```bash
# Remove existing Lustre client packages
dnf remove lustre-client kmod-lustre-client
```

This prevents version conflicts between client and server components that would block the server installation.

### 2. Verify Security Configuration
```bash
# Ensure SELinux is disabled (required for Lustre)
sestatus
# Should show: SELinux status: disabled
```

Lustre requires SELinux to be disabled for proper operation.

### 3. Reboot, update, and reboot again
The first reboot ensures all old Lustre kernel modules are completely unloaded before installing server components.

```bash
# Reboot to completely clear kernel modules
reboot

# Update system packages
dnf update
reboot
```

### 4. Install ZFS and DKMS Backend
ZFS provides the backend storage system that Lustre uses. This process automatically installs DKMS.

```bash
# Add required repositories
dnf install -y https://dl.fedoraproject.org/pub/epel/epel-release-latest-8.noarch.rpm
dnf install -y https://zfsonlinux.org/epel/zfs-release-2-3$(rpm --eval "%{dist}").noarch.rpm

# Install ZFS with kernel development headers
dnf install -y kernel-devel zfs
```

### 5. Verify DKMS and ZFS Installation
```bash
# Check DKMS module compilation
dkms status

# Load ZFS kernel module
modprobe -v zfs

# Verify ZFS functionality
zpool version
```

### 6. Configure Lustre Repository
Create the Lustre server repository configuration, this is for version `lustre-2.15.4/el8.9`, which does work with RHEL 8.10.
```bash
cat > /etc/yum.repos.d/lustre.repo << EOF
[lustre-server]
name=lustre-server
baseurl=https://downloads.whamcloud.com/public/lustre/lustre-2.15.4/el8.9/server/
exclude=*debuginfo*
enabled=0
gpgcheck=0
EOF
```

### 7. Enable Development Tools
```bash
# Enable CodeReady repository for build dependencies
dnf config-manager --set-enabled codeready-builder-for-rhel-8-rhui-rpms
```

### 8. Install Lustre Server Components
This installs:
- **lustre-dkms**: Dynamic kernel module support
- **lustre-osd-zfs-mount**: ZFS object storage device support
- **lustre**: Core Lustre server utilities

```bash
# Install Lustre server with ZFS backend support
dnf --enablerepo=lustre-server install lustre-dkms lustre-osd-zfs-mount lustre
```


### 9. Verify Installation
```bash
# Load Lustre kernel module
modprobe -v lustre

# Test Lustre control interface
lctl
```

Within the `lctl` interface, you can verify networking:
```bash
lctl > ping 172.31.26.176
12345-0@lo
12345-172.31.26.176@tcp

lctl > list_nids
172.31.26.176@tcp
```

### 10. Clean up and create a new image

Run the AMI clean up script:
```
sudo /usr/local/sbin/ami_cleanup.sh
```

Go to the console, select the instance, and choose "Create Image".
