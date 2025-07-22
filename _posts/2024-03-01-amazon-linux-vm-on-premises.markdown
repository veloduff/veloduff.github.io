---
layout: post
title:  "Running Amazon Linux as a VM On-Premises with QEMU"
date:   2024-03-01 15:00:00 -0700
categories: aws virtualization qemu
---

*Updated on 2025-01-21*

Setting up Amazon Linux in a virtual machine on your local machine can be incredibly useful for development, testing, and learning AWS services without spinning up cloud instances. This guide walks through the process of running Amazon Linux 2023 as a VM using QEMU on macOS.

## Why Run Amazon Linux On-Premises?

Amazon Linux provides the same environment you'll find in EC2 instances, making it perfect for:
- **Local development** that mirrors your production environment
- **Testing scripts and configurations** before deploying to AWS
- **Learning AWS tools** without incurring cloud costs
- **Offline development** when internet connectivity is limited

## Prerequisites

This setup uses QEMU with KVM acceleration to function as a type-1 hypervisor, providing near-native performance for your Amazon Linux VM.

## Step-by-Step Setup

### 1. Create Cloud-Init Configuration Files

First, we need to create the cloud-init configuration files that will initialize our VM.

Create a `meta-data` file:
```yaml
local-hostname: al2023vm01
```

Create a `user-data` file (don't modify this until you verify it works):
```yaml
#cloud-config
#vim:syntax=yaml
users:
  - default
chpasswd:
  list: |
    ec2-user:amazon
```

This configuration sets the password for the `ec2-user` account to `amazon`.

### 2. Install Required Tools

Install `dvdrtools` which includes the `mkisofs` command:
```bash
brew install dvdrtools
```

Install QEMU:
```bash
brew install qemu
```

Install [RealVNC Viewer](https://www.realvnc.com/en/connect/download/viewer/macos/) for connecting to the VM's display.

### 3. Create the Seed ISO

Create the `seed.iso` file using the configuration files:
```bash
mkisofs -output seed.iso -volid cidata -joliet -rock user-data meta-data
```

This ISO contains the cloud-init data that will configure your VM on first boot.

### 4. Download the Amazon Linux Image

Download the VM image (.qcow2 file) from AWS, [current location](https://docs.aws.amazon.com/linux/al2023/ug/outside-ec2-download.html)

The download location changes frequently, so check the [official AWS documentation](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/amazon-linux-2-virtual-machine.html) if the above link does not work. 

### 5. Launch the Virtual Machine

Start the VM with the following command:
```bash
qemu-system-x86_64 -m 2G -smp 6 \
  -cdrom seed.iso \
  -drive file=amzn2-kvm-2.0.20240412.0-x86_64.xfs.gpt.qcow2,if=virtio \
  -vga virtio \
  -display vnc=localhost:0 \
  -usb -device usb-tablet \
  -cpu host \
  -machine type=q35,accel=hvf
```

**Command breakdown:**
- `-m 2G`: Allocates 2GB of RAM
- `-smp 6`: Uses 6 CPU cores
- `-cdrom seed.iso`: Mounts our cloud-init configuration
- `-drive file=...`: Specifies the Amazon Linux disk image
- `-display vnc=localhost:0`: Enables VNC access on localhost:5900
- `-cpu host -machine type=q35,accel=hvf`: Enables hardware acceleration on macOS

### 6. Connect to Your VM

Open RealVNC Viewer and connect to `localhost:0` (or `localhost:5900`).

You should see the Amazon Linux boot process, and once complete, you can log in with:
- **Username:** `ec2-user`
- **Password:** `amazon`

## What's Next?

Once your Amazon Linux VM is running, you can:

1. **Install AWS CLI** and configure it with your credentials
2. **Set up development tools** like Docker, Python, or Node.js
3. **Test AWS SDK applications** locally
4. **Test system configuration** in a safe environment
5. **Create snapshots** of your configured VM for quick restoration

## Troubleshooting Tips

- **Performance issues?** Ensure hardware acceleration is working with the `accel=hvf` option
- **Can't connect via VNC?** Check that port 5900 isn't blocked by your firewall
- **VM won't boot?** Verify the .qcow2 file downloaded completely and isn't corrupted
- **Cloud-init not working?** Double-check the YAML formatting in your user-data file

## Conclusion

Running Amazon Linux on-premises gives you a powerful development environment that closely mirrors AWS EC2 instances. This setup is particularly valuable for developers working with AWS services who want to test locally before deploying to the cloud.

The QEMU-based approach provides excellent performance while keeping your development environment completely under your control. Whether you're learning AWS, developing cloud applications, or just need a reliable Linux environment, this setup delivers the flexibility and familiarity of Amazon Linux right on your local machine.
