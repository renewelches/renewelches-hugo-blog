---
layout: post
title: "Creating a Debian Cloud Image Template for Proxmox"
subtitle: "Build reusable VM templates with cloud-init support"
date: 2026-01-23 11:00:00
author: "Rene Welches"
description: "Step-by-step guide to creating a Debian generic cloud image template in Proxmox. Learn how to download, configure, and convert cloud images into reusable VM templates with cloud-init support."
image: "/img/proxmox-homelab-banner.svg"
publishDate: 2026-01-23 18:00:00
tags:
  - Proxmox
  - Debian
  - Cloud-Init
  - Template
  - Virtualization
categories: [homelab, Proxmox]
URL: "/2026/01/23/proxmox-debian-cloud-image"
---

## Overview

This guide walks through creating a Debian cloud image template in Proxmox. Cloud images are pre-built, minimal OS images optimized for virtualized environments, featuring a slimmed-down kernel with minimal hardware drivers and pre-installed cloud-init for automated configuration.

They enable automated configuration of hostname, SSH keys, networking, and more at first boot. Sure, you could also create a template manually, set up SSH keys within the template and set up a default user, but that would be just half the fun and you would still use a bloated OS with a lot of stuff you don't need.

## What We're Doing

We are following for the most part the official document on [Proxmox Cloud Images](https://pve.proxmox.com/wiki/Cloud-Init_Support#:~:text=Proxmox%20VE%20generates%20an%20ISO,is%20a%20requirement%20for%20OpenStack). But instead of using Ubuntu we are going to use Debian 13.

## Step-by-Step Guide

### Step 1: Download the Debian Cloud Image

SSH into one of your Proxmox nodes and download the official Debian cloud image:

```bash
# Navigate to the directory where proxmox stores iso images
cd /var/lib/vz/template/iso

# Download the latest Debian 13 (trixie) generic cloud image
wget https://cloud.debian.org/images/cloud/trixie/latest/debian-13-generic-amd64.qcow2
```

and verify the checksums.

Note: Proxmox will not allow you to import an image with an ending `qcow2` via the UI. You will have to do this via command line interface.

### Step 2: Create a New VM

Create a VM that will become our template:

```bash
# Create a new VM with ID 1001 (choose an unused ID)
qm create 1001 --name debian-13-cloud-template --memory 2048 --net0 virtio,bridge=vmbr0 --scsihw virtio-scsi-pci
```

`1001`: Your VM ID, choose an unsed ID.
`--name debian-13-cloud-template`: The name of the VM aka template.
`--memory 2048`: The amount RAM for the VM
`--cores 2`: The number of CPU cores for the VM
`--net0 virtio,bridge=vmbr0`: Configures the first network interface. `virtio` is a paravirtualized network adapter that provides better performance than emulated hardware because the guest OS knows it's virtualized. `bridge=vmbr0` connects this interface to the Proxmox bridge `vmbr0`, which is typically the default bridge linking VMs to your physical network. Without this parameter, the VM would have no network connectivity.
`--scsihw virtio-scsi-pci`: Sets the SCSI controller type. `virtio-scsi-pci` is a paravirtualized SCSI controller that offers better performance and more features than emulated controllers (like LSI). It supports advanced features like TRIM/discard for SSDs, multiple queues for improved I/O parallelism, and hot-plugging of disks. This is the recommended controller type for modern Linux guests.

### Step 3: Import the Cloud Image as Disk

Import the downloaded qcow2 image as the VM's disk:

```bash
# Import the disk to your storage (adjust 'local-lvm' to your storage name)
qm importdisk 1001 debian-13-generic-amd64.qcow2 local-lvm
```

`1001`: The VM ID to import the disk into (must match the VM created in Step 2).
`debian-13-generic-amd64.qcow2`: The source disk image file you downloaded. It needs to be the full path if you are not running it the directory where you downloaded the image e.g. `/var/lib/vz/template/iso/debian-13-generic-amd64.qcow2`
`local-lvm`: The target storage where the disk will be stored. Adjust this to match your Proxmox storage name (e.g., `local-zfs`, `ceph-pool`, etc.).

### Step 4: Attach the Disk to the VM

Configure the VM to use the imported disk:

```bash
# Attach the disk as scsi0
qm set 1001 --scsihw virtio-scsi-pci --scsi0 local-lvm:vm-1001-disk-0

# Set the boot order
qm set 1001 --boot c --bootdisk scsi0
```

### Step 5: Add Cloud-Init Drive

Add a cloud-init drive for automated configuration:

```bash
# Add cloud-init CD-ROM drive
qm set 1001 --ide2 local-lvm:cloudinit

# Enable serial console for cloud-init output, Debian images require a serial console
qm set 1001 --serial0 socket --vga serial0
```

Some images rely on serial0 being present as a display option.

### Step 5: Convert the VM to a template

```bash
qm template 1001
```

### Step 6: Configure Cloud-Init Defaults

This step is optional, and the cloud-init values can be set at VM creation e.g. via Terraform.
Set default cloud-init parameters (change as needed by your use case):

```bash
# Set default user (optional)
qm set 1001 --ciuser admin

# Set SSH public key (recommended)
qm set 1001 --sshkey ~/.ssh/id_rsa.pub

# Configure IP via DHCP
qm set 1001 --ipconfig0 ip=dhcp

# When using DHCP a running qemu agent is required, otherewise any cloned VM will not get an IP address. Enable the qemu agent:
qm set 1001 --agent 1
```

<!-- TODO: Add section about static IP configuration -->

### Step 7: Resize the Disk (Optional)

The cloud image comes with a small disk. Resize if needed:

```bash
# Add 20GB to the disk
qm resize 1001 scsi0 +20G
```

## All the Steps in a Single Bash Script

```bash
#!/bin/bash
set -e

TEMPLATE_ID=$(pvesh get /cluster/nextid) || { echo "Failed to get ID"; exit 1; }
TEMPLATE_NAME="debian-13-cloudinit-tmplt"
TEMPLATE_IMG=/mnt/pve/minis-cluster-nfs/template/iso/debian-13-generic-amd64.qcow2
STORAGE="local-lvm"

if [[ ! -f "$TEMPLATE_IMG" ]];
    then echo "$TEMPLATE_IMG"
    exit 1
fi

echo "Creating VM $TEMPLATE_ID with name $TEMPLATE_NAME"
qm create $TEMPLATE_ID --name $TEMPLATE_NAME --memory 2048 --cores 2 --net0 virtio,bridge=vmbr0 --scsihw virtio-scsi-pci

echo "Importing disk..."
qm importdisk $TEMPLATE_ID $TEMPLATE_IMG $STORAGE

echo "Configuring VM..."
echo "Attaching Disk..."
qm set $TEMPLATE_ID --scsihw virtio-scsi-pci --scsi0 ${STORAGE}:vm-${TEMPLATE_ID}-disk-0
echo "Setting boot order..."
qm set $TEMPLATE_ID --boot order=scsi0 --serial0 socket --vga std
echo "Adding Cloud-Init Drive..."
qm set $TEMPLATE_ID --ide2 ${STORAGE}:cloudinit

echo "Converting to template..."
qm template $TEMPLATE_ID

echo "Configure Cloud-Init Defaults..."
echo "Setting ci user..."
qm set $TEMPLATE_ID --ciuser admin
echo "Setting up puplic ssh key..."
qm set $TEMPLATE_ID --sshkey ~/.ssh/id_rsa.pub
echo "Enabling DHCP..."
qm set $TEMPLATE_ID --ipconfig0 ip=dhcp
echo "Enabling qemu-guest-agent..."
qm set $TEMPLATE_ID --agent 1
echo "Set ci user password..."
qm set $TEMPLATE_ID --cipassword
echo "Resizing Disk"
qm resize 1001 scsi0 +17G

echo "Template $TEMPLATE_ID created successfully!"
```

## Deploying VMs from the Template

### Clone the Template

```bash
# Full clone (independent copy) - replace 101 with a availble VM ID.
qm clone 1001 101 --name my-debian-vm --full

# Or linked clone (shares base image, saves space)
qm clone 1001 101 --name my-debian-vm
```

### Configure the Clone

```bash
# Set hostname and other cloud-init options
qm set 101 --ciuser myuser --ipconfig0 ip=192.168.1.100/24,gw=192.168.1.1
```

### Start the VM

```bash
qm start 101
```

## Cloud-Init Configuration Options

| Parameter      | Description           | Example                                 |
| -------------- | --------------------- | --------------------------------------- |
| `--ciuser`     | Default user          | `admin`                                 |
| `--cipassword` | User password         | `secretpass`                            |
| `--sshkey`     | SSH public key file   | `~/.ssh/id_rsa.pub`                     |
| `--ipconfig0`  | Network configuration | `ip=dhcp` or `ip=x.x.x.x/24,gw=x.x.x.x` |
